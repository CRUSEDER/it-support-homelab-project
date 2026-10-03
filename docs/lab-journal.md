# IT Homelab — Lab Journal

A running record of what's been built, tested, and learned in Alkali's Active Directory / IT Support homelab. Updated as work happens — read this at the start of a new session to pick up where things left off.

---

## Environment

- **Hypervisor:** VMware Workstation Pro, on a dedicated Windows laptop.
- **Domain Controller:** DC01, Windows Server 2025, IP `192.168.138.10`. Provides AD DS, DNS, Group Policy, file sharing.
- **Domain:** `corp.homelab.test`
- **Clients:** CLIENT01 (Kitchen), CLIENT02 (Bar), CLIENT03 (Service) — all Windows 11 Pro, domain-joined, secure channel healthy on all three.

## AD structure

```
Corp
├── Kitchen    (Steven – Head Chef, Lucas, Justina, Alex, Paul)
├── Bar        (Jonathan – Head, Arino, Joyce, Ahn)
├── Service    (Alia – Head, Kossi, Mirinda, Andrea)
├── IT         (Alkali)
└── Management (Mathias)
```

Department-based security groups are used throughout (`Kitchen_Staff`, `Bar_Staff`, `Service_Staff`, `IT_Staff`, `Management_Staff`) rather than assigning permissions to individual users — the point of this exercise is understanding *why* groups scale better than per-user permissions.

---

## Department status — Phase 1 (complete)

### Kitchen — ✅ Done (built before this journal started)
- `Kitchen_Staff` security group, `\\DC01\Kitchen` share, NTFS permissions.
- **Incident:** Joyce (not Kitchen staff) could initially access the Kitchen share due to NTFS inheritance carrying over a broad "Users" permission from the parent folder. Fixed by breaking inheritance and scoping access to `Kitchen_Staff` only.
- GPO: a Control Panel restriction applied to Kitchen users (via User Configuration), tested on Steven, verified with `gpupdate /force` and `gpresult /r`.

### Bar — ✅ Done and verified (built via GUI)
- `Bar_Staff` (Global, Security) created in `OU=Bar` via Active Directory Users and Computers. Members: Jonathan, Arino, Joyce, Ahn.
- `\\DC01\Bar` share created at `C:\Shares\Bar` via Explorer → Advanced Sharing. Share permission: Authenticated Users, Full Control (deliberately open — see "NTFS vs share permissions" below).
- NTFS: inherited "Users" permission (which every domain user gets by default) removed; inheritance broken and converted to explicit permissions; `Bar_Staff` granted **Modify**.
- **Tested:** Joyce (Bar_Staff) → access granted, could create/edit a file. Steven (Kitchen, not a member) → Access Denied.

### Service — ✅ Done and verified (built via PowerShell, then rebuilt via GUI after an incident)
- `Service_Staff` (Global, Security) created in `OU=Service` via PowerShell (`New-ADGroup`, `Add-ADGroupMember`). Members: Kossi, Alia, Mirinda, Andrea.
- `\\DC01\Service` share created at `C:\Shares\Service` via `New-SmbShare`.
- NTFS permissions applied via `icacls` (`/inheritance:r` then `/grant:r`) — same model as Bar: broad `Users` access removed, `Service_Staff` granted Modify.
- **Incident:** After DC01 was powered off/suspended and resumed the next day, `Service_Staff` had vanished from AD (`Get-ADGroup` returned "object not found"), while `Bar_Staff` (created earlier) and the `C:\Shares\Service` folder (created later) both survived. Cause not fully confirmed — most likely tied to how the VM's state was saved/suspended rather than a full snapshot revert. **Lesson: prefer a clean "Shut Down Guest OS" over VM suspend for DC01, and be deliberate about snapshots.**
- **Recovery lesson:** recreating a group with the *same name* is not enough — Windows permissions are bound to a group's **SID**, not its name. Fix: remove the orphaned SID entry and re-grant the *new* `Service_Staff` object explicitly.
- **Tested:** Kossi (Service_Staff) → access granted. Steven (Kitchen, not a member) → Access Denied.

### IT — ✅ Done and verified (built via GUI)
- `IT_Staff` (Global, Security) created in `OU=IT`. Member: Alkali.
- `\\DC01\IT` share created via Explorer → Advanced Sharing. NTFS on its own folder: `IT_Staff` granted **Modify**.
- **Layered cross-department access:** `IT_Staff` also added to the *existing* NTFS permissions of Kitchen, Bar, and Service — **Read & Execute only** — modeling real-world helpdesk visibility without expanding blast radius.
- **Tested:** Alkali (IT_Staff) → full access on IT; read-only (no write) on Kitchen. Steven (Kitchen, not IT_Staff) → denied on IT.

### Management — ✅ Done and verified (built via GUI)
- `Management_Staff` (Global, Security) created in `OU=Management`. Member: Mathias.
- `\\DC01\Management` share created. NTFS: `Management_Staff` granted **Modify** — deliberately no other group added, not even IT_Staff.
- **Tested:** Mathias (Management_Staff) → full access. Alkali (IT_Staff, not a member) → denied — confirming exclusivity held even for the group with the broadest access elsewhere.

**Phase 1 complete: all five departments have security groups, shares, and tested NTFS permissions.**

---

## Group Policy — Phase 2 (complete)

