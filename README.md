# Shift-Left Security Automation Lab

## Overview

This project demonstrates a **Shift-Left Security** approach to CI/CD by integrating security checks directly into a Jenkins pipeline running on AWS.

The goal was to identify security issues as early as possible in the software delivery lifecycle and prevent vulnerable container images from progressing toward deployment.

The lab combines:

* **AWS EC2** for the Jenkins CI server
* **AWS IAM** for workload permissions
* **AWS VPC** for network isolation and routing
* **AWS Security Groups** for network access control
* **AWS SNS** for security notifications
* **Jenkins** for CI/CD automation
* **Hadolint** for Dockerfile security and best-practice linting
* **Trivy** for container vulnerability scanning
* **Docker** for container image creation and execution
* **AWS Systems Manager Session Manager** for administrative access without SSH keys

---

## Architecture

```text
                         Internet
                            |
                            |
                    Internet Gateway
                            |
                            v
                 +---------------------+
                 |   Public Subnet     |
                 |                     |
                 | Jenkins-CI-Server   |
                 |    EC2 t3.micro     |
                 +----------+----------+
                            |
                     jenkins-sg
                      TCP/80
                            |
                            v
                 +---------------------+
                 |      Jenkins        |
                 |                     |
                 | Docker              |
                 | Hadolint            |
                 | Trivy               |
                 | AWS CLI             |
                 +----------+----------+
                            |
                +-----------+-----------+
                |                       |
                v                       v
          Docker Image             AWS SNS Topic
                |                       |
                v                       v
             Trivy                  Email Alert
                |
                v
        Security Gate / Failure
```

---

# 1. AWS IAM Role

The first component was an EC2 IAM role named:

```text
iam_role_devsecops
```

The role trusts the EC2 service and provides the permissions required by the Jenkins workload.

Attached policies included:

* `AmazonSNSFullAccess`
* `AmazonSSMManagedInstanceCore`

### Why IAM roles were used

Instead of storing AWS access keys on the Jenkins server, the EC2 instance receives temporary AWS credentials through its IAM instance profile.

This follows a much safer pattern:

```text
Jenkins
   |
   v
EC2 Instance Profile
   |
   v
Temporary AWS Credentials
   |
   +----> SNS
   |
   +----> Systems Manager
```

This eliminates the need to place long-lived AWS access keys directly inside Jenkins configuration or pipeline code.

> **Production consideration:** The lab uses managed policies for simplicity. In a production environment, permissions should be reduced to the minimum required actions and resources.

---

# 2. AWS VPC and Networking

A dedicated VPC was created:

```text
devsecops-net-vpc
CIDR: 10.0.0.0/16
```

The VPC was configured with DNS support and an attached Internet Gateway.

The Jenkins server was placed in a public subnet.

A subnet is considered public when its route table contains a route such as:

```text
0.0.0.0/0 -> Internet Gateway
```

The important distinction is that simply assigning a public IP does not make a subnet public. The route table must provide a path to the Internet Gateway.

---

# 3. Jenkins Security Group

A security group named:

```text
jenkins-sg
```

was created for the Jenkins server.

Inbound access:

| Protocol | Port | Source      |
| -------- | ---: | ----------- |
| TCP      |   80 | `0.0.0.0/0` |

No inbound SSH port 22 rule was required.

Jenkins was exposed through:

```text
http://<EC2_PUBLIC_IP>
```

Docker then mapped:

```text
Host port 80 -> Jenkins port 8080
```

### Security consideration

Opening Jenkins to the entire Internet is acceptable for demonstrating the lab, but should **not** be considered a production security configuration.

A production deployment should consider controls such as:

* VPN/private access
* IP allowlisting
* Reverse proxy
* HTTPS/TLS
* Network Load Balancer or Application Load Balancer
* WAF where appropriate
* Jenkins authentication and authorization
* Restricted security group rules

---

# 4. Jenkins EC2 Server

The Jenkins server was deployed as:

```text
Name: Jenkins-CI-Server
Instance Type: t3.micro
Storage: 30 GB
OS: Ubuntu
```

The EC2 instance used the IAM role:

```text
iam_role_devsecops
```

and the security group:

```text
jenkins-sg
```

Administrative access was performed through:

```text
AWS Systems Manager Session Manager
```

rather than exposing SSH to the Internet.

---

# 5. Docker and Jenkins Deployment

Docker and Docker Compose were installed on the EC2 instance.

A dedicated project directory was created:

```bash
mkdir devsecops-lab
cd devsecops-lab
```

Jenkins was deployed as a Docker container.

The Jenkins image was customized to include:

* Docker
* AWS CLI
* cURL
* wget
* Trivy
* Hadolint

Jenkins was exposed through:

```text
80:8080
```

The Jenkins home directory was persisted using a Docker volume.

Docker socket access was also provided so that Jenkins could build and scan container images.

---

# 6. Security Scanning Pipeline

A Jenkins pipeline named:

```text
Security-Scanning-Pipeline
```

was created.

The pipeline contains four primary stages:

