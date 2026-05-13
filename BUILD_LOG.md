# Web Application Security - BUILD_LOG.md

**Lab Execution Date**: May 2026 
**Operator**: David Mokom 
**Duration**: 13.5 hours (aggregated across 2 days)

---

## Environment Setup

- **Kali VM IP**: 192.168.9.130
- **SafeLine WAF IP**: 192.168.9.136 port 9443
- **Security Onion IP**: 192.168.9.128 port 514
- **DVWA Container**: 172.17.x.x:80

---

## Phase 1: WAF and Proxy Configuration

- Configured SafeLine to proxy traffic to the DVWA container
- Tested access by navigating to the DVWA login page through the SafeLine proxy
- Configured rate limiting policy under HTTP Flood settings
- Set the rule to 3 requests per 10 seconds with a 1 minute block
- Verified by manually refreshing the page until blocked with 403 Forbidden

---

## Phase 2: Custom Log Extraction Pipeline

**Problem Identified**: SafeLine natively exports CEF headers but hides the payload details in a local PostgreSQL database.

**Solution Implemented**:
- Created a Bash script to bypass container isolation
- Used `docker exec` to query the SafeLine database
- Extracted attack type, source IP, and URL path
- Formatted the output into CEF standard
- Piped the formatted logs to Security Onion via UDP port 514 using netcat
- Automated the script with a cron job to run every minute

**Result**: Full payload visibility achieved in SIEM via custom pipeline.

---

## Phase 3: Attack Execution

All attacks executed manually using `curl` against the DVWA application through the SafeLine proxy.

- **SQL Injection**: Payload `1' OR '1'='1`
- **Cross Site Scripting**: Payload `<script>alert(1)</script>`
- **Command Injection**: Payload `; cat /etc/passwd`
- **Local File Inclusion**: Payload `../../../etc/passwd`

**Result**: All 4 attack vectors detected and logged by SafeLine.

---

## Phase 4: ZAP Automation & Tactical Pivot

**Initial Approach**: Launched OWASP ZAP Automated Scan against the target.

**Issue Encountered**: ZAP fuzzer sent requests too rapidly and consumed all available RAM, causing a full freeze of the Kali VM.

**Tactical Pivot**: Shifted to a controlled manual Bash loop.
- Wrote a loop to send 10 curl requests with a 2 second sleep in between
- This allowed clear visibility into the exact moment the rate limit triggered
- Observed the first 3 requests return 200 OK
- Observed the 4th request return 403 Forbidden (rate limit threshold crossed)
- Requests 5-10 also returned 403 Forbidden

**Result**: Deterministic rate-limiting validation achieved.

---

## Phase 5: Detection Validation

**SafeLine Dashboard Check**:
- Confirmed the attack codes were extracted correctly
- Verified all 4 attack types logged in the PostgreSQL database

**Security Onion Hunt Interface**:
- Queried Suricata logs for the XSS attack
- Proved network layer detection independent of WAF logs

**Elastic Query**:
- Queried for successful HTTP 200 responses
- Proved legitimate traffic was passing through the WAF correctly
- Correlated with 403 Forbidden blocks to validate rate-limiting precision

**Screenshots Captured**: 11 total screenshots documenting each phase of the lab.

---

## Technical Decisions & Rationale

### Decision 1: Custom Log Pipeline Over Native Export

- SafeLine native Syslog exporter limited to CEF headers only
- Custom pipeline provided full attack payload visibility
- Demonstrated docker container networking knowledge
- Automation via cron proved production-ready thinking

### Decision 2: Manual Testing Over Continued ZAP Fuzzing

- ZAP resource exhaustion made VM unusable
- Manual loop provided granular threshold visibility
- Request-by-request observation of WAF blocking behavior
- Reproducible results for documentation and screenshot validation

### Decision 3: Three-Layer Detection Architecture

- WAF signature detection at application layer (SafeLine)
- Network-layer detection independent of WAF (Suricata)
- HTTP response code correlation for incident response (Elastic)
- Defense-in-depth validation proved completeness

---


## Lessons Learned

1. **Container Isolation**: `docker exec` is a valid method to extract logs from containerized applications when native exports are limited
2. **Resource Management**: Automated fuzzing tools must be constrained; manual testing often reveals threshold behavior more clearly
3. **Multi-Layer Detection**: No single detection layer catches everything; correlation across WAF, IDS, and SIEM provides completeness
4. **Rate-Limiting Effectiveness**: Simple threshold-based blocking is highly effective against brute-force and fuzzing attacks without requiring deep payload inspection

---

**Date Completed**: May 12, 2026 at 21:30 EDT