### Drive mapping — ✅ Done and verified (all five departments)
- One GPO per department (e.g. `Kitchen - Map Network Drive`), each linked to its own OU — kept modular by design.
- Configured via **User Configuration → Preferences → Windows Settings → Drive Maps**: Action = Create, Location = department's UNC path, a distinct drive letter per department, Reconnect checked.
- **Tested:** all five department drives confirmed appearing at logon for the correct users.
- **Concept:** Group Policy **Preferences** (e.g. Drive Maps) vs. **Policies** (e.g. the Control Panel restriction) — Preferences set a default the user could technically change; Policies are enforced. Preferences are the modern GUI replacement for logon scripts.

### Password Policy & Account Lockout Policy — ✅ Done, verified, and involved a real troubleshooting incident
- New GPO `Domain - Password and Lockout Policy`, linked at the **domain root** (not an OU) — deliberately, because domain account password/lockout policy is only honored by AD from a domain-linked GPO, never an OU-linked one (unlike everything else built so far).
- Configured under **Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies**: password history, min/max age, minimum length (10), complexity enabled; lockout threshold (5), lockout duration, reset counter.
- **Incident:** the policy silently did not apply — root cause was a GPO precedence conflict with the built-in `Default Domain Policy`, which already had Account Lockout Threshold explicitly set to 0 (disabled) and higher precedence (Link Order 1 vs. the new GPO's Link Order 2). Fixed by reordering the new GPO to Link Order 1, without editing Default Domain Policy's contents at all — since GPO conflicts resolve per-setting, giving the new GPO top precedence made every setting it explicitly defines win, for both password and lockout policy at once.
- **Tested end-to-end:** deliberately locked out a test account (Paul) with repeated wrong passwords → confirmed real lockout via ADUC's "Unlock account" indicator (not just an ambiguous Windows UI message) → unlocked the account → reset the password (weak password correctly rejected by policy, proving the Password Policy half also took effect) → set a compliant password with "must change at next logon" → confirmed full logon and forced change worked.

### Workstation Security Baseline — ✅ Done and verified (screen lock fully confirmed; removable storage confirmed as delivered)
- GPO `Corp - Workstation Security Baseline`, linked at the **`Corp` OU** (the parent of all department OUs) — uses GPO **inheritance down the OU tree**: linking once at a parent applies to every child OU beneath it, instead of repeating the GPO per department like drive mapping did. Chosen because "lock your screen" and "no USB drives" should apply company-wide.
- **Screen lock (User Configuration → Administrative Templates → Control Panel → Personalization):** Enable screen saver = Enabled; Screen saver timeout = 60 sec (test value; production value TBD, e.g. 600); Password protect the screen saver = Enabled; Force specific screen saver = `scrnsave.scr`.
- **Removable storage block (Computer Configuration → Administrative Templates → System → Removable Storage Access):** "All Removable Storage classes: Deny all access" = Enabled. Deliberately Computer Configuration, not User, since the restriction should apply to the machine regardless of who logs in.
- **Incident — screen lock not visually triggering (resolved):** `rsop.msc` had confirmed all four Personalization settings were genuinely applied to a live session, yet the screen never actually blanked/locked after the idle timeout. Initial hypothesis was a VM-specific idle-detection quirk. **Root cause found:** the test session had been running *before* the policy was fully applied — a background `gpupdate /force` updates the registry/RSoP data, but the already-running desktop session's screensaver watchdog doesn't necessarily re-read the new timeout live. A full logoff/logon (or restart) is what actually makes it take effect. **Confirmed twice:** first on CLIENT02 with a Bar user (`gpupdate /force` → restart → fresh logon → timed test → locked at exactly 60 seconds, resumed to a login prompt), then reproduced on CLIENT01 with Steven under the same fresh-logon procedure, ruling out both the VM-quirk theory and any CLIENT01/Kitchen-specific cause.
- **Removable storage — verified via delivery, not live behavior:** no physical USB device available to test the actual deny-access behavior. Verified instead via `gpresult /r` (run elevated) on CLIENT02, confirming `Corp - Workstation Security Baseline` appears under the **COMPUTER SETTINGS** section's Applied Group Policy Objects — proving the setting reached the machine correctly and is scoped as Computer Configuration (applies regardless of logged-in user), distinct from the User Configuration screen lock setting. **Note for later:** this confirms delivery only, not the live block — worth a real behavioral test if a spare USB drive becomes available.

### Still optional in Phase 2:
- Possibly software/settings deployment via GPO — likely to skip: legacy technique for pushing software (MSI assignment), largely superseded in modern shops by tools like Intune/SCCM. Worth knowing about conceptually for interviews, not necessarily worth building here.

**Phase 2 complete: drive mapping, domain-wide password/lockout policy, and the workstation security baseline (screen lock fully verified live; removable storage verified as delivered) are all done.**

---

## IT Support Troubleshooting Labs — Phase 3 (complete)

Format: Alkali picks one fault from a menu, sets it up himself *without telling the coach which one*, then reports only the symptom the way an end user would describe it. The cause isn't known going in — diagnostics have to be actually run and real output interpreted, rather than just recalling what was changed. Coaching comes as questions (symptom → scope → hypothesis → test → interpret → fix → verify), not answers.

### TICKET-01 — Paul can't access the Kitchen share or his mapped drive
- **Reported symptom:** Paul could apparently log in, but couldn't access `\\DC01\Kitchen` or his `K:` mapped drive. Initially unclear whether this affected all of Kitchen or just Paul.
- **Scoping step:** confirmed every other Kitchen user could access both the share and the K: drive normally — isolated to Paul alone, ruling out a share/GPO/DC-wide problem.
- **Investigation:** checked Paul's account in ADUC and found two tells: a **down-arrow on his user icon** (the standard visual marker for a disabled account) and the right-click context menu showing **"Enable Account"** instead of "Disable Account" (confirming the account's current state is disabled — the menu shows the action available, not the current state).
- **Root cause:** Paul's AD account was **disabled**. The initial "he can log in" report was misleading — it was based on an already-open session from before the account was disabled, not a live successful authentication. A forced logout + fresh login attempt correctly showed "Your account has been disabled."
- **Distinction reinforced:** *Locked* (too many bad password attempts, self-clears after the lockout duration) and *Disabled* (an admin turned the account off; stays off until manually re-enabled) are two separate account states with separate indicators and separate fixes — the "Unlock account" checkbox is unrelated to the Enable/Disable state.
- **Fix:** clicked "Enable Account" in ADUC.
- **Verified:** fresh login as Paul succeeded; both the Kitchen share and the K: mapped drive worked normally afterward — confirming the account state was the *only* thing wrong (Paul's group membership and the share/GPO config were never touched).

### TICKET-02 — Paul can't access the Kitchen share or K: drive (again — different cause)
- **Reported symptom:** Paul logged in fine this time (already in File Explorer), but got a Windows network error: "Windows cannot access \\DC01\Kitchen — You do not have permission to access \\DC01\Kitchen." K: drive also failing. On the surface, looked similar to TICKET-01, but the symptom shape was already different (an explicit access-denied error from the server, not a logon-time failure).
- **Scoping step — with a self-correction:** first tested access with the IT account, which worked — but this was flagged as the wrong comparison, since `IT_Staff` has its own separate, layered Read & Execute permission on the Kitchen share, unrelated to how `Kitchen_Staff` members get access. The valid test is another actual `Kitchen_Staff` member: tested with Steven, who accessed the share and K: drive normally — confirming the issue was isolated to Paul specifically, not the share/GPO.
- **Investigation:** checked Paul's **Member Of** tab in ADUC — he showed only `Domain Users` (the default every account gets), missing `Kitchen_Staff`. Cross-checked other Kitchen members' Member Of tabs, who all showed both `Domain Users` and `Kitchen_Staff`.
- **Root cause:** Paul had been removed from the `Kitchen_Staff` security group. Since both the share's NTFS permissions and the K: drive-mapping GPO are scoped to that group, losing membership broke both at once — explaining why the symptom looked identical to TICKET-01 despite having a completely different cause.
- **Fix:** re-added Paul to `Kitchen_Staff` in ADUC.
- **Verification reasoning applied correctly, without prompting:** anticipated that Paul's already-open session wouldn't reflect the new group membership immediately, because group memberships are baked into the Kerberos access token at logon time — a background refresh doesn't re-issue it. Had Paul restart (forcing a fresh logon and a new token) before retesting.
- **Verified:** after the restart, both the Kitchen share and the K: drive worked normally for Paul.

### TICKET-03 — Whole Kitchen department locked out of the share and K: drive
- **Reported symptom:** this was proactively scoped before being reported — the entire Kitchen department (not just one user) was denied on both `\\DC01\Kitchen` and the K: drive. A scope shift from the first two tickets, which were both isolated to Paul.
- **Reasoning step:** the K: drive is just a mapped shortcut to the same UNC path people reach directly — not a separate permission system. Since *both* paths failed for the *whole department* at once, the investigation went straight to the shared resource itself (the folder's NTFS permissions) rather than checking individual accounts.
- **Investigation:** checked the Kitchen folder's Security tab and found the `Kitchen_Staff` entry missing from the NTFS permission list entirely. Cross-checked the rest of the ACL and confirmed `IT_Staff` (the Phase 1 cross-department Read & Execute entry) was still intact — meaning this was a single, targeted entry removed, not a full ACL reset.
- **Root cause:** `Kitchen_Staff` had been removed from the Kitchen folder's NTFS permissions entirely.
- **Fix:** re-added `Kitchen_Staff` to the folder's NTFS permissions at **Modify** (matching the original Phase 1 configuration, not Full Control).
- **Concept correctly predicted before testing:** unlike TICKET-02 (a group-membership/Kerberos-token issue, which needed a fresh logon to take effect), an NTFS permission change is evaluated live by the file server on every access attempt — there's no cached token involved for file access, so no restart is needed. Correctly reasoned this through before testing rather than assuming "problems mean restart."
- **Verified:** tested with two different Kitchen members (Paul and Steven) — both regained access to the share and K: drive immediately, no restart required.

### TICKET-04 — Joyce can't access the Bar share (DNS misconfiguration on CLIENT02)
- **Reported symptom:** Joyce, trying to access the Bar share, was prompted to re-enter network credentials to connect to DC01, with the error "The system cannot contact a domain controller to service the authentication request." A different failure shape from any prior ticket — not a "permission denied" from the server, but a failure to even reach a domain controller in the first place.
- **Reasoning step:** correctly identified this as a network/name-resolution layer problem rather than a permissions problem, since the failure was happening *before* the point where identity or permissions would even be checked.
- **Investigation:** ran `ipconfig /all` on CLIENT02 and found the DNS Servers set to `8.8.8.8` (a public DNS resolver) and `192.168.138.2` (the default gateway) — neither of which is DC01 (`192.168.138.10`). Neither of those servers has any knowledge of `corp.homelab.test` or how to locate a domain controller via SRV records, which explained the exact symptom.
- **Root cause:** CLIENT02's DNS client settings had been pointed away from DC01.
- **Fix:** `Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses "192.168.138.10"` (after correcting a stray-space typo in the first attempt that broke the cmdlet name into two tokens — reading the "command not found" error correctly spotted the typo rather than assuming the command itself was wrong). Confirmed with `ipconfig /all` that only `192.168.138.10` was listed afterward.
- **Best-practice note applied:** deliberately did **not** leave a public DNS server as a secondary — a domain-joined client falling back to a resolver that knows nothing about the internal domain produces intermittent, confusing auth failures, worse than a clean consistent one. Any need for internet resolution belongs on a **forwarder** configured on DC01's own DNS server role, not a second DNS entry on the client.
- **Concept correctly predicted before testing:** DNS server settings are read fresh on every lookup, not cached in a logon token — correctly reasoned no restart would be needed, unlike the group-membership case in TICKET-02.
- **Verified:** Joyce accessed the Bar share and its files immediately after the DNS fix, no restart required.

### TICKET-05 — Steven's K: drive mapping stops delivering to new sessions (GPO not applying)
- **Setup (self-inflicted this time, for the "GPO not applying" scenario):** removed `Authenticated Users` from Security Filtering on the `Kitchen - Map Network Drive` GPO (leaving the filter list empty), and separately set the GPO's status to "User configuration settings disabled."
- **First test gave a misleading result:** K: was still present and working on Steven's machine after a `gpupdate /force` + restart. This looked like the GPO change had no effect — but the test was flawed, not the reasoning: K: had already been mapped into Steven's profile the day before, and nothing had actually removed it from his machine yet. A GPO change can't be judged by a session that already has the old result cached.
- **Diagnosis step:** ran `gpresult /r` as Steven and checked the **Applied Group Policy Objects** and **filtered out** lists under USER SETTINGS. `Kitchen - Map Network Drive` appeared in *neither* list — not applied, and not even listed as filtered out. This confirmed Group Policy itself no longer considers Steven eligible for that GPO at all: with an empty Security Filtering list, nobody has "Read" or "Apply Group Policy" permission on the GPO, so a client can't even enumerate it during processing (a GPO that's merely denied for one user/group still shows up in the "filtered out" list; a GPO nobody can read doesn't show up anywhere).
- **A wrong hypothesis eliminated along the way:** briefly considered whether the drive was coming from the user object's own **Home Folder** setting (ADUC → user Properties → Profile tab) rather than the GPO — checked Steven's Profile tab and found it set to "local path," ruling that out.
- **Proper test, run correctly the second time:** on Client01 (Steven's machine), ran `net use K: /delete` to fully remove the existing mapping, then logged Steven off and back on. K: did **not** come back — confirming the GPO really had stopped delivering the mapping to a session that didn't already have it.
- **Root cause:** two changes stacked on the `Kitchen - Map Network Drive` GPO — an emptied Security Filtering list (nobody has permission to have the GPO applied) and "User configuration settings disabled" (switches off the entire User Configuration half of the GPO regardless of filtering). Either alone would have broken the drive mapping for new sessions; both were set at once.
- **Fix:** re-added `Authenticated Users` back to Security Filtering, and set GPO Status back to "Enabled."
- **Verified:** logged Steven off and back on — K: reappeared automatically, with no manual recreation needed, confirming the GPO was delivering the mapping again on its own.
- **Key lesson from the false-negative first test:** a Group Policy **Preference** (like a Drive Map) is a one-time write into the user's profile, not an ongoing enforced state — so breaking the GPO that created it doesn't retroactively undo something already applied. To honestly test whether a GPO change took effect, the target has to be in a "clean" state with no leftover result from before the change, otherwise the test can pass or fail for the wrong reason.

### Domain trust failure — already demonstrated (not repeated as a Phase 3 ticket)
Rather than manufacture a sixth ticket, this category is considered already covered: the earlier real (unplanned) incident on CLIENT02/CLIENT03 — broken secure channels from cloned VMs, `Test-ComputerSecureChannel` returning False, `ERROR_NO_TRUST_SAM_ACCOUNT`, stale computer objects blocking rejoin, diagnosed with `nltest /sc_verify` and resolved by clearing the stale AD computer objects and rejoining — is a genuine, harder version of this exact fault category, done before this journal even started (see Environment section). No value in re-staging it artificially.

**Phase 3 complete: five deliberately-staged tickets (account state, group membership, NTFS permissions, DNS, GPO delivery) plus one real prior incident (domain trust) covering the full planned fault category list.**

---

## Post-Phase 3 — Real incident: Kitchen drive map misfiled in the wrong GPO

Discovered by accident days after Phase 3 closed, while just doing a routine check of the Kitchen drive mapping — not a staged exercise, an actual environment quirk.

- **Starting point:** opened `Kitchen - Map Network Drive` in the Group Policy Management Editor to review the Drive Maps preference item, and found **User Configuration → Preferences → Windows Settings → Drive Maps completely empty** — "There are no items to show in this view." Yet Steven and the rest of Kitchen could access K: normally at the same time.
- **First hypotheses tested and ruled out:**
  - *Stale console view* — refreshed the node (F5) and fully reopened the Group Policy Management Editor: still empty.
  - *Stale session/cache at the OS level* — shut down all VMs and the host laptop entirely, restarted everything: still empty. This ruled out any kind of caching at the console, session, or OS level.
  - *One-time-write leftover mapping (the TICKET-05 explanation)* — reused the TICKET-05 test directly: `net use K: /delete` + fresh logoff/logon on Steven's machine. **K: came back.** This proved the mapping is still being *actively delivered*, not just a leftover from before — ruling out "empty GPO, residual mapping" as the explanation and confirming the editor's "empty" view was accurate for *that* GPO, while something else was really doing the work.
- **Root cause found:** the Drive Maps preference item had actually been created inside a *different* GPO — `GPO - Kitchen User Restrictions` (the Control Panel-restriction GPO from Phase 1) — instead of `Kitchen - Map Network Drive`. Since both GPOs are linked to the Kitchen OU and both process User Configuration Preferences, Group Policy didn't care which GPO object held the item — it just delivered it. `Kitchen - Map Network Drive` was empty because it genuinely never held the setting; the documented "one GPO per purpose" design (see Phase 2) didn't match what was actually built.
- **Confirmation test — delete and observe:** deleted the misfiled item from `GPO - Kitchen User Restrictions` outright, then repeated the clean test (`net use K: /delete` + fresh logoff/logon) on Steven. This time K: did **not** come back — confirming that item really was the only thing delivering the Kitchen mapping, and that Kitchen now had no working drive map at all until it was rebuilt correctly.
- **Follow-on problem noticed:** deleting the misfiled item only stops *future* deliveries — it does nothing for machines that already had K: mapped and hadn't had a fresh logon since. Windows' own "reconnect at logon" behavior keeps a mapped drive coming back on its own once GPP has created it once, completely independent of Group Policy from that point on.
- **Real company-scale question raised and answered:** how do you actively remove a Preference-created setting from users who already have it, not just stop it for new sessions? Two mechanisms exist:
  1. **"Remove this item when it is no longer applied"** (Common tab on the preference item) — only works if it was enabled *before* the item stopped applying, since it relies on history already recorded on each client. Not available here since it wasn't set in advance.
  2. **A new preference item with Action = Delete** — an explicit "remove K: if present" instruction that doesn't depend on any client history, and will clean up any client regardless of how it got the mapping. This is the standard real-world technique for actively unwinding a Preference-delivered setting company-wide (relevant equally to retiring an obsolete setting or scrubbing an unwanted/unauthorized one).
- **Applied the Delete-action technique for real cleanup:** added a temporary Drive Maps item to `Kitchen - Map Network Drive` with **Action = Delete** (same path/letter). Had the Kitchen users who hadn't been retested since the earlier deletion (Lucas, Justina, Alex, Paul) each log on once — confirmed their lingering K: mappings were removed by the Delete action, proving the technique works regardless of how a client originally got the drive.
- **Rebuilt correctly:** removed the temporary Delete-action item, then created a proper Create-action Drive Map item in `Kitchen - Map Network Drive` (Action = Create, `\\DC01\Kitchen`, K:, Reconnect checked) — matching the other four departments' configuration and the project's documented design.
- **Verified:** fresh logon confirmed K: now maps correctly, delivered from the correct GPO this time.

**Key lesson:** a tool's display and the system's actual behavior can disagree — when they do, a live functional test is ground truth, not the UI. Also reinforces and extends the TICKET-05 lesson: Group Policy doesn't care which GPO object a setting lives in, only that some applicable GPO delivers it, which means a misconfiguration can hide for a long time behind an accidentally-working symptom.

---

## Key concepts learned so far

- **Security groups vs. individual permissions** — assign access once to a group, manage membership instead of touching permissions per person.
- **Share permissions vs. NTFS permissions** — Windows enforces the *more restrictive* of the two.
- **Permission inheritance** — new folders inherit their parent's permissions by default.
- **Principle of least privilege** — staff get Modify, not Full Control.
- **Layered/differentiated permissions** — one folder's ACL can hold multiple groups at different levels (e.g. Kitchen_Staff at Modify, IT_Staff at Read & Execute).
- **Blast radius** — why cross-department access should be Read & Execute, not Modify.
- **SIDs vs. names** — permissions bind to a SID, not a name; recreated objects are different objects.
- **Kerberos tokens / secure channel** — group memberships are baked into a token at logon; a domain-looking login doesn't guarantee a healthy trust relationship, and a background refresh doesn't retroactively add a new group to an already-issued token — a fresh logon is required.
- **NTFS/share permission checks are live, not cached** — unlike group membership (baked into the Kerberos token at logon), the file server evaluates the current ACL on every single access attempt. A permission fix takes effect immediately, with no logoff/logon needed.
- **DNS lookups are also live, not cached in a token** — like NTFS checks, a DNS client setting is read fresh on every lookup. Changing which DNS server a client uses takes effect immediately for new queries.
- **A mapped drive letter is not a separate access path** — K: is just a shortcut to the same UNC path reached directly; both go through the exact same underlying share and NTFS permissions, so a resource-level permission problem breaks both simultaneously.
- **Failures before vs. after identity is checked** — a "cannot contact a domain controller" error happens *before* authentication/permissions are ever evaluated (a network/name-resolution layer problem), which is a fundamentally different category from an access-denied error that happens *after* the server knows who you are. Reading which category an error belongs to narrows the investigation immediately.
- **A DNS client can have working general connectivity and still be unable to reach AD** — DHCP, IP addressing, and general internet DNS can all work fine while the client is still unable to locate a domain controller, because locating AD services specifically depends on querying the *right* DNS server for SRV records.
- **A mixed public/internal DNS configuration is worse than a clean failure** — leaving a public resolver as a secondary DNS server on a domain-joined client can cause intermittent, hard-to-diagnose auth failures if the client ever falls back to it, since public DNS has no knowledge of internal domain records. Internet resolution for clients should come from a forwarder configured on the internal DNS server, not a second client-side DNS entry.
- **Group Policy Preferences vs. Policies** — Preferences (e.g. Drive Maps) set a default; Policies are enforced.
- **A Preference is a one-time write, not an ongoing enforced state** — confirmed directly in TICKET-05: breaking the GPO that created a Drive Map didn't remove the mapping already sitting in a user's profile from before the change. Testing whether a GPO fix or break actually took effect requires a target with no leftover result already cached from before — otherwise a test can look like it passed (or failed) for the wrong reason.
- **Empty Security Filtering vs. a denied entry** — a GPO with *nobody* granted "Apply Group Policy" permission doesn't show up in `gpresult /r`'s "filtered out" list at all (a client can't even enumerate a GPO it has no Read permission on); a GPO that's explicitly denied to one user/group *does* show up there. The absence from both lists is itself diagnostic information.
- **A setting delivered by Group Policy is not tied to a specific GPO object** — Group Policy only cares whether *some* applicable, linked GPO delivers a given setting, not which one. A preference item built in the wrong GPO can work perfectly (and hide a real documentation/design error) as long as some other GPO in scope happens to process it too.
- **Deleting a Preference item only stops future delivery, not existing effect** — because a Drive Map (like any GPP-created setting) hands off to Windows' own "reconnect at logon" behavior once created, removing the source GPO item does nothing for machines that already have it. Actively removing an already-applied Preference from every affected client requires either "Remove this item when it is no longer applied" (set in advance, relies on client history) or a fresh preference item with **Action = Delete** (works regardless of history — the standard technique for company-wide cleanup of a Preference-delivered setting).
- **Trust live behavior over a tool's display when they disagree** — a GPMC editor window, an RSoP report, or any other management UI is a reading of the system, not the system itself; when it conflicts with an actual functional test, the functional test is ground truth, and the disagreement itself becomes the next thing to investigate.
- **Domain-linked vs. OU-linked GPOs** — password/account lockout policy for domain accounts is only honored from a GPO linked at the **domain root**; OU-linked GPOs are silently ignored for this specific setting category, unlike almost everything else in Group Policy.
- **GPO precedence / link order** — when multiple GPOs linked to the same container configure the same setting, the GPO with the **lowest link order number** wins (applied last). Conflicts resolve **per individual setting**, not per whole GPO — so a GPO with top precedence wins only for the specific settings it explicitly defines.
- **GPO inheritance down the OU tree** — a GPO linked at a parent OU (e.g. `Corp`) applies to every child OU beneath it (Kitchen, Bar, Service, IT, Management) automatically, unless something blocks it — the efficient alternative to relinking the same GPO repeatedly.
- **Windows sign-in throttling vs. AD account lockout** — Windows 11 adds its own delay between failed local sign-in attempts, independent of any AD policy. An ambiguous client-side message doesn't prove AD's lockout actually fired — the authoritative check is the account's status in ADUC (or `gpresult`/`rsop.msc` for "did this GPO apply at all").
- **Unlocking vs. resetting a password** — two distinct admin operations: unlocking clears the lockout flag without touching the password; resetting sets a new password without guaranteeing the account gets unlocked.
- **Disabled vs. locked accounts** — a further distinction from the same family: locked is a self-clearing automatic state from failed logons, disabled is a deliberate, sticky admin action. Different ADUC indicators (down-arrow icon and the Enable/Disable context-menu label vs. the "Unlock account" checkbox), different real-world causes, different fixes.
- **A stale/cached session can outlive the account state that created it** — the same lesson that showed up with secure channels and RSoP now showed up again with a disabled account: an already-open session doesn't necessarily reflect what AD would say right now. A forced fresh logon is often the only way to see the true current state.
- **Choosing a valid comparison account when scoping an issue** — testing with *any* other account isn't enough; the comparison has to actually go through the same access path as the affected user. Testing Kitchen access with an IT_Staff account (a different, layered permission) gave a misleading "it works" result — the valid test was another real Kitchen_Staff member.
- **Two different root causes can produce an identical-looking symptom** — TICKET-01 (disabled account) and TICKET-02 (missing group membership) both presented as "Paul can't access the Kitchen share or K: drive," but required completely different diagnostic paths and fixes. Matching symptom text to a remembered cause is a trap; each ticket needs its own investigation.
- **One user vs. everyone changes where you look first** — an issue isolated to one person points at that person's account/group state; an issue hitting a whole department at once points at the shared resource itself (a share's permissions, a GPO) — scoping the blast radius early shapes the whole investigation.
- **GPO troubleshooting layers** — `gpresult /r` / `rsop.msc` answers "did this GPO apply, and why or why not"; testing the underlying resource/behavior directly answers "is the setting actually taking effect in practice." A setting can be confirmed *delivered* (via RSoP) while still not visually *triggering*, which points the investigation toward the environment rather than the policy itself.
- **RSoP "Applied" vs. a live session actually using the new value** — RSoP/`gpresult` confirm the registry value is current, but a desktop session that was already running before the policy changed may not re-read certain values (like the screensaver idle timer) until a fresh logon. "Applied" and "already in effect in this exact running session" are not always the same thing — a subtler layer of GPO troubleshooting than scope/precedence alone.
- **Computer Configuration vs. User Configuration, in practice** — confirmed with two settings in the same GPO: the screen lock (User Configuration) follows the *user's* AD location and shows under `gpresult /r`'s USER SETTINGS; the removable storage block (Computer Configuration) follows the *machine's* AD location and shows under COMPUTER SETTINGS — independently of who's logged in.
- **Verifying delivery vs. verifying live behavior** — when a real test isn't possible (e.g. no spare USB drive to test a removable-storage block), confirming the policy reached the target via `gpresult /r` is a legitimate, honest interim verification — as long as it's documented as "delivery confirmed, behavior untested," not silently treated as a full pass.
- **Reading a PowerShell error message precisely** — a "term not recognized" error naming a partial or oddly-split command name is often a typo (e.g. a stray space breaking a cmdlet name into two tokens), not a sign the command or approach is wrong.
- **GUI and CLI equivalence** — every operation has been done both through the GUI and via PowerShell/CLI equivalents, deliberately, to build comfort with both.

---

## Commands & terminology reference

Quick definitions for the commands and terms that show up throughout this journal — a lookup table, not a substitute for the fuller explanations above.

**Active Directory / PowerShell**
- `New-ADGroup` — creates a new AD security group via PowerShell (the CLI equivalent of ADUC's "New > Group").
- `Add-ADGroupMember` — adds a user into an existing AD group via PowerShell.
- `Get-ADGroup` / `Get-ADUser` — look up an existing AD group/user object; used to confirm an object exists (or doesn't).
- `Test-ComputerSecureChannel` — checks whether a computer's trust relationship with the domain is healthy; returning `True` doesn't by itself prove a user can log in, and a working-looking login doesn't by itself prove this would return `True`.
- `Set-DnsClientServerAddress` — sets which DNS server(s) a network interface uses.

**Domain trust**
- `nltest /sc_verify:<domain>` — command-line tool for testing the secure channel (trust relationship) between a computer and the domain; "sc" = secure channel.

**File shares & permissions**
- `New-SmbShare` — creates a network share via PowerShell (the CLI equivalent of Explorer's "Share" dialog).
- **SMB (Server Message Block)** — the network protocol Windows uses for file sharing. A "share" is simply a folder made available over the network via SMB — `\\DC01\Kitchen` reaches DC01 over SMB.
- `icacls` — command-line tool for viewing/editing NTFS permissions from the terminal, instead of the Security tab in folder Properties.
- **ACL (Access Control List)** — the list attached to a file or folder saying who's allowed to do what with it. Every NTFS permission set in this project is really an ACL entry; the orphaned-SID incident (Service department) was an ACL entry pointing at a deleted group's old SID.

**Group Policy**
- `gpupdate /force` — forces a client to immediately re-check and reapply Group Policy, instead of waiting for the normal background refresh interval.
- `gpresult /r` — shows which GPOs actually applied (and which were filtered out) for the current user/computer — the main diagnostic tool used throughout Phase 3 for "is this GPO actually reaching the target, and why or why not."
- `rsop.msc` — opens the Resultant Set of Policy console: a live report of the final, merged settings after every applicable GPO is combined and conflicts resolved for the current session.
- **RSoP (Resultant Set of Policy)** — the concept behind `rsop.msc`: a *snapshot* of the effective settings for a specific user/computer at that moment — confirms a setting was delivered, not that it's already active in an already-running session (see the screen-lock incident in Phase 2).

**Other**
- `net use K: /delete` — fully removes a mapped network drive on a client. Used repeatedly in this project as the "clean test" method: remove the drive, force a fresh logon, and see whether it comes back on its own — the only honest way to test whether a GPO change actually took effect.

## Troubleshooting log

| Issue | Symptom | Root cause | Fix |
|---|---|---|---|
| Kitchen NTFS leak | Joyce could access Kitchen share despite not being staff | Inherited `Users` NTFS permission from parent folder | Broke inheritance, scoped to `Kitchen_Staff` only |
| CLIENT02/03 secure channel | Domain login appeared to work but `Test-ComputerSecureChannel` = False | Broken computer/domain trust from cloned VMs; stale computer objects | Rejoined clients to domain after clearing stale AD computer objects |
| Service_Staff disappeared | Group existed and worked, then `Get-ADGroup` returned not found after a VM power event | Likely VM state/suspend behavior on DC01 (not fully confirmed) | Recreated group and members; had to also fix NTFS permissions due to orphaned SID |
| Password/lockout policy not applying | Account didn't lock out after repeated wrong passwords; client only showed a generic "delay" message | `Default Domain Policy` (Link Order 1) already had Account Lockout Threshold explicitly set to 0, overriding the new GPO (Link Order 2) configuring threshold = 5 | Reordered the new GPO to Link Order 1 — resolved per-setting, so it also fixed the Password Policy half at the same time, without editing Default Domain Policy |
| Screen lock not visually triggering | Idle 60+ sec on CLIENT01, screen never blanked/locked, despite RSoP confirming the setting was delivered | The test session was already running before the policy took effect — a background `gpupdate /force` doesn't guarantee an already-running session's screensaver watchdog re-reads the new timeout live | Logged off/restarted and logged back in fresh; screen lock then triggered at exactly 60 seconds on both CLIENT02 (Bar user) and CLIENT01 (Steven), confirming it wasn't VM- or department-specific |
| TICKET-01: Paul can't access Kitchen share/K: drive | Paul reportedly "could log in" but got no access to the Kitchen share or his mapped K: drive; every other Kitchen user was fine | Paul's AD account was disabled; the "he can log in" report was based on a stale session opened before the disable, not a fresh live authentication | Enabled the account in ADUC; verified with a fresh logout/login and confirmed both share and drive access returned to normal |
| TICKET-02: Paul can't access Kitchen share/K: drive (again) | Same-looking symptom as TICKET-01 — Paul denied on the Kitchen share and K: drive — but this time logon itself worked fine, and the error was an explicit "you do not have permission" from the server | Paul had been removed from the `Kitchen_Staff` security group (had only `Domain Users`); both the share's NTFS permissions and the drive-mapping GPO are scoped to that group | Re-added Paul to `Kitchen_Staff`; anticipated the existing session's Kerberos token wouldn't reflect the new membership, had Paul restart for a fresh logon, then confirmed both share and drive access worked |
| TICKET-03: whole Kitchen department locked out | Entire Kitchen department denied on both the Kitchen share and the K: drive at once — not isolated to one user | `Kitchen_Staff` had been removed entirely from the Kitchen folder's NTFS permissions (IT_Staff's separate entry was untouched, confirming a targeted single-entry removal, not a full ACL reset) | Re-added `Kitchen_Staff` to the folder's NTFS permissions at Modify; correctly predicted no restart would be needed (NTFS checks are evaluated live, unlike cached group tokens) and confirmed with two different users (Paul, Steven) |
| TICKET-04: Joyce can't access Bar share (DNS) | Joyce prompted for network credentials with "the system cannot contact a domain controller" — a failure before authentication, not an access-denied error | CLIENT02's DNS client was pointed at `8.8.8.8` and the default gateway instead of DC01 (`192.168.138.10`), so the client couldn't locate a domain controller at all | Corrected DNS client setting to `192.168.138.10` only via `Set-DnsClientServerAddress`; confirmed with `ipconfig /all`; no restart needed since DNS settings are read live; verified Joyce could access the Bar share immediately |
| TICKET-05: Steven's K: drive stops delivering to new sessions (GPO not applying) | K: still worked in Steven's already-open/cached session even after the GPO was broken (a misleading first test); after properly deleting the drive on the client and forcing a fresh logon, K: did not come back | `Kitchen - Map Network Drive` GPO had Security Filtering emptied (nobody had Apply Group Policy permission) and GPO Status set to "User configuration settings disabled" — confirmed via `gpresult /r` showing the GPO in neither the Applied nor the filtered-out list | Restored `Authenticated Users` to Security Filtering and set GPO Status back to Enabled; verified by logging Steven off/on — K: reappeared automatically with no manual recreation |
| Kitchen drive map misfiled in the wrong GPO | `Kitchen - Map Network Drive`'s Drive Maps view showed empty even after console refreshes and a full shutdown/restart of everything, yet K: kept working for all of Kitchen | The Drive Maps preference item had actually been created in `GPO - Kitchen User Restrictions` (also linked to Kitchen OU) instead of `Kitchen - Map Network Drive` — Group Policy delivered it regardless of which GPO held it | Deleted the misfiled item (confirmed via clean test that it was the true source — K: stopped delivering); force-cleaned already-affected clients with a temporary Action = Delete preference item; rebuilt a proper Action = Create item in the correct GPO; verified with a fresh logon |

---

## Roadmap

- **Phase 1 — ✅ Complete:** All five departments have groups, shares, and tested NTFS permissions, including a layered cross-department model for IT and a walled-off model for Management.
- **Phase 2 — ✅ Complete:** Drive mapping (all 5 departments), domain-wide Password/Account Lockout Policy (including a real GPO precedence troubleshooting incident), and the Workstation Security Baseline (screen lock fully verified live after tracking down a stale-session root cause; removable storage block confirmed delivered via `gpresult /r`, live behavior untested pending a spare USB device). Software deployment via GPO skipped as a legacy technique.
- **Phase 3 — ✅ Complete:** Ticket-style troubleshooting labs, diagnose-first. Five deliberately-staged tickets resolved: TICKET-01 (disabled account), TICKET-02 (missing group membership), TICKET-03 (department-wide NTFS permission removal), TICKET-04 (DNS misconfiguration breaking domain controller location), TICKET-05 (GPO not applying — Security Filtering emptied + User Configuration disabled, including recovering from a misleading first test caused by leftover session state). The sixth planned category, domain trust failure, was already demonstrated in full by a real (unplanned) incident earlier in the project (CLIENT02/CLIENT03 secure channel repair) and wasn't restaged. Together these cover account state, group/token caching, live resource permissions, network/name resolution, GPO scope/delivery, and domain trust — six distinct diagnostic layers.
- **Post-Phase 3 real incident — ✅ Resolved:** Kitchen's drive map preference was discovered misfiled in `GPO - Kitchen User Restrictions` instead of `Kitchen - Map Network Drive`, found while doing a routine config check (not staged). Diagnosed with a genuinely harder puzzle than any Phase 3 ticket — a management console that disagreed with live behavior even after a full environment restart — and resolved by locating the real source, force-cleaning already-affected clients with an Action = Delete preference item, and rebuilding the setting in the correct GPO.
- **Phase 4 — ✅ Complete:** Full documentation pass for CV/LinkedIn/portfolio and interview use — this journal, the GitHub README, the technical reference, and the skills log.

**All four phases of this project are complete.**
