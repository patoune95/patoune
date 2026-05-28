---
title: "Shortening patch cycles: why 48h is no longer optional for critical CVEs"
date: 2026-05-28T10:00:00+02:00
description: "The most critical vulnerabilities are being exploited within hours. Here is how to restructure your patch cycles to respond in under 48 hours."
draft: false
author: Thomas L.
tags:
  - security
  - patch-management
  - vulnerability
  - devops
  - ci-cd
cover:
  image: /mythos/cover.jpg
  alt: "Critical CVE response pipeline, Mythos"
  relative: false
---

## Context: decades-old vulnerabilities, still exploitable

The Mythos Preview report (April 2026) highlighted an uncomfortable reality: among the 10,000+ vulnerabilities identified, the most critical ones are not necessarily the most recent. The SACK TCP bug in OpenBSD has been around for 27 years. The FFmpeg H.264 buffer overflow for 16 years. The FreeBSD NFS stack overflow for 17 years.

What has changed is the speed at which these flaws can now be exploited. With AI-assisted exploitation tools, the window between a patch being published and active exploitation is now measured in hours, not weeks.

Monthly or weekly patch cycles are no longer acceptable for critical CVEs.

## The problem: the exploitation window is shrinking

Most organisations still operate on patch cycles driven by operational logic: a weekly maintenance window, a monthly Patch Tuesday cycle, or worse, a change management process that adds several weeks between patch publication and deployment.

This model made sense in a world where exploiting a vulnerability required weeks of research and analysis. That is no longer the world we operate in.

The three main risks:

- **Automated exploitation**: AI models can analyse a published patch, infer the underlying vulnerability through diff analysis, and generate an exploit within hours
- **Asymmetry of means**: the attacker only needs to succeed once, the defender must block every attempt
- **Public visibility**: every CVE published with a CVSS score ≥ 9.0 immediately generates massive attention in the offensive security community

> ⚠️ The FreeBSD NFS stack overflow (CVE-2026-4747) allows unauthenticated RCE via 6 sequential packets. On an exposed NFS infrastructure, an attacker can compromise a machine in seconds once the exploit is available.

## The solution: a criticality-driven patch pipeline

### Define SLOs by severity level

The first step is to formalise patching SLOs (Service Level Objectives) that create an internal contractual obligation:

| CVSS Severity | Maximum delay | Deployment mode |
|---|---|---|
| Critical (≥ 9.0) | 48h | Emergency deployment, bypass standard change management |
| High (7.0 – 8.9) | 7 days | Accelerated deployment with lightweight review |
| Medium (4.0 – 6.9) | 30 days | Normal cycle |
| Low (< 4.0) | 90 days | Next planned release |

These SLOs must be tooled, not just documented. A known critical CVE unpatched beyond 48h must trigger an automatic alert.

### Automating detection and qualification

The pipeline starts with continuous CVE feed monitoring:

```bash
# Example: querying NVD API for critical CVEs
curl -s "https://services.nvd.nist.gov/rest/json/cves/2.0?cvssV3Severity=CRITICAL&resultsPerPage=10" \
  | jq '.vulnerabilities[].cve | {id: .id, published: .published, description: .descriptions[0].value}'
```

In practice, you do not query the NVD API manually. You integrate a scanning tool into the CI/CD pipeline:

- **Trivy** for container images and application dependencies
- **Grype** as an alternative to Trivy for CVE scanning
- **Syft** for generating auditable SBOMs (Software Bill of Materials)
- **Dependabot** for source code dependencies

The goal is to receive an alert the moment a CVE is published, not at the next scheduled scan.

### Structuring the response pipeline

An effective emergency patch pipeline looks like this:

![Security workflow](security_pipeline.svg)

- 🔴 = Urgent (CVE, alerts, deployment)
- 🟠 = Qualification phase
- 🟣 = Decision point
- 🟢 = Success (monitoring, closed)
- 🟡 = "Exposed + Patch" branch

### Quick qualification: avoiding false urgency

Not every critical CVE requires immediate action. Qualification must answer three questions:

