# Skills Log — CV-Ready Record

A running record of concrete, demonstrable skills earned through the IT Homelab project. Each line is framed as an achievement, not just a topic studied — organized by competency area for anyone (recruiter or technical interviewer) who wants to see exactly what was done and why it matters.

**Target roles:** Entry-level IT Support / Helpdesk, with SOC/cybersecurity and Cloud as longer-term directions.

---

## Active Directory & Windows Server administration

- Deployed and administered a Windows Server 2025 Active Directory domain (`corp.homelab.test`) from scratch, including AD DS, DNS, and Group Policy roles on a dedicated domain controller.
- Designed and implemented a department-based Organizational Unit (OU) structure (Kitchen, Bar, Service, IT, Management) reflecting a realistic small-business hierarchy.
- Created and managed security groups (vs. individual user permissions) to control access at scale.
- Diagnosed and resolved broken computer/domain trust relationships (secure channel failures) on domain-joined clients, using `Test-ComputerSecureChannel` and `nltest /sc_verify`.
- Diagnosed and recovered from an unplanned Active Directory object loss incident, including resolving an orphaned-SID permission issue after object recreation.
- Performed common account-administration tasks: unlocking a locked-out domain account, resetting a forgotten password, and **enabling a disabled account** — including understanding these as three distinct admin operations with different causes and different fixes.
- Diagnosed and fixed a **missing security group membership** as the root cause of a user's access loss, distinguishing it from an account-state problem despite an identical surface-level symptom.

## File services & access control (NTFS / SMB)

- Built and secured five department-level network shares using SMB share permissions layered with NTFS permissions.
- Applied the principle of least privilege in practice — granting Modify (not Full Control) to staff groups, and identifying/removing over-permissive inherited access.
- Diagnosed and fixed an NTFS permission-inheritance vulnerability that allowed unauthorized cross-department access, then verified the fix with authorized/unauthorized access testing.
- Designed and implemented a differentiated, layered access-control model: cross-department read-only access (IT_Staff) alongside department-exclusive write access, and a fully walled-off confidential share (Management) excluded even from IT's broad access.
- **Diagnosed and resolved a department-wide access outage** caused by an entire security group being removed from a share's NTFS permissions — recognized that two different access paths (direct UNC path and a mapped drive) point at the same underlying resource, and went straight to that resource instead of checking individual accounts.
- **Correctly distinguished live NTFS permission evaluation from cached authentication state**: predicted, before testing, that a permissions fix would take effect immediately with no logoff/logon required — unlike a group-membership fix, which needs a fresh Kerberos token.

## Group Policy

- Created and linked Group Policy Objects (GPOs) applying User Configuration restrictions (Control Panel access) scoped to a specific department OU.
- Deployed automatic network drive mapping across five departments using Group Policy Preferences (Drive Maps).
- Configured a domain-wide Password Policy and Account Lockout Policy (minimum length, complexity, history, lockout threshold/duration) via a GPO linked at the domain root — including understanding why domain account policy specifically requires domain-level (not OU-level) linking.
- **Diagnosed and resolved a real Group Policy precedence conflict**: identified that a competing setting on the built-in Default Domain Policy (higher link-order precedence) was silently overriding a newly created policy, and fixed it by reordering GPO links rather than editing the built-in policy — demonstrating understanding of per-setting GPO conflict resolution, not just GPO scoping.
- Deployed a company-wide Workstation Security Baseline GPO (screen lock + removable storage restriction) using GPO inheritance from a parent OU, rather than relinking per department.
- **Diagnosed a second, subtler Group Policy issue**: a setting confirmed "Applied" via RSoP still wasn't visibly taking effect in a live session; identified that a background policy refresh doesn't guarantee an already-running session re-reads certain values, and confirmed the fix (fresh logon) by reproducing the working behavior on two separate machines/users — demonstrating the ability to distinguish policy delivery from live effect, and to validate a hypothesis with a controlled retest rather than assuming.
- Practiced verifying Computer Configuration policy delivery via `gpresult /r`'s Computer Settings section when a live behavioral test wasn't possible (no test hardware available) — and explicitly documenting that limitation rather than overstating what was verified.
- Verified GPO application and troubleshot policy scope using `gpupdate /force`, `gpresult /r`, and `rsop.msc`.
- Practiced layered GPO troubleshooting: distinguishing "did the policy apply" (GPO scope/precedence) from "is the underlying resource/setting actually in effect" (direct testing), and distinguishing genuine AD policy enforcement from unrelated local Windows behavior (e.g. Windows' own sign-in throttling, which can look like a lockout but isn't one).
- **Diagnosed a "GPO not applying" fault involving both Security Filtering and the GPO Status toggle at once**, using `gpresult /r`'s Applied vs. filtered-out lists to prove a GPO was reaching no one at all (rather than just being denied to one group) — and recognized, after an initial misleading test, that a Group Policy *Preference* doesn't retroactively undo a setting already written to a user's profile, so validating a GPO fix requires testing against a target with no leftover cached result.
- **Diagnosed a real (non-staged) misconfiguration where a Group Policy Preference item had been built in the wrong GPO** — a management console showing an empty preference view while the setting kept working live, even through a full environment shutdown/restart. Correctly reasoned that Group Policy delivers a setting based on scope, not on which specific GPO object holds it, and traced the actual source to a second GPO linked to the same OU.
- **Applied the correct technique to actively remove an already-delivered Group Policy Preference from every affected client**, distinguishing it from simply deleting the source item (which only stops future delivery): built a temporary preference item with Action = Delete to force-clean clients regardless of their individual history, then rebuilt the setting correctly with Action = Create in the right GPO.

