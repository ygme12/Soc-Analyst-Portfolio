# Incident Report – JavaScript Code Detected in Requested URL (SOC166)

## Incident Summary

A web application security alert was generated after JavaScript code was detected within a URL request targeting a public-facing web server. The alert indicated a potential Cross-Site Scripting (XSS) attack attempt against the application's search functionality.

## Incident Details

| Field          | Value                                              |
| -------------- | -------------------------------------------------- |
| Alert Name     | SOC166 - JavaScript Code Detected in Requested URL |
| Classification | Web Application Attack                             |
| Attack Type    | Cross-Site Scripting (XSS) Attempt                 |
| Severity       | Medium                                             |
| Source IP      | 112.85.42.13                                       |
| Destination IP | 172.16.17.17                                       |
| Target Host    | WebServer1002                                      |
| Protocol       | HTTP                                               |
| HTTP Method    | GET                                                |

## Investigation Process

### Alert Review

The investigation began after a security alert identified JavaScript code embedded within a URL parameter. Such activity is commonly associated with web application attacks, particularly Cross-Site Scripting (XSS) attempts.

### Request Analysis

The suspicious request contained the following payload:

```text
/search/?q=<$script>javascript:$alert(1)<$/script>
```

The payload attempted to inject JavaScript code into the application's search parameter. The use of the JavaScript `alert()` function is a common technique used by attackers and security testers to determine whether user-supplied input is reflected and executed within a browser.

### Log Analysis

Web server logs were reviewed to identify related requests and determine whether exploitation had occurred.

Analysis revealed multiple requests originating from the same source IP address targeting the web application. HTTP response code 302 was observed, indicating that the requests were redirected rather than successfully processed by the application.

No evidence of successful script execution or compromise was identified.

### Threat Intelligence Review

The source IP address was reviewed using available threat intelligence resources. Historical information suggested previous suspicious activity associated with the IP address.

## Indicators of Compromise (IOCs)

| IOC Type        | Value                                   |
| --------------- | --------------------------------------- |
| Source IP       | 112.85.42.13                            |
| Destination IP  | 172.16.17.17                            |
| Hostname        | WebServer1002                           |
| HTTP Method     | GET                                     |
| Attack Payload  | <$script>javascript:$alert(1)<$/script> |
| Attack Category | Cross-Site Scripting (XSS)              |

## Impact Assessment

The activity represented an attempted web application attack targeting user input fields. If successful, a Cross-Site Scripting vulnerability could allow attackers to execute malicious scripts within users' browsers, potentially resulting in session theft, credential theft, or unauthorized actions.

No evidence was found indicating successful exploitation or compromise during this investigation.

## MITRE ATT&CK Mapping

| Technique ID | Technique                         |
| ------------ | --------------------------------- |
| T1190        | Exploit Public-Facing Application |
| T1059.007    | JavaScript                        |

## Containment and Recommendations

* Continue monitoring for repeated requests from the source IP.
* Implement strict input validation and sanitization.
* Deploy output encoding to prevent script execution.
* Utilize Web Application Firewall (WAF) protections.
* Conduct regular web application security assessments.
* Review web application logs for similar attack patterns.

## Final Verdict

The investigation confirmed a malicious attempt to inject JavaScript code into a web application through a URL parameter. The activity was consistent with Cross-Site Scripting (XSS) testing behavior. Analysis determined that the attack was unsuccessful, and no evidence of compromise was identified.

**Status:** Closed

**Classification:** Malicious Web Activity

**Confidence Level:** High
