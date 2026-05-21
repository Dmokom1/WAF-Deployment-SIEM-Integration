# Build Notes
# Web Application Security: SafeLine WAF Deployment and SIEM Integration

This file provides supporting build context for the main README. It documents the lab sequence, important artifacts, validation points, evidence interpretation, and screenshot mapping.

The README explains the full project story. These notes focus on how the lab was built, what was observed, and what the screenshots support.

---

## Purpose of This File

This project was built to practice web application security monitoring using DVWA, SafeLine WAF, Security Onion, Elastic, and OWASP ZAP.

These build notes focus on:

- DVWA application access
- SafeLine WAF deployment and review
- Syslog forwarding into Security Onion
- SafeLine attack record review
- Elastic detection rule configuration
- OWASP ZAP web testing traffic
- SafeLine rate-limiting validation
- Security Onion hunt visibility
- Screenshot-supported evidence

---

## Lab Environment

| Component | Details |
|---|---|
| Test Host | Kali Linux |
| Vulnerable Application | DVWA |
| WAF | SafeLine WAF Community Edition |
| SIEM / Log Review | Security Onion |
| Detection Review | Elastic / Kibana |
| Network Visibility | Suricata / Security Onion Hunt |
| Web Testing Tool | OWASP ZAP |
| Container Access | Docker / PostgreSQL query through `docker exec` |

---

## Corrected Lab Sequence

The final lab workflow followed this order:

1. Confirmed SafeLine WAF was active.
2. Confirmed DVWA was reachable in the lab.
3. Confirmed DVWA could be accessed through the SafeLine-facing path.
4. Reviewed SafeLine-related syslog forwarding configuration.
5. Confirmed Security Onion received syslog events.
6. Queried SafeLine PostgreSQL attack records using `docker exec`.
7. Created an Elastic detection rule for selected SafeLine attack-type codes.
8. Ran OWASP ZAP against the SafeLine-facing application path.
9. Reviewed SafeLine rate-limiting behavior.
10. Queried Security Onion Hunt for XSS-related activity.
11. Queried Security Onion Hunt for successful event activity.

---

## Important Artifacts and Values

| Item | Value |
|---|---|
| SafeLine dashboard path shown | `127.0.0.1:9443/statistics` |
| DVWA direct lab path shown | `127.0.0.1:8080/index.php` |
| SafeLine-facing application path shown | `192.168.9.136/dvwa` |
| Syslog destination shown | `192.168.9.131:514` |
| SafeLine PostgreSQL container | `safeline-pg` |
| SafeLine database | `safeline-ce` |
| SafeLine table queried | `public.mgt_detect_log_basic` |
| Fields queried | `attack_type`, `src_ip`, `url_path` |
| Attack-type values shown | `0`, `1`, `9`, `11` |
| ZAP target shown | `http://192.168.9.136` |
| ZAP requests shown | `1,913` |
| ZAP alerts shown | `88` |
| Rate limit reason shown | `3 Reqs within 10 seconds` |
| Rate limit action shown | `Block 1 minutes` |
| XSS Hunt query shown | `message: *XSS*` |
| Success Hunt query shown | `event.outcome: "success"` |

---

## Phase 1: SafeLine WAF and DVWA Access

SafeLine WAF was active and reachable in the lab environment. DVWA was also reachable and used as the vulnerable web application for testing.

### Screenshot Evidence

| Screenshot | What It Supports |
|---|---|
| `screenshots/01_SafeLine_WAF_Dashboard_Active.png` | SafeLine WAF dashboard active |
| `screenshots/02_DVWA_Login_Success.png` | DVWA application reachable in the lab |
| `screenshots/03_DVWA_Protected_via_SafeLine_WAF.png` | DVWA reachable through the SafeLine-facing path |

### Observation

This phase confirmed that the web application and WAF components were reachable.

The screenshots support application access and WAF availability. They do not prove detection or blocking by themselves.

---

## Phase 2: Syslog Forwarding and Security Onion Reception

SafeLine-related logging was configured to forward syslog data to the SIEM environment.

### Screenshot Evidence