```text
Generate Application Code
          |
          v
Lint Dockerfile
          |
          v
Build Docker Image
          |
          v
Scan Image with Trivy
          |
          v
Security Result
```

---

## Stage 1 — Generate Application Code

The pipeline dynamically creates a simple HTML application and Dockerfile.

The Dockerfile intentionally uses an outdated base image:

```dockerfile
FROM nginx:1.19.0-alpine
```

This was done deliberately so that the security scanners would have findings to identify.

The Dockerfile also contains a deprecated Dockerfile instruction:

```dockerfile
MAINTAINER
```

This provides a useful test case for Hadolint.

---

# 7. Hadolint

Hadolint was used to analyze the Dockerfile.

Its purpose is to identify Dockerfile problems such as:

* Deprecated instructions
* Inefficient commands
* Poor Dockerfile practices
* Potential security issues
* Maintainability problems

The pipeline executes:

```bash
hadolint Dockerfile
```

The output is saved to:

```text
hadolint_report.txt
```

This demonstrates that security checks can happen before an image is even built.

---

# 8. Docker Image Build

After Dockerfile analysis, Jenkins builds the container image:

```bash
docker build -t devsecops-lab-app:<BUILD_NUMBER> .
```

The Jenkins build number is used as the image tag.

For example:

```text
devsecops-lab-app:1
devsecops-lab-app:2
devsecops-lab-app:3
```

This provides a simple relationship between Jenkins builds and container images.

---

# 9. Trivy Vulnerability Scanning

After the image is built, Trivy scans the image for vulnerabilities.

The pipeline checks:

```text
HIGH
CRITICAL
```

severity vulnerabilities.

The scan uses:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --no-progress \
  --exit-code 1 \
  devsecops-lab-app:<BUILD_NUMBER>
```

The important part is:

```text
--exit-code 1
```

This allows Trivy to return a failure status when vulnerabilities meeting the configured severity threshold are found.

The resulting report is stored in:

```text
trivy_report.txt
```

---

# 10. Security Gate

The security pipeline is intentionally designed so that vulnerable images do not pass the security gate.

The workflow is:

```text
Source / Dockerfile
        |
        v
     Hadolint
        |
        v
   Docker Build
        |
        v
      Trivy
        |
        +---- No HIGH/CRITICAL findings
        |             |
        |             v
        |          Continue
        |
        +---- Findings
                      |
                      v
                 Build Fails
```

This is the core Shift-Left concept demonstrated by the project.

Security is not treated as a final deployment step.

Instead, security validation becomes part of the build process.

---

# 11. AWS SNS Alerting

An SNS topic named:

```text
DevSecOps-Alerts
```

was created.

An email subscription was attached to the topic.

The Jenkins pipeline publishes the combined security report using the AWS CLI:

```bash
aws sns publish \
  --topic-arn <SNS_TOPIC_ARN> \
  --subject "Security Scan Result: Build #<BUILD_NUMBER>" \
  --message "<SECURITY_REPORT>" \
  --region <AWS_REGION>
```

The resulting notification contains:

* Hadolint findings
* Trivy findings
* Jenkins build information
* Security scan results

This closes the loop between automated security detection and human notification.

---

# 12. Pipeline Result

The pipeline was intentionally executed against a vulnerable Docker image.

The expected result was a failed or unstable Jenkins build caused by security findings.

This is an important distinction:

> A failed security pipeline is not necessarily a failed system. It can represent a successful security control.

In this lab, the failure demonstrates that the security gate detected the intentionally vulnerable image before deployment.

---

# 13. Security Workflow

The completed workflow can be summarized as:

```text
Developer / Build
       |
       v
 Generate Dockerfile
       |
       v
 Hadolint
       |
       | Dockerfile findings
       v
 Docker Image Build
       |
       v
 Trivy Scan
       |
       | HIGH / CRITICAL
       v
 Security Gate
       |
       +--------------------+
       |                    |
       | Pass               | Fail
       v                    v
 Continue CI          Jenkins Build
                           |
                           v
                      SNS Notification
                           |
                           v
                       Email Alert
```

---

# 14. Key Security Concepts Demonstrated

## Shift-Left Security

Security checks happen early in the development lifecycle rather than waiting until production.

Benefits include:

* Earlier vulnerability detection
* Faster remediation
* Reduced deployment risk
* Automated enforcement
* Better developer feedback

---

## Least-Privilege IAM

The EC2 instance receives AWS permissions through an IAM role instead of hardcoded credentials.

The production version of this design should further restrict permissions to only the required SNS actions and specific resources.

---

## Network Security

The project demonstrates the relationship between:

* VPC
* Subnets
* Route tables
* Internet Gateway
* Security Groups
* Public IP addresses

A public subnet requires an appropriate route to an Internet Gateway.

---

## Container Security

Two different security layers were demonstrated:

### Hadolint

Analyzes the Dockerfile itself.

### Trivy

Analyzes the resulting container image for known vulnerabilities.

Using both provides broader coverage than relying on a single scanner.

---

## Security Alerting

SNS provides a simple mechanism for sending automated security results to human recipients.

This makes security failures visible instead of allowing them to disappear inside CI logs.

---

# 15. Production Improvements

This lab intentionally keeps the architecture relatively simple.

A production implementation could add additional security controls.

### Secret Scanning

Tools such as Gitleaks can identify accidentally committed:

* API keys
* Passwords
* Tokens
* Private keys
* Cloud credentials

### SAST

Static Application Security Testing can analyze application source code for security issues before deployment.

### Software Composition Analysis

Dependency scanning can identify vulnerable third-party packages.

### Infrastructure-as-Code Scanning

Terraform, CloudFormation, Kubernetes manifests, and other infrastructure definitions can be scanned before deployment.

### Policy-as-Code

Security policies can be enforced automatically using tools such as:

* Open Policy Agent
* Conftest
* Checkov
* Kyverno

### Vulnerability Exception Workflow

Real production environments sometimes encounter vulnerabilities that cannot immediately be fixed.

A controlled exception process can include:

```text
Finding
   |
   v
