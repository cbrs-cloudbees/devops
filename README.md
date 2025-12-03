# Trivy End-to-End Practical Guide

## Overview

Trivy (by Aqua Security) is an open-source vulnerability scanner used widely in DevSecOps pipelines. It scans container images, filesystems, Git repositories, Kubernetes clusters, and IaC templates for vulnerabilities (CVEs), misconfigurations, secrets, and SBOMs.

This document provides a complete, practical, real-world reference including:

* What is Trivy
* Key features & advantages
* Supported scan types
* Installing and updating vulnerability DB
* Using Trivy locally and in CI/CD
* Jenkins pipeline end-to-end example
* Uploading scan results to private S3
* Threshold enforcement to block deployment
---
## What is Trivy?

Trivy is a comprehensive security scanner created to identify vulnerabilities and security issues in:

* Container images
* Filesystems & packages (OS + application dependencies)
* Kubernetes clusters
* Infrastructure-as-Code (IaC) templates
* Source code repositories
* Dockerfiles
* SBOM (CycloneDX / SPDX)

Trivy is simple to run, fast, and suitable for CI/CD.

---

## Key Advantages of Trivy

| Benefit                             | Description                             |
| ----------------------------------- | --------------------------------------- |
| Zero‑config quick start             | Minimal setup required                  |
| Continuous vulnerability DB updates | Always current without upgrading binary |
| Multi‑resource scanning             | Images, files, Git repos, K8s, IaC      |
| Works offline / internal networks   | Enterprise support                      |
| CI/CD friendly                      | Exit codes and thresholds               |
| Multiple output formats             | table, json, sarif, template, cyclonedx |
| Built-in severity filtering         | HIGH/CRITICAL only                      |

---

## Supported Scan Types

| Scan Type                                        | Command Example                     |
| ------------------------------------------------ | ----------------------------------- |
| Container Image scan                             | `trivy image nginx:latest`          |
| Filesystem scan                                  | `trivy fs .`                        |
| Directory scan (dependency)                      | `trivy fs /path/app`                |
| Git Repository                                   | `trivy repo https://github.com/...` |
| Kubernetes cluster                               | `trivy k8s cluster`                 |
| Pod / Namespace / Node scans                     | `trivy k8s --namespace dev`         |
| IaC (Terraform, CloudFormation, Kubernetes YAML) | `trivy config .`                    |
| SBOM input                                       | `trivy sbom --input sbom.json`      |

---

## Installing Trivy

### Linux installation

```bash
sudo apt update
sudo apt install -y wget
VERSION=$(curl -s https://api.github.com/repos/aquasecurity/trivy/releases/latest | grep tag_name | cut -d '"' -f4)
wget https://github.com/aquasecurity/trivy/releases/download/${VERSION}/trivy_${VERSION}_Linux-64bit.tar.gz
tar zxvf trivy_${VERSION}_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/
trivy --version
```

### Docker usage

```bash
docker run --rm aquasec/trivy:latest image nginx:latest
```

---

## Vulnerability DB Updates

Trivy updates its vulnerability DB automatically every **12 hours**.
No Trivy version upgrade is required.

### Manual DB refresh

```bash
trivy --download-db-only
```

### Force refresh (delete cache)

```bash
rm -rf ~/.cache/trivy
trivy --download-db-only
```

### DB location

`~/.cache/trivy/db`

---

## Scanning a Docker Image

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --format json --output trivy-report.json myorg/myapp:1.0.0
```

| Option                     | Meaning                                |
| -------------------------- | -------------------------------------- |
| `--severity HIGH,CRITICAL` | Only high & critical vulnerabilities   |
| `--ignore-unfixed`         | Skip vulnerabilities without fixes     |
| `--format json`            | Useful for automation                  |
| `--exit-code 1`            | Fail pipeline if vulnerabilities found |

---

## Uploading Report to S3 (Private Bucket)

### S3 configuration

* Block public access = **ON**
* No public ACLs or policies
* Use IAM roles or keys for Jenkins access

### IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::myorg-trivy-reports/*"
    }
  ]
}
```

---

## Jenkins Pipeline Integration (End-to-End)

### Features included

✔ Trivy scan
✔ Upload JSON report to S3
✔ Print S3 URL in pipeline
✔ Threshold enforcement (fail build if >5 High+Critical)
✔ Still uploads report even if failure

