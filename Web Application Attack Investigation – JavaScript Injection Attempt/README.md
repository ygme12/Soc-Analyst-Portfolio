## Scenario

A web application security alert was generated after JavaScript code was detected within a URL parameter targeting WebServer1002 (172.16.17.17).

The suspicious request originated from the external IP address 112.85.42.13 and contained the following payload:

GET /search/?q=<$script>javascript:$alert(1)<$/script>

The payload strongly indicated an attempt to test for a Cross-Site Scripting (XSS) vulnerability within the web application's search functionality.

## Investigation Findings

Analysis determined that the source host attempted to inject JavaScript code through the application's search parameter.

The payload used the classic JavaScript alert function, a common technique used by attackers and penetration testers to determine whether user-supplied input is reflected and executed within a browser.

Further review of web logs identified additional requests from the same source IP targeting the web application. Threat intelligence checks indicated that the source IP had previously been associated with suspicious activity.

Log analysis showed HTTP response code 302 and response sizes of 0 bytes for the malicious requests, indicating that the attempted attack was unsuccessful.

## Indicators of Compromise

| IOC Type       | Value                                              |
| -------------- | -------------------------------------------------- |
| Source IP      | 112.85.42.13                                       |
| Destination IP | 172.16.17.17                                       |
| Hostname       | WebServer1002                                      |
| HTTP Method    | GET                                                |
| Attack Type    | Cross-Site Scripting (XSS) Attempt                 |
| Requested URL  | /search/?q=<$script>javascript:$alert(1)<$/script> |

## MITRE ATT&CK Mapping

| Technique ID | Technique                         |
| ------------ | --------------------------------- |
| T1190        | Exploit Public-Facing Application |
| T1059.007    | JavaScript                        |

## Final Verdict

Classification: Malicious

Severity: Medium

The investigation confirmed an attempted JavaScript injection attack consistent with Cross-Site Scripting (XSS) testing activity against a public-facing web application. Analysis indicated that the attack was unsuccessful and no evidence of compromise was identified.
