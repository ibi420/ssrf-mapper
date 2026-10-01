# ssrf-mapper — Automated SSRF Vulnerability Mapping & Testing Toolkit

> An automated Bash-based testing suite to map API attack surfaces, fuzz endpoints with SSRF payloads, probe cloud metadata services, and generate audit reports.

---

> [!NOTE]
> This tool was originally created to audit and security-test [doubtfire-api](https://github.com/thoth-tech/doubtfire-api), the backend API supporting the Doubtfire / OnTrack learning management system. The default wordlists ([api_endpoints.txt](api_endpoints.txt) and [payloads.txt](payloads.txt)) and port configurations include endpoints and test vectors tailored for auditing that architecture as well as general web APIs.

---

## Table of Contents

- [ssrf-mapper — Automated SSRF Vulnerability Mapping \& Testing Toolkit](#ssrf-mapper--automated-ssrf-vulnerability-mapping--testing-toolkit)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Test Cases](#test-cases)
  - [Prerequisites](#prerequisites)
  - [Quick Start](#quick-start)
  - [Repository Structure](#repository-structure)
  - [Understanding the Results](#understanding-the-results)
    - [Response Classifications](#response-classifications)
  - [Methodologies \& Architecture](#methodologies--architecture)
    - [1. Parameter Fuzzing](#1-parameter-fuzzing)
    - [2. Controlled Payloads](#2-controlled-payloads)
    - [3. HTTP Request Automation](#3-http-request-automation)
    - [4. Response Analysis](#4-response-analysis)
    - [5. Logging and Reporting](#5-logging-and-reporting)
  - [Evolution History](#evolution-history)
  - [Remediation Recommendations](#remediation-recommendations)

---

## Overview

Server-Side Request Forgery (SSRF) occurs when an attacker manipulates a server-side application into sending HTTP requests to an unexpected or internal destination. In applications that rely on external API calls, webhooks, or remote asset fetching, SSRF can allow an attacker to:

- Access internal services (such as localhost APIs, databases, and internal cache instances).
- Query cloud instance metadata endpoints (such as AWS `http://169.254.169.254/latest/meta-data/` or Google Cloud metadata).
- Bypass perimeter firewalls and network segmentation.
- Probe internal subnets and open TCP ports.
- Escalate into Remote Code Execution (RCE) via internal service protocols.

`ssrf-mapper` automates the discovery, probing, and reporting of SSRF vectors across candidate API endpoints.

---

## Test Cases

| Test                      | Description                                                                             | Payload Examples                                                  |
| ------------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| **URL Injection**         | Inject internal addresses into parameters (e.g. `url`, `redirect`, `callback`)          | `http://127.0.0.1:80`, `http://169.254.169.254/latest/meta-data/` |
| **Header Manipulation**   | Insert custom headers (`Host`, `X-Forwarded-For`) to bypass basic reverse-proxy filters | `X-Forwarded-For: 127.0.0.1`                                      |
| **Large File Response**   | Force the server to download large responses to evaluate resource exhaustion            | Large local files (e.g. `/var/log/secure`)                        |
| **Timeout / Slow Server** | Test application behavior and worker starvation when internal ports hang                | Connections to filtered or dropped internal ports                 |
| **Protocol Probing**      | Probe for non-HTTP internal TCP services using alternate URI schemes                    | `redis://127.0.0.1:6379`, `mysql://127.0.0.1:3306`                |

---

## Prerequisites

The script runs in any Linux / Bash environment (including WSL on Windows) and requires:

- **Bash** (v4.0+)
- **cURL** (command-line HTTP client)
- **jq** (command-line JSON processor; see [jq GitHub Releases](https://github.com/stedolan/jq/releases))
- **grep** (standard keyword matching utility)

Optional tools for deeper investigation:
- **Burp Suite** (configure as proxy to inspect and replay flagged requests)
- **Docker** (for hosting target API containers locally)

---

## Quick Start

1. **Make the script executable:**
   ```bash
   chmod +x ssrf_mapper.sh
   ```

2. **Run the tool:**
   ```bash
   ./ssrf_mapper.sh
   ```

3. **Follow the interactive prompts:**
   - **Target Base URL:** e.g. `http://localhost`
   - **Target Port:** e.g. `4200`
   - **API Endpoints Wordlist:** Path to endpoint list (default: `api_endpoints.txt`)
   - **SSRF Payloads Wordlist:** Path to payload list (default: `payloads.txt`)
   - **Max Time Per Request:** Timeout in seconds (e.g. `5`)
   - **HTTP Method:** `GET` or `POST`

---

## Repository Structure

- **`ssrf_mapper.sh`** — The primary automation script for endpoint discovery and payload fuzzing.
- **`api_endpoints.txt`** — Target endpoint candidate list (seeded with [doubtfire-api](https://github.com/thoth-tech/doubtfire-api) routes).
- **`payloads.txt`** — Curated SSRF payload list containing localhost vectors, cloud metadata targets, protocol schemas, and obfuscated addresses.
- **`ssrf_logs/`** — Generated output directory containing scan result Markdown reports and timeout logs (ignored by git).

---

## Understanding the Results

When testing completes, output reports are saved in the `ssrf_logs/` directory:
- **Test Results:** `ssrf_logs/ssrf_test_results_<timestamp>.md`
- **Slow Endpoints:** `ssrf_logs/slow_endpoints_<timestamp>.md`

Console output uses color coding:
- **Red lines (`[!]`):** Indicate potential SSRF vectors where an internal response or sensitive keyword was returned.
- **Yellow lines:** Highlight timeouts, unusual status codes, or redirected paths.
- **Green lines:** Indicate clean validation where requests were safely rejected.

### Response Classifications
- **Expected / Clean:** Target returns explicit validation errors, `400 Bad Request`, `403 Forbidden`, or `404 Not Found`.
- **Potential SSRF:** Target returns `200 OK` or `302 Found` with body content reflecting internal IP addresses or cloud metadata.
- **Critical SSRF:** Target echoes cloud metadata attributes (e.g. IAM credentials, instance attributes) or internal configuration files.

---

## Methodologies & Architecture

### 1. Parameter Fuzzing
The script iterates across candidate endpoints and injects payloads into parameters commonly vulnerable to request forgery (e.g. `url`, `redirect`, `dest`, `callback`, `webhook`).

### 2. Controlled Payloads
Payloads prioritize non-destructive indicators to verify access without disrupting production or test services:
- Loopback addresses (`127.0.0.1`, `localhost`, alternate decimal/hex encodings).
- Cloud metadata service addresses (`169.254.169.254`, `metadata.google.internal`).
- Protocol scheme variants (`file://`, `redis://`, `mysql://`).
- Reference wordlists adapted from curated security collections such as [h0tak88r/Wordlists](https://github.com/h0tak88r/Wordlists).

### 3. HTTP Request Automation
Requests are piped through `curl` with configured timeouts (`--max-time`) to ensure slow or hanging internal connections do not stall the scan. Custom headers (such as `X-Forwarded-For` and `Host`) can be injected to test proxy bypass logic.

### 4. Response Analysis
Responses are examined for indicators of internal network access:
- HTTP status codes.
- Response body keywords (e.g. `latest/meta-data`, `AMI`, `computeMetadata`, database banners).
- Latency discrepancies indicating internal host availability.

### 5. Logging and Reporting
All probe attempts are compiled into Markdown documents with timestamped entries, making it simple to export findings directly to issue trackers or audit documentation.

---

## Evolution History

| Phase                | Changes Introduced                                                                                    | Objective                                                  |
| -------------------- | ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Initial Version**  | Manual `curl` requests per individual endpoint                                                        | Initial concept validation                                 |
| **Second Iteration** | Automated loop across endpoints and parameters                                                        | Broadened audit surface without repetitive manual commands |
| **Third Iteration**  | Wordlist integration and structured console logging                                                   | Improved testing repeatability                             |
| **Fourth Iteration** | Header fuzzing and payload obfuscation                                                                | Simulated filter bypass techniques                         |
| **Fifth Iteration**  | Background execution and detailed logging                                                             | Faster scans on larger endpoint collections                |
| **Final Version**    | Prerequisite checks, strict input validation, configurable timeout limits, and Markdown report output | Production-ready triage tool with clean audit traceability |

---

## Remediation Recommendations

1. **Strict URL Allow-listing:** Reject arbitrary user-supplied schemes and hostnames; allow requests only to explicit, trusted external domains.
2. **Block Private IP Ranges:** Enforce network-layer and application-layer blocks on private IP ranges (RFC 1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) and link-local addresses (`169.254.0.0/16`).
3. **Disable Unused URI Schemes:** Restrict HTTP client libraries to `http` and `https` only; explicitly disable `file://`, `gopher://`, `dict://`, and protocol wrappers.
4. **Enforce Short Timeouts:** Set aggressive connect and read timeouts on server-side HTTP requests to prevent worker exhaustion.
5. **IMDSv2 Migration:** In cloud environments (such as AWS), require IMDSv2 (session-token-based metadata requests) to mitigate simple SSRF queries to `169.254.169.254`.