| Screenshot | What It Supports |
|---|---|
| `screenshots/04_SafeLine_Syslog_Configuration.png` | Syslog forwarding configured in Nginx logging |
| `screenshots/05_Security_Onion_Syslog_Reception.png` | Security Onion receiving syslog events |

### Observation

The syslog configuration showed logging directed to:

`192.168.9.131:514`

Security Onion showed `syslog.syslog` events in Hunt.

This supports syslog reception in Security Onion. It should not be described as complete WAF field parsing or full payload visibility by itself.

---

## Phase 3: SafeLine Attack Record Review

SafeLine attack records were queried from the PostgreSQL container using `docker exec`.

### Screenshot Evidence

| Screenshot | What It Supports |
|---|---|
| `screenshots/06_SafeLine_Attack_Codes_Extraction.png` | PostgreSQL query returning SafeLine attack records |

### Query Reviewed

```sql
SELECT attack_type, src_ip, url_path
FROM public.mgt_detect_log_basic
ORDER BY id DESC
LIMIT 4;
```

### Observation

The query returned attack metadata from SafeLine, including:

- `attack_type`
- `src_ip`
- `url_path`

The visible attack-type values included:

- `0`
- `1`
- `9`
- `11`

The URL paths showed suspicious patterns such as encoded traversal, script tags, and password-file access attempts.

This supports the claim that SafeLine stored useful attack metadata in PostgreSQL. It does not prove full payload visibility across all traffic.

---

## Phase 4: Elastic Detection Rule Configuration

An Elastic detection rule was configured to match selected SafeLine attack-type codes in message data.

### Screenshot Evidence

| Screenshot | What It Supports |
|---|---|
| `screenshots/07_Elastic_Detection_Rule_Configuration.png` | Elastic rule matching selected SafeLine attack-type codes |

### Rule Logic

```kql
message: "attack_type: 0" OR
message: "attack_type: 1" OR
message: "attack_type: 9" OR
message: "attack_type: 11"
```

### Observation

The rule was configured as a query rule with critical severity.

The rule matched selected SafeLine attack-type codes related to common web attack categories such as SQL injection, XSS, command injection, and local file inclusion.

This supports custom detection logic based on observed SafeLine log content. It should not be described as a complete web attack detection system.

---

## Phase 5: OWASP ZAP Web Testing

OWASP ZAP was used to generate automated web testing traffic against the SafeLine-facing application path.

### Screenshot Evidence

| Screenshot | What It Supports |
|---|---|
| `screenshots/08a_ZAP_Fuzzer_Traffic.png` | OWASP ZAP automated scan traffic completed |

### Observation

The ZAP screenshot showed a completed scan against:

`http://192.168.9.136`

The scan generated:

- `1,913` requests
- `88` alerts

This supports that ZAP generated high-volume web testing traffic.

The screenshot does not prove that every ZAP alert was manually validated or that the Kali VM froze. Those claims should not be included unless separately documented.

---

## Phase 6: SafeLine Rate-Limiting Review

SafeLine rate-limiting behavior was reviewed after repeated web request activity.

### Screenshot Evidence

| Screenshot | What It Supports |
|---|---|
| `screenshots/08b_WAF_Rate_Limiting_Backend.png` | SafeLine rate-limiting entries and block action |

### Observation

The SafeLine rate-limiting page showed entries for:

`192.168.9.136`

The reason shown was:

`3 Reqs within 10 seconds, Basic Access Limit was triggered`

The action shown was:

`Block 1 minutes`

This confirms that SafeLine rate limiting triggered during lab testing.

This should be described as observed rate-limiting behavior in the lab, not full DDoS protection or production-grade bot mitigation.

---

## Phase 7: Security Onion XSS-Related Hunt Review

Security Onion Hunt was used to search for XSS-related activity.

### Screenshot Evidence

| Screenshot | What It Supports |
|---|---|
| `screenshots/09_Suricata_XSS_Alert.png` | Security Onion Hunt query for XSS-related activity |

### Query Used

```text
message: *XSS*
```

### Observation