1. **Is the vulnerable component present in our stack?** A buffer overflow in OpenBSD is not urgent if the infrastructure runs on Ubuntu.
2. **Is the attack surface exposed?** An NFS RCE is critical only if NFS is accessible from an untrusted network.
3. **Is a patch available?** If not, what temporary mitigations can be applied immediately?

> 💡 A component inventory (SBOM, Software Bill of Materials) is a prerequisite to answer the first question in under 30 minutes. Without a SBOM, qualification becomes a manual search that consumes precisely the time we cannot afford to lose.

## In practice: integrating Trivy into a CI/CD pipeline

Here is an example GitHub Actions integration that blocks deployment if a critical CVE is detected:

```yaml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 */6 * * *'  # scan every 6h

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Scan image for critical CVEs
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:latest'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL'
          ignore-unfixed: true
```

The `exit-code: '1'` option blocks the pipeline if an unpatched critical CVE is detected. `ignore-unfixed: true` avoids blocking on vulnerabilities for which no patch exists yet; those cases can be tracked separately.

For application dependencies, Dependabot can automatically open update PRs:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10
    groups:
      security-patches:
        applies-to: security-updates
        update-types:
          - "patch"
          - "minor"
```

## Trivy: a powerful tool with a troubled track record

⚠️ **Trivy was compromised twice in 2026.**

A first breach on February 28 via a misconfigured GitHub Actions workflow, then a major attack on March 19 by the TeamPCP group: 76 release tags retroactively poisoned, malicious binaries distributed via GitHub Releases, Docker Hub, and Amazon ECR.

The infected versions stole CI/CD secrets and deployed a systemd persistence mechanism. Safe versions are `v0.69.3+` for the scanner, `v0.35.0+` for `trivy-action`, and `v0.2.6+` for `setup-trivy`. Before integrating Trivy into your pipeline, check the latest Aqua Security advisory and consider alternatives depending on your context. (sources at the bottom)

## Grype and Syft: a SBOM-first alternative

Given the Trivy compromises, **Grype** and **Syft** (both maintained by Anchore) are a solid and coherent alternative.

The philosophy differs from Trivy: instead of scanning the image or repository directly, the two steps are separated.

1. **Syft** generates a SBOM, a comprehensive inventory of all components present in the artefact
2. **Grype** scans that SBOM to detect known CVEs

This separation has a concrete advantage: the SBOM can be stored, versioned, and reused for future scans without rebuilding the image.

### Generating a SBOM with Syft

```bash
# Scan a container image
syft my-app:latest -o spdx-json > sbom.spdx.json

# Scan a directory (source code, dependencies)
syft dir:. -o spdx-json > sbom.spdx.json

# CycloneDX: more compact, well supported by downstream tools
syft my-app:latest -o cyclonedx-json > sbom.cdx.json
```

SPDX is the recommended format for interoperability. CycloneDX is more compact and better supported by some analysis tools.

### Scanning the SBOM with Grype

```bash
# Scan a container image directly
grype my-app:latest

# Scan an existing SBOM
grype sbom:./sbom.spdx.json

# Block only on critical CVEs with an available fix
grype sbom:./sbom.spdx.json --only-fixed --fail-on critical
```

The `--fail-on critical` option exits with a non-zero return code if a critical CVE is found, directly usable in a CI/CD pipeline.

### GitHub Actions integration

```yaml
name: SBOM + Vulnerability Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 */6 * * *'

jobs:
  sbom-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: my-app:latest
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Scan for CVEs
        uses: anchore/scan-action@v3
        with:
          sbom: sbom.spdx.json
          fail-build: true
          severity-cutoff: critical
          only-fixed: true

      - name: Store SBOM as artefact
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json
```

Storing the SBOM as a build artefact allows querying it when a CVE is published later, without rebuilding the image.

### Querying the SBOM for rapid qualification

When a CVE is published, the first question to answer is: "do we use this component?" With a SBOM available:

```bash
# Check for a specific component
cat sbom.spdx.json | jq '.packages[] | select(.name == "openssl") | {name, versionInfo}'

