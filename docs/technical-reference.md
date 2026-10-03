# IT Homelab — Technical Reference

The full, exhaustive version of the AD/OU/GPO/permissions setup — companion to the condensed [GitHub README](../README.md) and the narrative [lab journal](lab-journal.md). This is the doc to open if an interviewer says "walk me through exactly how you set that up" and wants the real detail, not the summary.

All GPO setting values below are confirmed from the live environment.

---

## 1. OU structure

```
corp.homelab.test
└── Corp
    ├── Kitchen
    ├── Bar
    ├── Service
    ├── IT
    └── Management
```

All five department OUs sit directly under a single parent `Corp` OU — this is what lets `Corp - Workstation Security Baseline` be linked once at `Corp` and inherit down to every department, instead of being relinked five times.

## 2. Users and computer placement

| Department OU | Users | Computer object |
|---|---|---|
| Kitchen | Steven (Head Chef), Lucas, Justina, Alex, Paul | CLIENT01 |
| Bar | Jonathan (Head), Arino, Joyce, Ahn | CLIENT02 |
| Service | Alia (Head), Kossi, Mirinda, Andrea | CLIENT03 |
| IT | Alkali | — |
| Management | Mathias | — |

CLIENT01/02/03 are placed in their matching department OU (not a separate "Computers" OU) so that Computer Configuration GPOs linked at the department or `Corp` level reach them correctly.

## 3. Security groups

| Group | Type | OU | Members |
|---|---|---|---|
| `Kitchen_Staff` | Global, Security | Kitchen | Steven, Lucas, Justina, Alex, Paul |
| `Bar_Staff` | Global, Security | Bar | Jonathan, Arino, Joyce, Ahn |
| `Service_Staff` | Global, Security | Service | Alia, Kossi, Mirinda, Andrea |
| `IT_Staff` | Global, Security | IT | Alkali |
| `Management_Staff` | Global, Security | Management | Mathias |