```groovy
pipeline {
  agent any

  environment {
    AWS_REGION  = 'ap-south-1'
    BUCKET_NAME = 'myorg-trivy-reports'
    IMAGE_NAME  = "myorg/myapp:${env.BUILD_NUMBER}"
    TRIVY_THRESHOLD = '5'
  }

  stages {

    stage('Build Docker Image') {
      steps {
        sh """
          docker build -t ${IMAGE_NAME} .
        """
      }
    }

    stage('Trivy Scan & Upload Report') {
      steps {
        script {
          sh """
            trivy image --severity HIGH,CRITICAL --ignore-unfixed --format json --output trivy-report.json ${IMAGE_NAME}
            trivy image --severity HIGH,CRITICAL --ignore-unfixed ${IMAGE_NAME} || true

            apt-get update && apt-get install -y jq

            HIGH_COUNT=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="HIGH")] | length' trivy-report.json)
            CRIT_COUNT=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' trivy-report.json)
            TOTAL=$((HIGH_COUNT + CRIT_COUNT))
            echo "TOTAL=$TOTAL" > vuln.txt
          """

          def total = sh(script: "grep TOTAL vuln.txt | cut -d '=' -f2", returnStdout: true).trim()
          def s3Key = "${env.JOB_NAME}/${env.BUILD_NUMBER}/trivy-report.json"

          sh "aws s3 cp trivy-report.json s3://${BUCKET_NAME}/${s3Key}"

          echo "S3 URL: https://${BUCKET_NAME}.s3.${AWS_REGION}.amazonaws.com/${s3Key}"

          if (total.toInteger() > TRIVY_THRESHOLD.toInteger()) {
            error("❌ Vulnerabilities exceed threshold (${total} > ${TRIVY_THRESHOLD})")
          }
        }
      }
    }

    stage('Deploy') {
      when {
        expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
      }
      steps {
        echo "Deploying application..."
      }
    }
  }
}
```

---

## Best Practices

| Recommended                          | Reason                                      |
| ------------------------------------ | ------------------------------------------- |
| Upload JSON to S3                    | Easy to parse and integrate with dashboards |
| Keep table output in Jenkins logs    | Developer readability                       |
| Fail at threshold, not on detection  | Better controlled blocking                  |
| Centralize Trivy cache for agents    | Speeds up pipeline runs                     |
| Use IAM roles instead of access keys | More secure                                 |

---

# Generating an HTML Trivy Report from JSON

Sometimes developers prefer a clean HTML report instead of raw JSON. You can generate an HTML file from Trivy’s JSON output and upload that alongside the JSON to S3.

## 1. Generate JSON in Jenkins

Make sure your Trivy stage produces `trivy-report.json`:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --format json \
  --output trivy-report.json \
  ${IMAGE_NAME}
```
```python
import json
import sys

if len(sys.argv) != 3:
    print("Usage: python trivy-json-to-html.py <input-json> <output-html>")
    sys.exit(1)

input_file = sys.argv[1]
output_file = sys.argv[2]

with open(input_file, 'r') as f:
    data = json.load(f)

vulns = []
for result in data.get('Results', []):
    target = result.get('Target', 'unknown-target')
    for v in result.get('Vulnerabilities', []) or []:
        vulns.append({
            'target': target,
            'id': v.get('VulnerabilityID'),
            'pkg': v.get('PkgName'),
            'severity': v.get('Severity'),
            'installed': v.get('InstalledVersion'),
            'fixed': v.get('FixedVersion', '-'),
            'title': v.get('Title', ''),
            'desc': v.get('Description', '')[:300]  # short description
        })

html = [
    "<html>",
    "<head>",
    "  <meta charset='UTF-8'>",
    "  <title>Trivy Vulnerability Report</title>",
    "  <style>",
    "    body { font-family: Arial, sans-serif; }",
    "    table { border-collapse: collapse; width: 100%; }",
    "    th, td { border: 1px solid #ddd; padding: 8px; font-size: 13px; }",
    "    th { background-color: #f2f2f2; }",
    "    .CRITICAL { background-color: #ffdddd; }",
    "    .HIGH { background-color: #ffe5cc; }",
    "  </style>",
    "</head>",
    "<body>",
    "  <h1>Trivy Vulnerability Report</h1>",
    f"  <p>Total findings: {len(vulns)}</p>",
    "  <table>",
    "    <tr>",
    "      <th>Severity</th>",
    "      <th>ID</th>",
    "      <th>Package</th>",
    "      <th>Installed</th>",
    "      <th>Fixed</th>",
    "      <th>Target</th>",
    "      <th>Title</th>",
    "      <th>Description</th>",
    "    </tr>"
]

for v in vulns:
    sev = v['severity']
    html.append(
        f"    <tr class='{sev}'>"  # class for color
        f"<td>{sev}</td>"
        f"<td>{v['id']}</td>"
        f"<td>{v['pkg']}</td>"
        f"<td>{v['installed']}</td>"
        f"<td>{v['fixed']}</td>"
        f"<td>{v['target']}</td>"
        f"<td>{v['title']}</td>"
        f"<td>{v['desc']}</td>"
        "</tr>"
    )

html.extend([
    "  </table>",
    "</body>",
    "</html>"
])

with open(output_file, 'w') as f:
    f.write("\n".join(html))

print(f"HTML report written to {output_file}")
```
### Use the above script in Jenkinsfile
```bash
# Generate HTML from JSON
python3 scripts/trivy-json-to-html.py trivy-report.json trivy-report.html
```

