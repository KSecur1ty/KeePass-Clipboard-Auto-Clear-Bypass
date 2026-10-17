# KeePass Clipboard Auto-Clear Bypass When Windows Is Locked

A security vulnerability affecting the automatic clipboard-clearing mechanism in KeePass, allowing sensitive data to remain accessible beyond the configured timeout when Windows is locked.

| Field | Details |
|---|---|
| **Vendor** | Dominik Reichl / KeePass |
| **Affected products** | KeePass Password Safe 2.x and 1.x |
| **Affected versions** | KeePass 2.61.1 and KeePass 1.43 |
| **Fixed versions** | KeePass 2.62 and KeePass 1.44 (announced by the developer) |
| **Initial disclosure** | 14 September 2026 |
| **Operating system tested** | Windows 11 |
| **Status** | Vendor notified; fix verified in a development build |

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Affected Versions](#2-affected-versions)
- [3. Technical Description](#3-technical-description)
- [4. Steps to Reproduce](#4-steps-to-reproduce)
- [5. Verification and Additional Testing](#5-verification-and-additional-testing)
- [6. Security Impact and Attacker Scenarios](#6-security-impact-and-attacker-scenarios)
- [7. Root Cause and Remediation](#7-root-cause-and-remediation)
- [8. Responsible Disclosure Timeline](#8-responsible-disclosure-timeline)
- [9. Conclusion](#9-conclusion)

---

## 1. Executive Summary

KeePass contains a security vulnerability in its automatic clipboard-clearing mechanism that can cause copied credentials and other sensitive entry data to remain accessible well beyond the configured clipboard timeout when Windows is locked.

The vulnerability was reproduced in **KeePass 2.61.1 on Windows 11**. Following the initial report, the developer confirmed that the same issue also affected **KeePass 1.43**.

When a user copies a value from KeePass and locks Windows before the configured clipboard timeout expires, KeePass may fail to clear the clipboard when the deadline passes. The copied value remains accessible after the Windows session is unlocked, even when the KeePass workspace is locked and the source database is no longer open.

The issue was confirmed using ordinary paste operations and PowerShell's `Get-Clipboard`. Windows Clipboard History was disabled throughout testing, establishing that the problem concerned the normal clipboard contents rather than historical clipboard storage.

The security impact is an unintended extension of the exposure period for sensitive information. A clipboard timeout configured for a few seconds can effectively be bypassed, leaving a previously copied password or other secret accessible for a substantially longer period.

The developer investigated the report, explained the underlying cause, and implemented changes to retry failed clipboard-clearing operations and use an absolute expiration time. The researcher tested the supplied development build and confirmed that the reported behaviour was fixed. The developer stated that the improvements would be included in KeePass 2.62 and KeePass 1.44.

## 2. Affected Versions

| Product | Affected version | Fixed version announced by the developer |
|---|---|---|
| KeePass 2.x | 2.61.1 | 2.62 |
| KeePass 1.x | 1.43 | 1.44 |

The initial vulnerability was reproduced in KeePass 2.61.1. The developer subsequently confirmed that KeePass 1.43 was affected as well and implemented the same remediation in the 1.x edition.

The developer's stated release target for both fixed versions was the first week of November 2026, subject to unforeseen issues. This was a planned release window, not a confirmed publication date.

## 3. Technical Description

KeePass provides a configurable option to clear copied entry data from the clipboard after a specified interval. This feature is intended to reduce the time during which sensitive information remains accessible outside the encrypted password database.

In the affected implementation, the clipboard-clear countdown was handled by `MainForm.OnTimerMainTick`. When the countdown reached zero, KeePass called:

```csharp
ClipboardUtil.ClearIfOwner();
```

The developer explained that Windows blocks access to the clipboard while the Windows session is locked. Consequently, an automatic clearing attempt can fail when the configured timeout expires during that period.

In the affected versions, the failed operation was not reliably retried. The clipboard could therefore retain the copied value after the timeout had expired, with no subsequent automatic cleanup until another event changed the situation.

This was particularly problematic when KeePass was configured to lock its workspace when Windows was locked. Although the database was locked as intended, the copied secret remained available outside the database through the Windows clipboard.

The developer implemented two changes:

1. **Retry failed clipboard clearing.** KeePass retries the clearing operation once per second after a failure, continuing until it succeeds.
2. **Use an absolute expiration time.** Instead of relying exclusively on a decrementing timer counter, KeePass records an expiration deadline and checks the current time against it.

The retry mechanism addresses temporary failures to access the clipboard, while the absolute expiration time makes enforcement of the configured timeout more reliable.

## 4. Steps to Reproduce

### Test environment

- Operating system: Windows 11
- KeePass version: 2.61.1
- Configuration: default single-instance KeePass configuration
- Automatic clipboard clearing: enabled
- Clipboard timeout: 10 seconds
- Windows Clipboard History: disabled

### Procedure

1. Open a KeePass database containing a test entry.
2. Enable the option to lock the KeePass workspace when Windows is locked.
3. Configure the clipboard-clear timeout to 10 seconds.
4. Copy a password from a normal KeePass entry using the standard copy operation.
5. Immediately lock Windows using `Win+L`, before the clipboard timeout expires.
6. Leave Windows locked for substantially longer than the configured timeout.
7. Unlock Windows.
8. Confirm that the KeePass workspace is locked as expected.
9. Attempt to paste the copied password into another application using `Ctrl+V`.
10. Alternatively, run the following command in PowerShell:

```powershell
Get-Clipboard
```

### Expected behaviour

The copied value should be cleared when the configured timeout expires. If Windows temporarily prevents KeePass from accessing the clipboard, KeePass should clear the value when clipboard access becomes available again.

### Actual behaviour

The copied value remains retrievable after Windows is unlocked, despite the configured timeout having expired.

Access to the KeePass database is not required to retrieve the stale clipboard value.

## 5. Verification and Additional Testing

The researcher conducted multiple tests to establish that the issue was reproducible and specifically associated with the clipboard timeout expiring while Windows was locked.

| Scenario | Result |
|---|---|
| Windows remains locked beyond the clipboard timeout | Copied value remains available |
| Windows is locked with approximately one second remaining | Copied value remains available after the timeout |
| Windows is unlocked before the timeout expires | Normal countdown behaviour resumes and the value is subsequently cleared |
| An additional minute passes after unlocking | Stale value remains available |
| Windows is locked and unlocked repeatedly | Stale value persists |
| Clipboard timeout is changed after the value becomes stale | Existing value remains available |
| Source database is closed | Stale value persists |
| All open databases are closed | Stale value persists |
| Another database is opened and closed in the same KeePass process | Stale value persists |
| KeePass process exits | Value is cleared |
| Windows Clipboard History is disabled | Issue remains reproducible |
| Clipboard is queried using `Get-Clipboard` | Value is returned |
| Clipboard is accessed using normal paste | Value can be pasted |

The behaviour was also tested with the KeePass option to lock the workspace when Windows is locked both enabled and disabled.

The results demonstrate that the stale clipboard value is not dependent on the continued availability of the database from which it originated.

The researcher also tested KeePassXC under a comparable Windows lock and clipboard-timeout scenario. The same behaviour was not reproduced in KeePassXC.

Following the developer's remediation, the researcher tested the supplied KeePass 2.x development build and confirmed that the clipboard was cleared after Windows was unlocked when the configured timeout had expired during the locked session.

## 6. Security Impact and Attacker Scenarios

The vulnerability undermines the security purpose of KeePass's configurable clipboard timeout by allowing sensitive information to remain available beyond the period selected by the user.

Affected data may include:

- Passwords
- Usernames
- URLs
- Other sensitive entry fields

### 6.1 Credential exposure after workstation unlock

A user copies a password, locks their workstation, and leaves it unattended. The configured clipboard timeout expires while Windows is locked, but the clipboard value is not cleared.

When the workstation is subsequently unlocked, a person or process with access to the user's session may retrieve the previously copied password without opening KeePass or unlocking its database.

This creates an opportunity for credential theft after the user reasonably expects the clipboard timeout to have removed the secret.

### 6.2 Malware operating in the user's session

Malware or another untrusted process with sufficient access to the user's Windows session may retrieve clipboard contents.

The vulnerability increases the period during which a copied secret may be available to such software. Instead of being limited to the configured timeout, the secret can remain accessible long after the workstation was locked.

For example, malware that accesses the clipboard after the user returns to their workstation may recover a credential copied before the previous lock, even though the user expected it to have been cleared.

This does not mean that the vulnerability independently installs malware, grants elevated privileges, or provides access to a remote attacker. It extends the opportunity to obtain sensitive data for an attacker or process that can already access the clipboard.

### 6.3 Exposure despite KeePass workspace locking

The issue is especially relevant to users who enable automatic KeePass workspace locking when Windows is locked.

The user may reasonably expect the combination of workspace locking and a short clipboard timeout to reduce the risk of credential exposure when leaving the workstation.

However, the KeePass database can be locked while a previously copied secret remains available in the native Windows clipboard. The clipboard is outside the encrypted database and does not require the database to be reopened to retrieve the copied value.

### 6.4 Extended exposure beyond the configured timeout

A user may configure a short clipboard timeout specifically to minimise the exposure of copied secrets.

In the reported scenario, a timeout of 10 seconds can effectively become a period measured in minutes or hours, depending on how long the KeePass process remains running and whether the clipboard is overwritten or successfully cleared.

This represents a failure to enforce the intended lifetime of sensitive clipboard data.

### Security boundary and limitations

The vulnerability does not constitute a direct bypass of KeePass database encryption, master-password authentication, or Windows authentication. An attacker must still have an opportunity to access the clipboard.

The security concern is that the configured clipboard-clearing control does not reliably enforce its intended time limit, leaving sensitive information exposed longer than expected.

## 7. Root Cause and Remediation

The initial source review identified the countdown logic in `MainForm.OnTimerMainTick` and the call to `ClipboardUtil.ClearIfOwner()` when the countdown reached zero.

The initial report suggested using an absolute expiration time so that clipboard clearing would depend on elapsed time rather than exclusively on a decrementing timer counter.

The developer clarified that the main window timer continued to operate while Windows was locked, but Windows blocked access to the clipboard. The failure occurred when KeePass attempted to clear the clipboard without being able to access it.

The developer implemented both the proposed absolute-expiry-time approach and a retry mechanism for failed clearing operations.

The researcher verified that the supplied KeePass 2.x development build corrected the reported issue.

The developer stated that the improvements would be included in KeePass 2.62 and KeePass 1.44, covering the affected 2.x and 1.x branches respectively.

## 8. Responsible Disclosure Timeline

### 14 September 2026 — Initial disclosure

The researcher reported the clipboard auto-clear failure to Dominik Reichl, the KeePass developer.

The report included reproduction steps, timing tests, source-level observations, security impact, and a suggested fix based on an absolute clipboard-expiration time.

### Following the initial report — Vendor investigation and remediation

The developer explained that Windows blocks clipboard access while the session is locked.

The developer implemented retry logic for failed clipboard-clearing operations and introduced an absolute expiration time for automatic clipboard clearing.

### Subsequent testing — Fix verified in KeePass 2.x development build

The researcher tested the development build supplied by the developer and confirmed that the clipboard was cleared after Windows was unlocked when the configured timeout had expired while the session was locked.

### Subsequent vendor confirmation — KeePass 1.x also affected

The developer confirmed that the same clipboard-clearing problem affected KeePass 1.x and that the corresponding improvements had been implemented in that edition.

The developer identified KeePass 1.43 as affected and KeePass 1.44 as the intended fixed version.

### Planned release — KeePass 2.62 and KeePass 1.44

The developer stated that both fixed versions were planned for release within the first week of November 2026, subject to unforeseen issues.

The final release status should be verified before publication.

## 9. Conclusion

This vulnerability allows KeePass clipboard data to remain accessible beyond the configured automatic-clear timeout when the timeout expires while Windows is locked.

The issue was reproducible in KeePass 2.61.1 and was subsequently confirmed by the developer to affect KeePass 1.43 as well.

The impact is an extended exposure window for credentials and other sensitive information outside the encrypted KeePass database. The problem is particularly relevant when users rely on short clipboard timeouts and automatic workspace locking as complementary security controls.

The developer implemented a fix consisting of retrying failed clipboard-clearing operations and checking an absolute expiration time. The researcher confirmed that the reported behaviour was corrected in the supplied KeePass 2.x development build.

The developer stated that the fix would be included in KeePass 2.62 and KeePass 1.44.

The report is submitted for vulnerability tracking and assessment based on the reproducible behaviour, the confidentiality impact, the affected versions, and the vendor's remediation.