One group per department, used for both NTFS access and GPO/GPP scoping (drive maps target these groups' OUs). No permissions are ever assigned to an individual user directly.

## 4. Shares and permissions matrix

All shares live at `C:\Shares\<Department>` on DC01, published as `\\DC01\<Department>`.

| Share | UNC path | Share permission | NTFS — department group | NTFS — `IT_Staff` | NTFS — everyone else |
|---|---|---|---|---|---|
| Kitchen | `\\DC01\Kitchen` | Authenticated Users — Full Control | `Kitchen_Staff` — **Modify** | Read & Execute | No access |
| Bar | `\\DC01\Bar` | Authenticated Users — Full Control | `Bar_Staff` — **Modify** | Read & Execute | No access |
| Service | `\\DC01\Service` | Authenticated Users — Full Control | `Service_Staff` — **Modify** | Read & Execute | No access |
| IT | `\\DC01\IT` | Authenticated Users — Full Control | `IT_Staff` — **Modify** | — (own share) | No access |
| Management | `\\DC01\Management` | Authenticated Users — Full Control | `Management_Staff` — **Modify** | **No access** (deliberately walled off) | No access |

**Why share permissions are all wide open:** every share leaves Share permissions at Authenticated Users/Full Control on purpose, so there's exactly one place access control actually happens — the NTFS ACL. Windows enforces whichever of the two (share vs. NTFS) is more restrictive for a given user, so a wide-open share with a tight NTFS ACL behaves identically to a tight share, without having to keep two permission sets in sync.

**Inheritance:** each department folder has inheritance from its parent broken, so the default `Users` (all domain users) permission inherited from the parent `C:\Shares` folder is not present — this is what the Kitchen NTFS-leak incident (Joyce reaching the Kitchen share) was actually caused by, and the fix pattern (break inheritance, explicit ACL only) was then applied to Bar and Service from the start.

## 5. GPO documentation

| GPO name | Linked at | Configuration area | Key settings | Security filtering |
|---|---|---|---|---|
| `Kitchen - Map Network Drive` | Kitchen OU | User Config → Preferences → Drive Maps | Action: Create · `\\DC01\Kitchen` · Drive K: · Reconnect: on | Authenticated Users (default) |
| `Bar - Map Network Drive` | Bar OU | User Config → Preferences → Drive Maps | Action: Create · `\\DC01\Bar` · Drive letter: **B:** · Reconnect: on | Authenticated Users (default) |
| `Service - Map Network Drive` | Service OU | User Config → Preferences → Drive Maps | Action: Create · `\\DC01\Service` · Drive letter: **S:** · Reconnect: on | Authenticated Users (default) |
| `IT - Map Network Drive` | IT OU | User Config → Preferences → Drive Maps | Action: Create · `\\DC01\IT` · Drive letter: **I:** · Reconnect: on | Authenticated Users (default) |
| `Management - Map Network Drive` | Management OU | User Config → Preferences → Drive Maps | Action: Create · `\\DC01\Management` · Drive letter: **M:** · Reconnect: on | Authenticated Users (default) |
| `GPO - Kitchen User Restrictions` | Kitchen OU | User Config → Administrative Templates → Control Panel | Prohibit access to Control Panel and PC settings = Enabled | Authenticated Users (default) |
| `Domain - Password and Lockout Policy` | **Domain root** (`corp.homelab.test`) | Computer Config → Policies → Security Settings → Account Policies → **Password Policy** | Enforce password history: **5 passwords remembered** · Maximum password age: **90 days** · Minimum password age: **1 day** · Minimum password length: **10 characters** · Complexity requirements: **Enabled** · Minimum password length audit / Relax minimum length limits / Reversible encryption: **Not Defined** | Authenticated Users (default) |
| *(same GPO)* | *(same)* | Computer Config → Policies → Security Settings → Account Policies → **Account Lockout Policy** | Lockout threshold: **5 invalid attempts** (confirmed via TICKET testing) · Lockout duration: **1 minute (60 seconds)** · Reset counter after: **10 minutes** | — |
| `Default Domain Policy` (built-in) | Domain root | Computer Config → Security Settings → Account Policies | Account Lockout Threshold explicitly set to 0 (disabled) — conflicts with the GPO above | Authenticated Users (default) |
| `Corp - Workstation Security Baseline` | **`Corp` OU** (parent — inherited by all 5 department OUs) | User Config: Control Panel → Personalization. Computer Config: System → Removable Storage Access | Screen saver: Enabled · Timeout: 60 sec (test value) · Password-protect: Enabled · Screen saver executable: `scrnsave.scr`. Removable Storage: "All Removable Storage classes: Deny all access" = Enabled | Authenticated Users (default) |

**Link order / precedence at the domain root:** `Domain - Password and Lockout Policy` is Link Order 1, `Default Domain Policy` is Link Order 2 — the reverse of the AD default, changed deliberately after the precedence-conflict incident so the custom policy's explicitly-defined settings win. GPO conflicts resolve **per setting**, not per whole GPO, so this reordering only overrides the specific settings the new GPO defines (password/lockout) — it doesn't blank out anything else Default Domain Policy might otherwise still control.

**Preferences vs. Policies used here:** the five drive-map GPOs and nothing else use **Group Policy Preferences** (a one-time, user-editable default). Everything else — Control Panel restriction, password/lockout, screen saver, removable storage — is a **Policy** (continuously enforced, not user-editable). This distinction is why a drive map can quietly "misfire" into the wrong GPO and still work (Preferences don't care about enforcement source), while a Policy setting reliably shows up correctly scoped in `gpresult`.

## 6. Security Filtering

Security Filtering controls **who a GPO actually applies to**, which is a separate question from **where it's linked**.

- **Linking** (an OU, or the domain root) determines which container of objects a GPO is even in scope for.
- **Security Filtering** decides which specific users/computers/groups within that container actually get it applied, by checking who has the **"Apply Group Policy"** permission on the GPO itself.

By default, every GPO's Security Filtering list contains **Authenticated Users** — meaning "anyone in the linked container gets this, no further restriction." That's the default kept on every GPO in this project.

Security Filtering is the tool for applying a GPO to only part of a container it's linked to — e.g. linking one GPO to the whole `Corp` OU but scoping it, via Security Filtering, to just `Management_Staff`, instead of creating a separate OU.

**What happens if the filter list is emptied entirely** (not narrowed to a different group, but left with nobody in it): with an empty Security Filtering list, *nobody* has permission to have the GPO applied — it doesn't fail quietly for everyone the way a normal deny would. A GPO explicitly denied to one user/group still shows up in `gpresult /r`'s "filtered out" list (Group Policy can see it, just won't apply it to that target). A GPO with an empty filter list doesn't appear in *either* the Applied or the "filtered out" list at all, because no one has permission to even enumerate it during processing. That absence from both lists is itself diagnostic — it means "nobody can read this GPO," not "this GPO was evaluated and excluded."

## 7. Verification checklist

- [x] Password Policy values confirmed via GPMC (history: 5, max age: 90 days, min age: 1 day, min length: 10, complexity: Enabled).
- [x] Drive letters confirmed: K (Kitchen), B (Bar), S (Service), I (IT), M (Management) — first letter of each department.
- [x] Account Lockout Policy values confirmed: duration 1 minute (60 sec), reset counter after 10 minutes, threshold 5 invalid attempts.
- [ ] Confirm whether Security Filtering on any GPO has been narrowed from the default `Authenticated Users` to a specific department group — the table above assumes the default was kept everywhere except where TICKET-05 temporarily emptied it (since restored).

All GPO setting values are confirmed except the Security Filtering check above.

---

*Companion to the [GitHub README](../README.md) (condensed, external-facing) and [lab journal](lab-journal.md) (full narrative log, in build order).*
