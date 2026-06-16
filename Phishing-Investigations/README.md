
# Phishing Email Investigation – Malicious Excel 4.0 Macro Document

## Overview

This project documents the investigation of a phishing email containing a malicious Microsoft Excel attachment that leveraged Excel 4.0 macros. The objective was to determine whether the attachment was malicious, identify indicators of compromise (IOCs), assess the risk to the organization, and recommend remediation actions.

## Scenario

A security alert was generated indicating that a phishing email containing an Excel document had been delivered to a user. Initial analysis suggested the attachment contained Excel 4.0 macros, a technique frequently used by threat actors to execute malicious code and evade traditional detection mechanisms.

## Investigation Objectives

* Analyze the phishing email.
* Examine the attached Excel document.
* Identify malicious behavior associated with Excel 4.0 macros.
* Extract and validate indicators of compromise.
* Determine the severity of the threat.
* Recommend containment and remediation actions.

## Tools Used

* Let's Defend SIEM
* VirusTotal
* Threat Intelligence Sources
* Email Header Analysis Tools
* Microsoft Office Analysis Techniques

## Investigation Process

### Step 1 – Alert Review

The alert was reviewed to identify the affected user, sender information, attachment details, and any associated indicators.

### Step 2 – Email Analysis

The email content was examined for phishing characteristics including:

* Suspicious sender information
* Social engineering techniques
* Urgent language
* Unexpected attachments

### Step 3 – Attachment Analysis

The attached Excel file was analyzed to determine whether it contained embedded macros or malicious functionality.

Key findings included:

* Presence of Excel 4.0 macro functionality
* Potential execution of malicious commands
* Behavior consistent with malware delivery techniques

### Step 4 – Threat Intelligence Validation

Indicators extracted from the attachment and email were checked against external intelligence sources to identify known malicious activity.

### Step 5 – Risk Assessment

The document presented a significant security risk because Excel 4.0 macros are commonly used to deliver malware, establish persistence, and initiate additional malicious activity.

## Indicators of Compromise

| IOC Type        | Value                                                           |
| --------------- | ---------------------------                                     |
| Sender Email    | trenton@tritowncomputers.com                                    |
| Attachment Name |11f44531fb088d31307d87b01e8eabff.zip                             |
| File Hash       |d05014be92f45cc92ac01b42acc21c9889f30778b823e259ac3b6df17daf37c9 |
| Domain          | Document from investigation                                     |
| IP Address      | 222.227.81.181                                                  |

## MITRE ATT&CK Mapping

| Technique ID | Technique                         |
| ------------ | --------------------------------- |
| T1566.001    | Spearphishing Attachment          |
| T1204.002    | User Execution: Malicious File    |
| T1059        | Command and Scripting Interpreter |

## Findings

The investigation determined that the email was a phishing attempt designed to deliver a malicious Excel document containing Excel 4.0 macros. The attachment posed a significant risk and could potentially result in malware execution if opened by the recipient.

## Recommendations

* Block the malicious sender.
* Quarantine similar emails.
* Block identified indicators of compromise.
* Disable unnecessary macro execution where possible.
* Conduct user awareness training on phishing threats.
* Monitor for additional activity related to the campaign.

## Lessons Learned

This investigation demonstrated the importance of analyzing email attachments, validating indicators through threat intelligence, and understanding legacy macro techniques that continue to be abused by attackers.

## Skills Demonstrated

* Phishing Email Analysis
* Threat Intelligence
* IOC Analysis
* Malware Triage
* Email Security
* Incident Response
* MITRE ATT&CK Mapping