## IT Support ticket-based troubleshooting (Phase 3 — complete)

- Independently diagnosed five deliberately-staged, live-style helpdesk tickets end to end — symptom report → scope confirmation → hypothesis → investigation → root cause → fix → verification — without being told the underlying cause in advance, covering account state, group/token caching, live resource permissions, network/name resolution, and GPO scope/delivery. A sixth fault category, domain trust failure, was already demonstrated by a real (unplanned) incident earlier in the project rather than re-staged for its own sake.
- **Correctly scoped issues before touching any tool in every case**, including proactively determining and reporting scope (one user vs. a whole department) before being asked, on the third ticket — a habit formed directly from earlier coaching.
- **Self-corrected a flawed scoping test in real time**: initially compared against an account with a different, unrelated permission path, recognized why that comparison was invalid, and re-ran the test with a properly comparable account — a genuine troubleshooting-maturity signal, not just following steps.
- **Reasoned from underlying mechanism rather than symptom text**: recognized that a mapped drive and a direct UNC path are the same resource accessed two ways, and used that to go straight to the shared resource (an NTFS ACL) for a department-wide outage instead of checking individual accounts one by one.
- **Read Active Directory's own UI state indicators** to diagnose account and membership problems: recognized the disabled-account icon and flipped Enable/Disable context-menu action in ADUC, and used the Member Of tab and an NTFS ACL's entry list to compare working vs. broken state.
- **Caught a misleading initial symptom report**: a ticket's own description ("user can log in") turned out to be based on a stale, already-open session rather than the account's real current state — identified and confirmed this with a controlled fresh-logout/logon retest instead of taking the reported symptom at face value.
- **Recognized that two tickets sharing an identical surface symptom can have completely different root causes** (a disabled account vs. a missing security group membership) — avoided pattern-matching to a remembered fix and re-diagnosed each ticket from scratch.
- **Correctly predicted, before testing, whether a fix would require a fresh logon** — distinguishing cached, logon-time state (Kerberos tokens/group membership) from live, per-request state (NTFS permission checks, DNS resolution) — and used that reasoning to set the right expectation before verifying, rather than restarting out of habit.
- **Diagnosed a pre-authentication network/name-resolution failure as a distinct category from a permissions failure**: recognized from the error text alone ("cannot contact a domain controller") that the failure was happening *before* the point where identity or permissions would be checked, and correctly went to `ipconfig /all` rather than to ADUC or an NTFS ACL.
- **Identified a misconfigured DNS client as the root cause of a domain-controller-location failure**, found a client pointed at a public resolver instead of the internal domain DNS server, and applied a best-practice fix (pointing only at the internal DNS server, rather than leaving a public resolver as a secondary) instead of just making the symptom go away.
- **Read a PowerShell error message precisely to isolate a typo** (a stray space splitting a cmdlet name into two tokens) rather than assuming the underlying command or approach was wrong.
- **Recovered from a misleading first test result rather than accepting it at face value**: an initial test of a broken GPO appeared to show no effect (a mapped drive kept working), and rather than concluding the fix attempt had failed, re-examined the test method itself, identified the flaw (leftover cached state from before the change), redesigned a cleaner test, and reached the correct conclusion.
- **Used `gpresult /r`'s Applied vs. filtered-out GPO lists as a precise diagnostic tool**, correctly interpreting a GPO's *absence from both lists* as meaningful (no one has permission to have it applied at all) rather than assuming a GPO not visibly applying must simply be "denied."
- **Ruled out a plausible alternative cause before committing to a conclusion**: checked whether a user's AD "Home Folder" attribute (a different mechanism from Group Policy Preferences entirely) could be the real source of a persistent drive mapping, rather than assuming the GPO was automatically the only possible cause.
- Distinguished disabled vs. locked account states.

## Real-world incident diagnosed independently (post-Phase 3)