# Quick tabular view
syft my-app:latest -o table | grep -i ffmpeg
```

Without a SBOM, this check becomes a manual search through Dockerfiles, lock files, and transitive dependencies, precisely the kind of operation that causes the 2-hour qualification window to be exceeded.

## Best practices

### Separate the emergency patch lane from standard change management

Change management exists for a good reason: avoiding regressions in production. But a two-week validation process is incompatible with a 48h SLO for critical CVEs.

The solution is not to remove change management, but to define a **fast lane** for critical security patches:

- Review by two people instead of CAB approval
- Mandatory automated tests as a merge condition
- Canary deployment (5-10% of traffic) for 2h before full rollout
- Automatic rollback if error metrics increase

### Automate regression testing to deploy safely at speed

You cannot move fast if you are afraid of breaking things. The counterpart of an emergency deployment is a solid automated test suite:

- **Smoke tests**: verify the application starts and responds
- **Integration tests** on critical paths (authentication, payments, etc.)
- **Contract tests** for exposed APIs

Without these tests, every security patch is an operational risk that pushes teams to delay deployments. That is exactly the opposite of what we are trying to achieve.

### Maintain an up-to-date component inventory (SBOM)

A SBOM generated at every build and stored with the artefact allows answering in seconds: "do we use this vulnerable component?". The [Grype and Syft](#grype-and-syft-a-sbom-first-alternative) section covers how to set this up in practice.

## Conclusion

The pressure exerted by automated exploitation tools forces a fundamental rethinking of patch cycles. 48h for critical CVEs is not an ambitious target: it is the minimum required to maintain an acceptable security posture.

What makes this target achievable in practice:

- Formalised SLOs that create an obligation, not just a recommendation
- A continuous scanning pipeline that detects CVEs in real time
- An emergency deployment lane decoupled from standard change management
- Sufficient automated tests to deploy quickly without taking unnecessary risks
- An up-to-date SBOM to quickly qualify exposure

The most frequently overlooked point is the SBOM. Without precise knowledge of what is running in production, all the upstream mechanics are slowed down by a manual qualification phase that consumes precisely the time we cannot afford to waste.

## Links

- [NVD: National Vulnerability Database](https://nvd.nist.gov/)
- [Trivy: Open source vulnerability scanner](https://github.com/aquasecurity/trivy)
- [Grype: Anchore CVE scanner](https://github.com/anchore/grype)
- [Syft: SBOM generator](https://github.com/anchore/syft)
- [Dependabot: GitHub docs](https://docs.github.com/en/code-security/dependabot)
- [SPDX: SBOM standard](https://spdx.dev/)
- [OpenSSF Scorecard: Open source project security health](https://securityscorecards.dev/)

## Sources: Trivy compromise

- [GitHub Advisory GHSA-69fq-xp46-6x23](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23)
- [Aqua Security: full investigation](https://www.aquasec.com/blog/trivy-supply-chain-attack-what-you-need-to-know/)
- [StepSecurity: 2nd compromise](https://www.stepsecurity.io/blog/trivy-compromised-a-second-time---malicious-v0-69-4-release)
- [CrowdStrike: technical analysis](https://www.crowdstrike.com/en-us/blog/from-scanner-to-stealer-inside-the-trivy-action-supply-chain-compromise/)

## Acronyms

| Acronym | Meaning |
|---|---|
| API | Application Programming Interface |
| CAB | Change Advisory Board |
| CI/CD | Continuous Integration / Continuous Delivery |
| CVE | Common Vulnerabilities and Exposures |
| CVSS | Common Vulnerability Scoring System |
| ECR | Elastic Container Registry, Amazon container image registry |
| AI | Artificial Intelligence |
| NFS | Network File System |
| NIST | National Institute of Standards and Technology |
| NVD | National Vulnerability Database, official CVE database maintained by NIST |
| PR | Pull Request |
| RCE | Remote Code Execution |
| SACK | Selective ACKnowledgement, TCP selective acknowledgement mechanism |
| SBOM | Software Bill of Materials |
| SLO | Service Level Objective |
