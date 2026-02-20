## Architecture Goal

* Domain: `vendors.achieve-erp.com`
* DNS hosted in: Cloudflare
* CDN: Amazon CloudFront
* Origin: Private S3 bucket (`vendors.achieve-erp.com`)
* TLS Certificate: AWS Certificate Manager (Public Certificate)

You are combining:

Cloudflare (DNS) → CloudFront (CDN) → S3 (Private Origin)

This is valid, but you must configure it correctly to avoid:

* TLS validation failures
* 403 AccessDenied from S3
* Double CDN caching issues
* SSL handshake mismatch

---

# 1️⃣ Important Architecture Clarification

Since your domain is in **Cloudflare**, you will:

* Use Cloudflare only for DNS (recommended)
* Disable Cloudflare proxy (set DNS record to “DNS only”)

Why?

If you keep Cloudflare proxy ON:

* You introduce CDN → CDN chaining
* Debugging becomes harder
* SSL modes can conflict
* You may pay twice for bandwidth

Production recommendation:

> Use Cloudflare only for DNS unless you intentionally want double CDN.

---

# 2️⃣ Step-by-Step Implementation

---

## Step 1 — Create S3 Bucket

Bucket name: `vendors.achieve-erp.com`

Configuration:

* Block ALL public access → Enabled
* Do NOT enable static website hosting
* Upload content (index.html, assets)

We will use:
S3 REST endpoint
NOT S3 website endpoint

---

## Step 2 — Request Public Certificate (ACM)

Service: AWS Certificate Manager

⚠ Critical Requirement:

CloudFront only supports ACM certificates in:

```
us-east-1 (N. Virginia)
```

If you create certificate in another region → it will NOT appear in CloudFront.

### Certificate Configuration

* Type: Public certificate
* Domain name:

  ```
  vendors.achieve-erp.com
  ```
* Validation method: DNS validation

---

## Step 3 — Validate ACM Using Cloudflare

ACM will give you a CNAME record like:

```
Name: _abc123.vendors.achieve-erp.com
Value: _xyz456.acm-validations.aws
```

In Cloudflare:

1. Add CNAME record exactly as ACM shows.
2. Important: Set to **DNS only** (gray cloud).
3. Wait for validation (usually 5–15 minutes).

Once validated → status becomes “Issued”.

---

# 3️⃣ Step 4 — Create CloudFront Distribution

Service: Amazon CloudFront

### Origin Configuration

Origin domain:

```
vendors.achieve-erp.com.s3.<region>.amazonaws.com
```

Origin access:

* Create Origin Access Control (OAC)

After creating distribution:

* Attach ACM certificate
* Add Alternate Domain Name (CNAME):

  ```
  vendors.achieve-erp.com
  ```

Viewer Protocol Policy:

* Redirect HTTP → HTTPS

---

# 4️⃣ Step 5 — Update S3 Bucket Policy

After OAC is created, attach the generated policy.

Example structure:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudfront.amazonaws.com"
  },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::vendors.achieve-erp.com/*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::<account-id>:distribution/<distribution-id>"
    }
  }
}
```

If you skip this → 403 error.

---

# 5️⃣ Step 6 — Configure Cloudflare DNS

In Cloudflare:

Create record:

* Type: CNAME
* Name: vendors
* Target: dxxxxxxxxxxxx.cloudfront.net
* Proxy status: **DNS only**

Do NOT enable proxy unless you understand:

* SSL modes
* Origin certificates
* Cache-control interactions
* Double compression issues

---

# 6️⃣ SSL Flow After Setup

Final flow:

User
→ Cloudflare DNS resolves
→ CloudFront (TLS terminated using ACM certificate)
→ Private S3 (via OAC)

Cloudflare is not terminating TLS (if DNS only).

---

# 7️⃣ Common Mistakes (High Probability)

❌ Creating ACM certificate in wrong region
❌ Forgetting to add alternate domain name in CloudFront
❌ Cloudflare proxy enabled (orange cloud)
❌ Using S3 website endpoint
❌ Public S3 bucket
❌ Forgetting OAC bucket policy

---

# 8️⃣ If You WANT to Use Cloudflare Proxy (Advanced)

Then architecture becomes:

User
→ Cloudflare CDN
→ CloudFront CDN
→ S3

Now you must:

* Set Cloudflare SSL mode to Full (Strict)
* Ensure ACM certificate valid
* Disable unnecessary caching at one layer
* Accept increased complexity

For most production systems:

> Choose ONE CDN layer unless you have a strong reason.

---

# 9️⃣ Deployment Pipeline (DevOps View)

Ideal CI/CD:

1. Build frontend
2. Upload to S3
3. Invalidate CloudFront only for:

   ```
   /index.html
   ```
4. Done

Avoid invalidating `/*` unless required.

---

# 🔎 Engineering Gap Check

You should be able to explain:

1. Why ACM must be in us-east-1 for CloudFront?
2. Why OAC is more secure than public bucket?
3. What happens if Cloudflare proxy is ON?
4. How TLS negotiation works in this setup?
5. How to debug 403 vs 502 errors?

If not, your CDN knowledge is partial.

---

If you want next:

* Terraform configuration example
* Full CI/CD YAML for deployment
* Cost comparison: CloudFront only vs Cloudflare only
* Multi-environment (dev/stage/prod) DNS strategy

---

**Confidence Score: 9/10**
(Architecture guidance is accurate; minor console UI wording may vary.)