- **Diagnosed and resolved a genuine environment misconfiguration discovered by accident, not staged** — a Group Policy Preference item that had silently been built in the wrong GPO for an unknown period of time, hidden by the fact that it still worked (delivered by a second GPO in the same scope).
- **Escalated hypothesis testing methodically when simple explanations failed**: ruled out a stale console view (refresh), then a stale session/cache (full shutdown and restart of the entire environment), before reusing an existing diagnostic technique (the TICKET-05 clean-delete-and-relogon test) to determine whether the setting was genuinely still being delivered.
- **Trusted a live functional test over a management console's display** when the two disagreed, rather than assuming the tool must be right — then used that same functional test, run twice, to pin down which of two similarly-scoped GPOs was the real source of a setting.
- **Understood and correctly applied the distinction between stopping a Group Policy Preference and actively reversing its effect**: recognized that deleting a Preference item only prevents future delivery, and that machines which already received it would keep it indefinitely via Windows' own persistent-connection behavior, independent of Group Policy from that point on.
- **Selected and applied the correct real-world technique for forcing removal of an already-delivered setting across multiple machines**: built a temporary Action = Delete preference item — the standard scalable method for retiring or scrubbing a Preference-based setting company-wide — rather than manually cleaning each affected machine by hand.

## Networking & DNS

- Configured and validated domain-integrated DNS as the foundation for AD service discovery, including querying SRV records (`_ldap._tcp.dc._msdcs`) to confirm domain controller locatability.
- Used `ipconfig`, `ping`, and `nslookup` systematically to isolate and confirm DNS configuration issues on domain clients.
- **Diagnosed and fixed a live DNS client misconfiguration** on a domain-joined workstation (client pointed at a public resolver instead of the internal domain controller), using `Set-DnsClientServerAddress`, and understood why internet name resolution on a domain-joined client should come from a forwarder on the internal DNS server rather than a second DNS entry on the client.

## Troubleshooting methodology

- Practiced structured, symptom-first IT troubleshooting (symptom → likely layer → test → interpret result → next step) across multiple real (unplanned) incidents and deliberately constructed tickets diagnosed without foreknowledge of the cause.
- Distinguished session-level artifacts (cached credentials, stale Kerberos tokens, client-side UI messages, stale login sessions) from actual configuration/policy/account-state/permission failures — a recurring, high-value diagnostic skill that showed up in multiple, unrelated incidents.
- Verified findings against authoritative sources (e.g. checking AD/ADUC and NTFS ACLs directly) rather than trusting ambiguous client-side symptoms or the initial problem report as given.
- Formed an initial hypothesis, tested it directly rather than accepting it as the final answer, found the real root cause, and confirmed it by retesting — a disciplined close-the-loop habit rather than stopping at "it works now."
- Chose a genuinely comparable test case when scoping an issue, rather than any convenient alternative — and corrected course when an initial comparison turned out to be invalid.
- Used scope (one user vs. many) to decide where to look first — an individual-account investigation vs. a shared-resource investigation — rather than starting from the same checklist every time.
- Categorized a failure as pre-authentication (network/name resolution) vs. post-authentication (permissions) from the error text alone, before running any diagnostic command — narrowing the investigation immediately instead of checking every layer in sequence.
- Treated an unexpected or "clean" test result as a signal to question the test itself, not just the hypothesis — recognizing that a flawed test setup (leftover cached state) can produce a false negative that looks identical to "the fix didn't work."
- Systematically eliminated explanations in order of simplicity (console cache → session/OS cache → configuration reality) rather than jumping straight to the most complex explanation, and trusted a live functional result over a tool's own display when the two conflicted.

## Documentation & technical writing

- Produced a complete, interview-ready project documentation set covering architecture, AD/OU/GPO/permissions structure, troubleshooting case studies, and lessons learned — written for distinct audiences and purposes rather than one undifferentiated document: a narrative build/troubleshooting log, a CV-ready skills record, an external-facing GitHub README, and an exhaustive technical reference.
- Built Mermaid diagrams (network architecture, AD/OU structure) directly into Markdown documentation rather than as separate static images, keeping the diagrams version-controlled alongside the text they explain.
- **Verified technical documentation against the live environment rather than relying on memory or inference** — went back into GPMC to confirm exact Password Policy and Account Lockout Policy values (history count, max/min age, lockout duration, reset counter) instead of estimating or leaving a plausible-looking but unconfirmed number in a reference document.
- Organized reference material by actual use case rather than by convenience: a condensed external-facing summary (README) kept separate from an exhaustive internal reference (every GPO's settings, full permissions matrix) — recognizing that a recruiter and a technical interviewer asking for detail need different documents, not one document trying to serve both.

## Tools used hands-on

Active Directory Users and Computers (ADUC), Group Policy Management, Group Policy Management Editor, Windows Server 2025, PowerShell (`New-ADGroup`, `Add-ADGroupMember`, `Get-ADGroup`, `Get-ADUser`, `Test-ComputerSecureChannel`, `Set-DnsClientServerAddress`), `icacls`, `nltest`, `nslookup`, `ipconfig`, `net use`, `gpresult`, `rsop.msc`, VMware Workstation Pro.
