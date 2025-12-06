# flaws2.cloud CTF Challenge

> **Warning**
> This repository may contain spoilers for the flaws2.cloud CTF challenge. If you want the full learning experience, attempt the challenge yourself first at [flaws2.cloud](http://flaws2.cloud) before reviewing any notes or progress here. You have been warned. No refunds. Management is not responsible for ruined "aha!" moments.

A hands-on AWS security challenge demonstrating cloud penetration testing and incident response skills using AI-assisted tooling.

## What is This?

[flaws2.cloud](http://flaws2.cloud) is a capture-the-flag (CTF) challenge created by Scott Piper of Summit Route. It teaches real-world AWS security concepts through two complementary paths:

- **Attacker Path**: Exploit common AWS misconfigurations to escalate privileges and access protected resources
- **Defender Path**: Analyze CloudTrail logs to detect and investigate the attack chain

This repository provides a hardened environment for completing the challenge with Claude Code and AWS MCP (Model Context Protocol) servers as an AI-powered assistant.

## Skills Demonstrated

### Cloud Security
- AWS IAM privilege escalation techniques
- S3 bucket misconfiguration exploitation
- Lambda function security analysis
- ECR (Elastic Container Registry) enumeration
- Credential harvesting and lateral movement

### Incident Response
- CloudTrail log analysis
- Attack timeline reconstruction
- Indicator of compromise (IOC) identification
- Security event correlation

### Modern Tooling
- AI-assisted security research with Claude Code
- Infrastructure-as-code analysis
- AWS CLI and API proficiency

## Environment Setup

This project uses a VS Code devcontainer with:

- **Kali Linux** base image with essential CTF tools
- **AWS CLI** for direct AWS interaction
- **Claude Code** with three specialized MCP servers:
  - `aws-api-mcp-server` - Execute AWS CLI commands
  - `iam-mcp-server` - Analyze IAM policies and permissions
  - `ccapi-mcp-server` - Cloud Control API for resource management

### Prerequisites

- Docker Desktop
- VS Code with Dev Containers extension
- Anthropic API key (for Claude Code)
- AWS credentials for the challenge

### Quick Start

1. Clone this repository
2. Open in VS Code
3. When prompted, "Reopen in Container"
4. Configure AWS credentials in `~/.aws/credentials`
5. Start Claude Code: `claude`

## Challenge Progress

### Attacker Path
- [x] Level 1
- [x] Level 2
- [x] Level 3 ✅ **COMPLETE!**

### Defender Path
- [ ] Level 1
- [ ] Level 2
- [ ] Level 3

## Resources

- [flaws2.cloud Official Site](http://flaws2.cloud)
- [AWS CLI Documentation](https://docs.aws.amazon.com/cli/)
- [AWS IAM Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/)

## Spoilers

<details>
<summary>Click here for the solution to Level 1</summary>
<br>
<details>
<summary>Are you sure?</summary>
<br>
<details>
<summary>Really sure?</summary>
<br>
<details>
<summary>You've only been at this for like 5 minutes...</summary>
<br>
<details>
<summary>Fine. But I'm disappointed in you.</summary>
<br>
<pre>
No.

Try harder.

         ___
        /   \
       |  |  |
       |  |  |
       |_____|
       | AWS |
       |_____|
          |
         /|\
        / | \
       /  |  \
      /   |   \
     /    |    \
    /     |     \
   /      |      \
  /       |       \
 /_______|_______\

The real flag was the
friends we made along the way.

(And the crippling fear that
 your S3 buckets are public)
</pre>
</details>
</details>
</details>
</details>
</details>

## Disclaimer

This environment is intended for authorized security education only. The flaws2.cloud challenge is a legitimate learning platform. Never apply these techniques against systems without explicit authorization.

---

*Completed with Claude Code - AI-assisted cloud security research*
