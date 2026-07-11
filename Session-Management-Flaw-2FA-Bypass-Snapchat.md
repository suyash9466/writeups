# Session Management Flaw: Pre-Existing Sessions Not Invalidated After TOTP 2FA Enrollment (Snapchat)

##Overview

Two-Factor Authentication (2FA) is meant to add a second layer of protection, especially as a remediation step when an account may already be compromised. For 2FA to be effective as a remediation measure, enabling it should force re-authentication of any existing sessions.
In this case, a session management flaw was identified where enabling TOTP-based 2FA (via Google Authenticator) on Snapchat did not invalidate or challenge pre-existing authenticated sessions, allowing those sessions to continue performing sensitive account actions unchallenged.
Vulnerability Description
The vulnerability exists because enabling 2FA on one device does not propagate a session invalidation or re-authentication requirement to other already-authenticated sessions.
Key Issues:

* Existing sessions are not invalidated after 2FA enrollment
* Existing sessions are not challenged to re-authenticate using the new factor
* Sensitive account actions (password change, email change) can be performed from the old session without any 2FA prompt
* Username change requests the current password, but not 2FA verification Authentication Flow Analysis

##Expected Secure Flow:

1. User enables TOTP-based 2FA on any device
2. Server invalidates or flags all other active sessions for re-authentication
3. Any existing session attempting sensitive actions is challenged with the new 2FA factor
4. Only re-verified sessions can proceed with sensitive account changes

##Vulnerable Flow:

1. Account is logged in on Device A and Device B
2. TOTP-based 2FA is fully enrolled on Device B
3. Device A's pre-existing session remains active and untouched
4. Device A accesses account security settings without any 2FA challenge
5. Password and email are changed successfully from Device A with no re-authentication
6. Username change on Device A asks only for the current password — no 2FA step

##Proof of Concept (PoC)
A video PoC was recorded and submitted with the report, demonstrating that Device A's session remained fully authenticated and was able to change the account password and email address after 2FA was enrolled on Device B.

Steps Performed:

1. Logged in to the same account on two separate devices (Device A and Device B)
2. Kept Device A's session active and idle
3. Enabled TOTP-based 2FA via Google Authenticator on Device B
4. Completed 2FA setup fully on Device B
5. Returned to Device A and accessed account security settings
6. Changed account password from Device A — no 2FA challenge presented
7. Changed account email from Device A — no 2FA challenge presented
8. Attempted username change from Device A — prompted for password only, no 2FA

Technical Insight
The server does not treat 2FA enrollment as a session-level trust boundary change. It fails to re-evaluate the trust level of already-authenticated sessions once a new authentication factor is added, and instead continues to honor them at their original (pre-2FA) trust level indefinitely.
This means the protection 2FA is meant to provide only applies going forward to new logins, not retroactively to sessions that were established before 2FA existed on the account.
Impact

* If a session has been compromised via session hijacking, cookie theft, or physical device access, enabling 2FA from another device does not revoke attacker access
* An attacker with a valid pre-existing session can change the account password and email, effectively locking out the legitimate owner
* Undermines the core purpose of 2FA as a remediation step for a suspected account compromise
* Username changes have partial protection (password re-entry) but still lack 2FA verification

Severity
Medium

Reason: No authentication bypass is required to exploit this — it relies entirely on an attacker already holding a valid session. However, the ability to fully take over account recovery channels (password + email) from an unchallenged session, specifically at the moment 2FA is meant to lock them out, makes this more than a low-impact finding.

Real-World Risk
2FA is frequently recommended as the first response to suspected account compromise. If enabling it does not invalidate existing attacker sessions, users following standard security advice may believe their account is now protected when it is not. An attacker already inside the account can preemptively lock out the real owner by changing the password and email before the owner regains control.

Mitigation / Fix
To prevent such issues:

* Invalidate all other active sessions immediately upon successful 2FA enrollment
* Alternatively, force re-authentication (including the new 2FA factor) for any existing session before allowing sensitive actions
* Require 2FA verification — not just password re-entry — for username changes
* Require 2FA verification for password and email changes universally, regardless of session age
* Notify the user (via email/push) whenever a sensitive account change occurs, with an option to immediately revoke sessions
* Log and monitor sensitive account changes made shortly after 2FA enrollment

Tools Used

* Manual testing across two physical devices
* Google Authenticator (TOTP)
* Screen recording for PoC

Security Insight
Enabling a new authentication factor should be treated as a trust boundary event, not just an addition to future logins. Failing to re-validate existing sessions against a newly added factor can silently defeat the very protection that 2FA is meant to provide during incident response.

References

* OWASP Session Management Cheat Sheet
* OWASP Top 10 – Broken Authentication
* HackerOne Report #3787271

Disclosure Note
This vulnerability was identified through authorized testing on my own Snapchat account and reported responsibly via HackerOne. All testing was performed within the bounds of Snapchat's bug bounty program scope. No other users' data or accounts were accessed or affected.