The screenshot showed XSS-related hunt activity over the selected time window.

This provided a network/security-monitoring view alongside the WAF records.

The screenshot should not be overstated as proof that Suricata detected every web attack type tested.

---

## Phase 8: HTTP Success Event Review

Security Onion Hunt was also used to review successful event activity.

### Screenshot Evidence

| Screenshot | What It Supports |
|---|---|
| `screenshots/10_HTTP_Success_Validation.png` | Security Onion Hunt query for successful event activity |

### Query Used

```text
event.outcome: "success"
```

### Observation

The screenshot showed successful event activity over the selected time window, including a visible spike around the testing period.

This supports review of allowed or successful activity in Security Onion.

The screenshot does not prove perfect correlation between every allowed request and every blocked request.

---

## Evidence Interpretation Notes

These notes keep the project explanation accurate:

- SafeLine was active and DVWA was reachable in the lab.
- DVWA was shown both directly and through the SafeLine-facing path.
- Syslog forwarding was configured and Security Onion received syslog events.
- The PostgreSQL query showed SafeLine attack metadata, not complete traffic visibility.
- The Elastic rule matched selected SafeLine attack-type codes in message data.
- ZAP generated high-volume web testing traffic, but individual findings were not fully validated in the screenshots.
- SafeLine rate limiting triggered during lab testing.
- Security Onion Hunt showed XSS-related activity and successful event activity.
- The project should be described as a controlled lab workflow, not a production WAF deployment.

---

## Key Lessons Learned

1. WAF testing should include both blocking behavior and log visibility.
2. Syslog reception does not automatically mean every field is parsed cleanly.
3. Querying backend records can help validate what a security tool stores internally.
4. Detection rules should match evidence that actually exists in the logs.
5. Automated scanners like ZAP can generate useful traffic, but the results still need careful interpretation.
6. Rate limiting should be validated with observed behavior, not assumed from configuration alone.
7. Security Onion Hunt can provide supporting network/security telemetry alongside WAF records.
8. Public documentation should separate what was observed from what would need more validation.

---

## Improvements for a Future Version

Future improvements could include:

- Exporting the exact log forwarding script or configuration in a safe, documented way.
- Normalizing SafeLine fields into structured Elastic fields instead of matching on message text.
- Capturing the alert result generated by the custom Elastic rule.
- Capturing clearer request and response examples for allowed versus blocked traffic.
- Testing each attack type separately and mapping each one to its SafeLine attack code.
- Adding packet-level evidence for selected web attacks.
- Adding a table that maps each test action to SafeLine, Security Onion, and Elastic evidence.
- Reducing reliance on broad message searches by building cleaner field-based detections.
- Testing the rule with benign traffic to understand false-positive risk.

---

## Screenshot Map

| Screenshot | What It Supports |
|---|---|
| `screenshots/01_SafeLine_WAF_Dashboard_Active.png` | SafeLine WAF dashboard active |
| `screenshots/02_DVWA_Login_Success.png` | DVWA application reachable in the lab |
| `screenshots/03_DVWA_Protected_via_SafeLine_WAF.png` | DVWA reachable through the SafeLine-facing path |
| `screenshots/04_SafeLine_Syslog_Configuration.png` | Syslog forwarding configured in Nginx logging |
| `screenshots/05_Security_Onion_Syslog_Reception.png` | Security Onion receiving syslog events |
| `screenshots/06_SafeLine_Attack_Codes_Extraction.png` | PostgreSQL query returning SafeLine attack records |
| `screenshots/07_Elastic_Detection_Rule_Configuration.png` | Elastic rule matching selected SafeLine attack-type codes |
| `screenshots/08a_ZAP_Fuzzer_Traffic.png` | OWASP ZAP automated scan traffic completed |
| `screenshots/08b_WAF_Rate_Limiting_Backend.png` | SafeLine rate-limiting entries and block action |
| `screenshots/09_Suricata_XSS_Alert.png` | Security Onion Hunt query for XSS-related activity |
| `screenshots/10_HTTP_Success_Validation.png` | Security Onion Hunt query for successful event activity |