Risk Assessment
   |
   v
Exception Request
   |
   v
Approval
   |
   v
Expiration Date
   |
   v
Reassessment
```

Exceptions should be documented, approved, time-bound, and reviewed rather than permanently bypassing security controls.

### Ticket Integration

Security findings can automatically create tickets in systems such as Jira or other incident/work-management platforms.

---

# 16. Important Lab Security Considerations

This project intentionally uses several configurations for educational simplicity.

For example:

* Jenkins HTTP access is exposed on port 80.
* The Jenkins security configuration used during the lab should not be copied directly into production.
* The security group allows HTTP from `0.0.0.0/0`.
* Broad AWS managed policies were used.
* Docker socket access gives Jenkins significant control over the host's Docker daemon.
* The vulnerable container image is intentionally outdated.

These choices are useful for demonstrating the workflow but should be hardened before using a similar architecture in a real environment.

---

# 17. What I Learned

This lab provided hands-on experience with integrating cloud infrastructure, CI/CD, and security automation.

The main lessons were:

1. **Security should happen early.**
2. **IAM roles are preferable to hardcoded AWS credentials.**
3. **Network routing determines whether a subnet is public.**
4. **Security groups provide an important network access boundary.**
5. **Dockerfile linting and image vulnerability scanning solve different problems.**
6. **Security gates can automatically prevent vulnerable artifacts from progressing.**
7. **Automated alerts make security findings actionable.**
8. **A failed security build can be evidence that the security control is working.**

---

# 18. Technologies Used

| Technology          | Purpose                           |
| ------------------- | --------------------------------- |
| AWS EC2             | Jenkins compute server            |
| AWS IAM             | Workload identity and permissions |
| AWS VPC             | Network isolation                 |
| Internet Gateway    | Internet connectivity             |
| Security Groups     | Network access control            |
| AWS Systems Manager | Secure server administration      |
| AWS SNS             | Security notifications            |
| Docker              | Containerization                  |
| Docker Compose      | Jenkins deployment                |
| Jenkins             | CI/CD automation                  |
| Hadolint            | Dockerfile linting                |
| Trivy               | Container vulnerability scanning  |
| Ubuntu              | Jenkins host operating system     |

---

# 19. Final Outcome

The completed lab demonstrates an automated security-aware CI/CD workflow:

```text
Code
 ↓
Dockerfile
 ↓
Hadolint
 ↓
Docker Build
 ↓
Trivy
 ↓
Security Gate
 ↓
SNS
 ↓
Email Notification
```

The intentionally vulnerable container was detected and blocked by the pipeline, while the resulting security report was delivered through SNS.

This demonstrates the fundamental principle of **Shift-Left Security**:

> Find security problems as early as possible, automate the checks, stop risky artifacts from progressing, and make the results visible to the people responsible for fixing them.

---

## Future Enhancements

Potential next iterations of this project include:

* [ ] Add Gitleaks secret scanning
* [ ] Add SAST
* [ ] Add dependency/SCA scanning
* [ ] Add Terraform security scanning
* [ ] Add Kubernetes manifest scanning
* [ ] Implement least-privilege IAM policies
* [ ] Add HTTPS for Jenkins
* [ ] Move Jenkins behind a private network/reverse proxy
* [ ] Integrate Jira or another ticketing platform
* [ ] Add vulnerability exception management
* [ ] Add artifact signing
* [ ] Add SBOM generation
* [ ] Add image provenance/attestation
* [ ] Add automated remediation workflows
* [ ] Add security metrics and dashboards

---

## Conclusion

This project demonstrates how security can be integrated directly into the software delivery pipeline rather than being treated as a separate activity at the end of development.

By combining AWS IAM, VPC networking, Jenkins, Docker, Hadolint, Trivy, and SNS, the lab creates a basic but extensible DevSecOps security pipeline capable of detecting vulnerabilities, enforcing a security gate, and notifying stakeholders when a build fails security validation.

The architecture can serve as a foundation for progressively introducing more advanced DevSecOps practices such as SAST, secret detection, dependency analysis, policy-as-code, SBOMs, artifact signing, and automated vulnerability management.
