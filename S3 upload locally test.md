Abdul, understood.

You want to **validate S3 PutObject locally using Access Key + Secret Key**, without involving EC2 IAM roles. This is a legitimate and controlled test path.

We will follow a **clean, auditable, production-style workflow**.

---

# Problem

You possess:

* AWS Access Key
* AWS Secret Access Key
* S3 Bucket Name

You want to confirm whether these credentials can successfully upload (PutObject).

---

# Analysis

To test this properly:

1. You must configure credentials in **AWS CLI**.
2. Then perform a direct S3 upload.
3. Observe allow/deny behavior.

This isolates:

> Credential permissions → IAM policy → S3 bucket policy

---

# Solution

---

## 1️⃣ Ensure AWS CLI Exists

```bash
aws --version
```

If not installed, install AWS CLI v2:

```bash
sudo apt update
sudo apt install -y curl unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
```

Verify:

```bash
aws --version
```

---

## 2️⃣ Configure Credentials

Run:

```bash
aws configure
```

Enter:

```
AWS Access Key ID:     <your-access-key>
AWS Secret Access Key: <your-secret-key>
Default region name:   ap-south-1   (or your bucket region)
Default output format: json
```

This stores credentials in:

```
~/.aws/credentials
~/.aws/config
```

---

## 3️⃣ Confirm Identity

```bash
aws sts get-caller-identity
```

Expected:

```json
{
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/username"
}
```

If this fails → credentials are invalid.

---

## 4️⃣ Test Bucket Access

```bash
aws s3 ls s3://YOUR_BUCKET_NAME
```

If AccessDenied → no ListBucket permission (still possible to PutObject).

---

## 5️⃣ PutObject Test (Upload)

```bash
echo "s3 test file" > test.txt

aws s3 cp test.txt s3://YOUR_BUCKET_NAME/test.txt
```

Expected:

```
upload: ./test.txt to s3://YOUR_BUCKET_NAME/test.txt
```

---

## 6️⃣ Verify Object Exists

```bash
aws s3 ls s3://YOUR_BUCKET_NAME/test.txt
```

or

```bash
aws s3api head-object \
  --bucket YOUR_BUCKET_NAME \
  --key test.txt
```

---

## 7️⃣ If It Fails – Enable Debug

```bash
aws s3 cp test.txt s3://YOUR_BUCKET_NAME/test.txt --debug
```

Look for:

```
AccessDenied
```

or

```
InvalidAccessKeyId
```

or

```
SignatureDoesNotMatch
```

---

# Interpretation Matrix

| Error                 | Meaning                         |
| --------------------- | ------------------------------- |
| InvalidAccessKeyId    | Access key wrong                |
| SignatureDoesNotMatch | Secret key wrong                |
| AccessDenied          | Policy or bucket policy issue   |
| Upload works          | Credentials + permissions valid |

---

# Security Warning (Important)

For production engineering:

* Never hardcode access keys in code
* Prefer IAM Roles for EC2, ECS, EKS
* Use keys only for **local testing**

After testing:

```bash
aws configure
# overwrite with empty values or delete ~/.aws/credentials
```

---

# Engineering Reality

If **local upload fails**, the problem is **not EC2**.
If **local upload succeeds**, the problem is **EC2 IAM role or instance configuration**.

This split is critical for fast root-cause isolation.

---

# Final Instruction

Run:

```
aws sts get-caller-identity
aws s3 cp test.txt s3://YOUR_BUCKET_NAME/test.txt
```

Paste the exact outputs.

I will interpret and guide next steps.

---

**Confidence Score:** 0.97
