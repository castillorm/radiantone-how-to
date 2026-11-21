# RadiantOne 7.4 How-To: Persistent Cache with a Synthetic HDAP Source

**Category:** Persistent cache and replication  
**Estimated duration:** 90–120 minutes  
**Audience:** Identity engineers, administrators, and support engineers  
**Platform:** RadiantOne 7.4, nonproduction; PowerShell examples for Windows  
**Status:** Guided exercise; confirm version-specific console labels and cache refresh options in your installed patch

> **Public GitHub safety:** All example identities, DNs, and addresses are fictional. Never publish real credentials, hostnames, internal IP addresses, customer configurations, or unredacted logs. This is an independent educational exercise, not official Radiant Logic documentation.

## What you will learn

1. Create an isolated HDAP-backed LDAP source and import three test identities.
2. Build a **virtual Raw Layer** using the **Directory Tree Wizard**.
3. Verify source-to-Raw DN mapping and prove that the uncached view works.
4. Enable and initialize persistent cache on the appropriate Raw branch.
5. Determine which refresh mechanism actually applies to your configuration.
6. Test create, modify, and delete propagation and recognize stale cache data.
7. Locate initialization evidence, LDAP server logs, Sync Engine logs, and periodic refresh logs.
8. Restore settings and collect publication-safe evidence.

## Architecture and key concepts

```text
Synthetic HDAP source
  o=lab-source
    ou=people
      uid=alice.smith
      uid=bob.jones
      uid=carol.lee
          |
          | LDAP backend connection / DN mapping
          v
Virtual Raw namespace (initially uncached)
  ou=raw,o=pcache-lab
          |
          | Explicit cache configuration + initial population
          v
Persistent cache (materialized HDAP-backed image)
          |
          v
LDAP client queries published Raw namespace
```

The diagram is **conceptual**. The LDAP client queries the published namespace; do not assume the cache is serving reads until cache configuration and status confirm it. A **virtual view** retrieves source records through a mapping; **persistent cache** stores a materialized image. A **refresh pipeline** maintains that image. The appropriate refresh mechanism depends on the source and how the cache is configured.

## Lab configuration worksheet

| Setting | Synthetic example | Verify locally |
|---|---|---|
| Source naming context | `o=lab-source` | Available / unused |
| Source user container | `ou=people,o=lab-source` | Exists after import |
| Raw root naming context | `o=pcache-lab` | Available / unused |
| Published Raw branch | `ou=raw,o=pcache-lab` | DN mapping confirmed |
| Source LDAP data source | `LAB_HDAP_SOURCE` | Test connection succeeds |
| Source LDAP host | `localhost` (example only) | Actual reachable host |
| Source LDAP port | `2389` (example only) | Actual listener |
| Bind account | `<LAB_BIND_DN>` | Appropriate permissions |
| Installation directory | `<RLI_HOME>` | Actual path |
| Cache root | `<ACTUAL_CACHE_ROOT_DN>` | Read from console |
| Refresh type | `<ACTUAL_REFRESH_TYPE>` | Read from console |

**Prerequisites:** RadiantOne 7.4 Main Control Panel and Server Control Panel access; permission to configure HDAP, LDAP data sources, namespace backends and persistent cache; LDAP Browser or Apache Directory Studio; and a disposable lab environment. Before initialization, identify the leader node and check available disk space.

**Do not** perform this exercise on a customer or production naming context. Do not create a second backend under an already occupied Raw branch.

---

## Lab 1 — Create the HDAP source (15–20 minutes)

### 1A. Create a dedicated source naming context

1. Sign in to the **RadiantOne Main Control Panel** with a lab administrator account.
2. Click **Directory Namespace**.
3. Expand the directory namespace tree and find the control for creating a **new root naming context**. Depending on the 7.4 patch and tree selection, this may be available from a context menu or an **Add** control.
4. Enter `o=lab-source` as the naming context.
5. Select the **HDAP / RadiantOne Universal Directory** storage backend when prompted. **Do not select an LDAP proxy backend for the source store.**
6. Review the proposed store and naming context, then click **Create**, **OK**, or **Save** as presented by your build.
7. Expand the namespace tree and confirm `o=lab-source` is present and backed by HDAP.

**UI verification checkpoint:** The exact new-HDAP-store dialog labels are not established by the supplied tutorial. If your screen differs, stop and verify the store-creation workflow in the installed 7.4 guide rather than guessing which backend type to select.

### 1B. Prepare synthetic LDAP entries

Save this as `sample-users.ldif` locally:

```ldif
dn: o=lab-source
objectClass: top
objectClass: organization
o: lab-source

dn: ou=people,o=lab-source
objectClass: top
objectClass: organizationalUnit
ou: people

dn: uid=alice.smith,ou=people,o=lab-source
objectClass: top
objectClass: inetOrgPerson
uid: alice.smith
cn: Alice Smith
sn: Smith
givenName: Alice
mail: alice.smith@example.com
employeeNumber: LAB1001

dn: uid=bob.jones,ou=people,o=lab-source
objectClass: top
objectClass: inetOrgPerson
uid: bob.jones
cn: Bob Jones
sn: Jones
givenName: Bob
mail: bob.jones@example.com
employeeNumber: LAB1002

dn: uid=carol.lee,ou=people,o=lab-source
objectClass: top
objectClass: inetOrgPerson
uid: carol.lee
cn: Carol Lee
sn: Lee
givenName: Carol
mail: carol.lee@example.com
employeeNumber: LAB1003
```

If `o=lab-source` already exists, remove the first LDIF entry before import. If `ou=people` exists, omit that entry too.

### 1C. Import with LDAP Browser

1. Open **LDAP Browser** (or Apache Directory Studio).
2. Create or select a connection to the RadiantOne LDAP listener; use your actual host and port.
3. Authenticate with a lab account that can add entries to the source store.
4. Browse to `o=lab-source`.
5. Select the LDIF import action (typically **Import** or **Import LDIF** in your LDAP client, not the RadiantOne cache wizard).
6. Choose `sample-users.ldif`, review the import preview, and run the import.
7. Confirm there are no `entryAlreadyExists`, `invalidAttributeSyntax`, or `objectClassViolation` errors.

### 1D. Validate source records

In LDAP Browser choose **Search** and enter:

```text
Base DN: ou=people,o=lab-source
Scope: Subtree
Filter: (objectClass=inetOrgPerson)
Attributes: uid,cn,sn,givenName,mail,employeeNumber
```

**Expected:** Three entries, `alice.smith`, `bob.jones`, and `carol.lee`. Record the count. If this fails, do not continue to the Raw Layer.

---

## Lab 2 — Create the Raw Layer with the Directory Tree Wizard (20–25 minutes)

> **Chosen approach:** The Directory Tree Wizard mounts an LDAP backend into a virtual namespace. It avoids the manual ORX/DVX Context Builder workflow. Do not mix the two procedures unless you deliberately want to compare them.

### 2A. Create and test the LDAP data source

1. In **Main Control Panel**, click **Settings**.
2. Expand **Server Backend** in the navigation tree.
3. Click **LDAP Data Sources**.
4. Click **Add**.
5. Set **Data Source Name** to `LAB_HDAP_SOURCE`.
6. Enter the LDAP server **Host** (for example `localhost` only if the source listener is local).
7. Enter the actual LDAP **Port**. `2389` is only a placeholder.
8. Choose the connection security supported by your lab listener. Do not disable TLS verification merely to make a connection test pass.
9. Enter the dedicated lab **Bind DN** and password in the console; **do not** record either value in GitHub.
10. Where the data source dialog provides a source base or search base, use `ou=people,o=lab-source`. If the data source dialog has no such field, you will supply this DN in the backend mount mapping.
11. Click **Test Connection** (or the equivalent test control in your patch).
12. Confirm a successful bind, then click **Save** / **OK**.

**Expected:** `LAB_HDAP_SOURCE` appears in the LDAP Data Sources list. A successful bind alone does not prove that the account can search the source subtree; use the source search from Lab 1 as an additional check.

### 2B. Launch the Directory Tree Wizard

1. Click the **Wizards** tab in Main Control Panel.
2. Select **Directory Tree Wizard**.
3. On the Welcome screen click **Next**.
4. Enter `o=pcache-lab` as the **root naming context**.
5. Click **Next** to open the hierarchy designer.

If `o=pcache-lab` already exists, do not blindly recreate it. Inspect the existing namespace and either reuse an intentionally empty lab root or choose a different lab root.

### 2C. Add the Raw label

1. Select the `o=pcache-lab` root in the hierarchy designer.
2. Click **Create Level**.
3. Set **Level Type** to `ou`.
4. Enter `raw` as the level value.
5. Click **OK**.
6. Confirm the hierarchy displays:

```text
o=pcache-lab
  ou=raw
```

7. Select `ou=raw` and click **Next** to proceed to backend configuration.

### 2D. Mount the LDAP backend

1. Select the `ou=raw` leaf node on the backend configuration page.
2. Click **Mount LDAP Backend**.
3. Open the **Data Source** dropdown and choose `LAB_HDAP_SOURCE`.
4. Set the source/remote base to `ou=people,o=lab-source` if the wizard asks for it.
5. Set the published/mapped base to `ou=raw,o=pcache-lab` if the wizard asks for it.
6. Review the resulting backend mapping before continuing.
7. Click **Next**.
8. On the review screen, confirm the root, Raw label, source data source, and source base.
9. Click **Create**; wait for the confirmation; click **OK** and **Finish** if shown.

**Critical mapping:**

```text
Remote Base DN: ou=people,o=lab-source
Mapped Base DN: ou=raw,o=pcache-lab
```

### 2E. Inspect the resulting namespace mapping

1. Return to **Directory Namespace**.
2. Expand **Root Naming Contexts** and then `o=pcache-lab`.
3. Select `ou=raw` or its LDAP backend child (the selected object depends on your console tree).
4. Open the **Proxy Backend** tab, if present.
5. Locate **Remote Base DN** and **Mapped Base DN**.
6. Confirm the two values shown above. Correct them only if the editor allows it and you understand the resulting DN rewrite.
7. Click **Save** and confirm **Yes** if prompted.

If the **Proxy Backend** tab is not shown, inspect the selected backend's properties; do not assume the wizard created the correct mapping without verification.

### 2F. Prove the Raw virtual view works *before* caching

1. Open **Directory Browser** or your LDAP client.
2. Start a new subtree search with:

```text
Base DN: ou=raw,o=pcache-lab
Scope: Subtree
Filter: (objectClass=inetOrgPerson)
Attributes: uid,cn,mail,employeeNumber
```

3. Confirm three identities appear under the Raw base.
4. Open Alice's entry and confirm its DN is `uid=alice.smith,ou=raw,o=pcache-lab`.
5. Compare its `mail` and `employeeNumber` values with the HDAP source.

**Checkpoint:** Source count = 3; Raw count = 3; DNs mapped correctly. This proves the *virtual view*, **not** the persistent cache.

### 2G. Optional: prove the Raw view is virtual

Before caching, modify Alice's source `mail` attribute to `alice.virtual@example.com`, save, and immediately search Raw again. If the Raw view is reading the source, it should show the changed value. Restore `alice.smith@example.com` before the cache lab. This is a diagnostic observation, not a guaranteed latency benchmark.

---

## Lab 3 — Configure and initialize persistent cache (20–25 minutes)

> **Version-sensitive section:** Cache creation and refresh screens can differ between RadiantOne 7.4 patches and deployment architectures. The instructions below give the navigation and decisions to verify, but do not invent a button label or assert that a cache exists until the console confirms it.

### 3A. Record the uncached baseline

1. In **Directory Namespace**, select `ou=raw,o=pcache-lab`.
2. Note whether the selected branch is currently configured as an LDAP proxy/virtual backend.
3. Capture the source data source name and remote/mapped base DN values.
4. Run the Raw search from Lab 2 once more and record the count of three.
5. Identify the RadiantOne **leader** using the Main Control Panel dashboard or your cluster administration procedure.

### 3B. Open cache configuration

1. In Main Control Panel open **Directory Namespace**.
2. Find and expand the **Cache** node, if available in your console.
3. Inspect existing cached branches. Confirm no cache already owns `ou=raw,o=pcache-lab` or an overlapping suffix.
4. Select the Raw branch or the cache-creation workflow provided for that branch.
5. Choose **persistent cache**, not a transient memory cache.
6. Associate the cache with the already validated Raw virtual backend; verify the displayed source backend and base DN.
7. Review the cache root DN that the wizard proposes. Write the *actual* value into `<ACTUAL_CACHE_ROOT_DN>` in your worksheet.
8. Save the configuration and reopen it to verify the selected branch is marked for persistent caching.

**Stop condition:** If you cannot identify the exact cache-creation action or its source view, stop here and inspect the installed 7.4 documentation or share a sanitized screenshot. Do not create a new HDAP store at the same DN as the Raw proxy branch as a workaround.

### 3C. Inspect and select the refresh mechanism

1. Select the configured persistent-cache branch.
2. Open **Refresh Settings** (or the equivalent cache refresh configuration panel).
3. Identify the available refresh mechanisms and record which one is actually selected.
4. For a compatible HDAP source, check whether **HDAP Trigger** real-time refresh is supported and configured. **Do not assume** it is automatically active just because the source uses HDAP.
5. If the environment instead uses periodic refresh, record the interval and confirm which periodic job will maintain the cache.
6. Save and reopen the settings to confirm persistence.
7. Locate the refresh pipeline, job, or processor status screen associated with the configured mechanism.

**Why this matters:** HDAP Trigger, connector-based sync, and periodic cache refresh have different evidence and log locations. Troubleshooting the wrong mechanism can be misleading.

### 3D. Initialize the cache

1. Confirm you are administering the leader node.
2. Open the cached branch's **Refresh Settings**.
3. Click **Initialize** if this action is offered in your 7.4 patch.
4. Select the option to generate a new source LDIF/image from the backend view, rather than importing an unrelated customer file.
5. Confirm the intended source DN and cache root.
6. Start initialization.
7. Open **Server Control Panel → Tasks** and find the initialization task.
8. Wait for a terminal status; inspect the task's details and error output.
9. Return to the cache configuration and confirm it reports initialized/ready or the corresponding successful state.

**Expected:** Initialization completes and three synthetic user records are present. The cache may contain additional container entries; count only `(objectClass=inetOrgPerson)` for the user total.

### 3E. Prove the cache is configured, not merely returning virtual results

1. Reopen the Raw branch configuration and confirm **persistent cache** is enabled.
2. Confirm the initialization task completed successfully.
3. Confirm the configured cache root and refresh mechanism.
4. Query `ou=raw,o=pcache-lab` and verify three user entries.
5. Record these four independent pieces of evidence. **A successful LDAP query alone is insufficient proof of cache usage.**

---

## Lab 4 — Locate logs and capture a baseline (15 minutes)

### 4A. Know which evidence to check

| Component or evidence | Purpose | When relevant |
|---|---|---|
| `vds_server.log` | LDAP server/backend errors and exceptions | Query, bind, routing or server issues |
| `sync_engine/sync_engine.log` | Sync Engine and HDAP Trigger processing | When Sync Engine / HDAP Trigger is configured |
| `periodiccache.log` | Periodic cache refresh activity | Only when periodic refresh is configured |
| Connector logs | Connector capture or ingestion failures | Only when a connector is part of the refresh topology |
| Sync agent logs | Agent processing issues | Only when a sync agent is configured |
| Server Control Panel **Tasks** | Initialization and task results | During cache initialization or other tasks |
| Cache status / refresh settings | Configured cache root, state, refresh method | Always |

**Important:** Filenames and log folder locations may vary by patch or installation configuration. Discover actual files on disk; do not infer that every component is installed or enabled.

### 4B. Find the logs on Windows

Open **PowerShell** on the lab server (or a permitted administrative workstation with access to the log directory):

```powershell
# Replace with the real installation directory.
$RLI_HOME = 'C:\radiantone\vds'
$LogRoot = Join-Path $RLI_HOME 'vds_server\logs'

if (-not (Test-Path $LogRoot)) {
    Write-Warning "Log directory not found: $LogRoot. Verify RLI_HOME."
} else {
    Get-ChildItem -Path $LogRoot -Recurse -File |
        Where-Object { $_.Name -match 'vds_server|sync_engine|periodiccache|connector|cache' } |
        Select-Object FullName, Length, LastWriteTime
}
```

The sample `$RLI_HOME` path is not an assertion about your installed directory.

### 4C. Read the server log

```powershell
$ServerLog = Join-Path $LogRoot 'vds_server.log'
if (Test-Path $ServerLog) {
    Get-Content -Path $ServerLog -Tail 100
}
```

### 4D. Monitor Sync Engine when applicable

```powershell
$SyncLog = Join-Path $LogRoot 'sync_engine\sync_engine.log'
if (Test-Path $SyncLog) {
    Get-Content -Path $SyncLog -Tail 50 -Wait
} else {
    Write-Host 'Sync Engine log not found at this example path; inspect discovered files.'
}
```

Press **Ctrl+C** to stop live monitoring.

### 4E. Search for troubleshooting keywords

```powershell
Get-ChildItem -Path $LogRoot -Recurse -File -Filter '*.log' |
    Select-String -Pattern 'ERROR','WARN','cache','trigger','initialize','refresh' |
    Select-Object Path, LineNumber, Line
```

This search may generate many matches; narrow to a specific file and a short test window when possible. **Do not publish raw output** without sanitizing it.

### 4F. Temporarily enable diagnostic logging

1. Open **Main Control Panel → Settings → Logs → Log Settings**, if these labels match your build.
2. Find the **Sync Engine** logging component if the refresh topology uses it.
3. Record the original log level.
4. Temporarily set **DEBUG** for the shortest practical test window.
5. Save the change and perform one synthetic source modification.
6. Inspect the corresponding log entries.
7. Restore the original level immediately after the test.

DEBUG can increase disk usage and expose attribute values. Avoid global DEBUG logging when targeted component logging is sufficient.

---

## Lab 5 — Test synchronization and troubleshoot stale cache (15–20 minutes)

### 5A. Update Alice's email

1. In LDAP Browser navigate to the **source** entry `uid=alice.smith,ou=people,o=lab-source`.
2. Edit the `mail` attribute from `alice.smith@example.com` to `alice.updated@example.com`.
3. Click the LDAP client's **Save** / **Commit** control.
4. Re-read the **source** entry to confirm the change persisted.
5. Search the **Raw** branch for `(uid=alice.smith)`.
6. Note whether and when `mail: alice.updated@example.com` appears.
7. Inspect the refresh job/pipeline status and its relevant log for the same time window.

**Expected:** With a healthy, functioning refresh mechanism, the cache eventually reflects the updated value. Do not claim a specific propagation time unless measured.

### 5B. Add David to the source

Import this single-entry LDIF into `ou=people,o=lab-source`:

```ldif
dn: uid=david.miller,ou=people,o=lab-source
objectClass: top
objectClass: inetOrgPerson
uid: david.miller
cn: David Miller
sn: Miller
givenName: David
mail: david.miller@example.com
employeeNumber: LAB1004
```

1. Confirm David exists in the **source**.
2. Search Raw using `(objectClass=inetOrgPerson)`.
3. If Raw still shows three entries, check refresh status and logs rather than immediately reinitializing.
4. Once synchronized, Raw should show **four** user entries.

### 5C. Delete David from the source

1. In LDAP Browser select `uid=david.miller,ou=people,o=lab-source`.
2. Choose the LDAP client's **Delete Entry** action and confirm the exact DN.
3. Verify the entry is absent from the **source**.
4. Query Raw again after refresh.
5. Confirm the Raw user count returns to **three**.

### 5D. Restore the initial lab state

1. Change Alice's source `mail` back to `alice.smith@example.com`.
2. Verify the source and Raw views both show the original value.
3. Verify the user count remains three.
4. Restore any diagnostic logging settings changed in Lab 4.

### 5E. If the cache is stale, follow this order

1. **Source truth:** Does the HDAP source show the modification?
2. **Mapping:** Does the Raw branch point to the intended source and DN?
3. **Cache configuration:** Is persistent caching actually enabled and initialized?
4. **Refresh type:** Is HDAP Trigger, periodic refresh, or another mechanism configured?
5. **Pipeline/job status:** Is that mechanism running successfully?
6. **Logs:** Is there evidence of detection, processing, or an error during the test window?
7. **Repair:** Only after identifying the cause, consider the documented delta-refresh or reinitialization procedure for the *actual* cache root. Do not use a guessed DN or perform destructive cache operations on production.

---

## Troubleshooting reference

| Symptom | First UI check | Log/evidence | Likely next step |
|---|---|---|---|
| Source search returns zero | Source HDAP namespace and LDAP permissions | `vds_server.log` | Fix source data or bind/search permissions |
| Data source test fails | **Settings → Server Backend → LDAP Data Sources** | `vds_server.log` | Correct host, port, TLS or bind configuration |
| Raw returns zero but source returns three | **Directory Namespace → Raw → Proxy Backend** | `vds_server.log` | Check remote/mapped base DNs |
| Raw returns three but cache not initialized | **Cache** configuration and **Tasks** | Initialization task output | Initialize the intended cache branch |
| Cache initialization fails | **Refresh Settings → Initialize**, **Tasks** | Task details, server logs | Check source image, schema and disk |
| Source update not reflected in Raw | Refresh mechanism and job status | `sync_engine.log` or `periodiccache.log` as applicable | Diagnose pipeline, schedule or failed updates |
| Query fails after caching | Cache status, branch DN and ACLs | `vds_server.log` | Check namespace routing and permissions |
| DEBUG logging too noisy | **Settings → Logs → Log Settings** | Log growth | Restore previous level |

### Example diagnostic messages — *fictional, not verbatim RadiantOne output*

```text
[LAB EXAMPLE] Source change detected for uid=alice.smith
[LAB EXAMPLE] Cache update applied
[LAB EXAMPLE] Cache initialization failed: backend image unavailable
```

Use these only as interpretation examples; do not search for these exact strings expecting them to appear in real RadiantOne logs.

---

## Lab evidence worksheet

| Checkpoint | Expected | Actual result / notes |
|---|---|---|
| Source HDAP users | 3 | |
| Raw virtual users before caching | 3 | |
| Alice DN mapped correctly | Yes | |
| Persistent cache enabled | Confirmed in UI | |
| Initialization task | Successful | |
| Refresh mechanism | Recorded from UI | |
| Alice update reflected | Yes, after refresh | |
| David added | Count = 4 | |
| David deleted | Count = 3 | |
| Original values restored | Yes | |
| Original log levels restored | Yes | |

**Safe evidence:** Screenshots cropped to synthetic lab entries, a written description of observed events, and redacted error categories. Do not commit full task output, configuration exports, access tokens, service-account names, internal hosts or raw debug logs.

## Cleanup and rollback

1. Verify all synthetic source changes are restored and the cache is healthy.
2. Restore the original logging levels.
3. If the lab is disposable, follow your installed product's documented procedure to **disable and remove the lab cache**, then remove the lab Raw namespace, lab LDAP data source, and lab source HDAP store **in dependency order**.
4. Verify each DN belongs exclusively to this lab before deleting anything.
5. Do not use force-delete or low-level store removal commands without the correct version-specific procedure and a backup.

## Publication checklist

- [ ] Only `example.com` synthetic identities and lab DNs appear.
- [ ] No real server names, IP addresses, credentials or tokens.
- [ ] No unredacted screenshots or log files.
- [ ] UI labels verified against the installed 7.4 patch, or clearly marked as version-sensitive.
- [ ] Cache root and refresh mechanism recorded as *environment-specific*, not hard-coded universal defaults.
- [ ] Guide has been tested end-to-end in an isolated environment before being marked **Tested**.

## Official documentation to consult

- [RadiantOne 7.4 Directory Tree Wizard](https://developer.radiantlogic.com/idm/v7.4/identity-service-wizards-guide/02-directory-tree-wizard/)
- [RadiantOne 7.4 Persistent Cache Management](https://developer.radiantlogic.com/idm/v7.4/operations-guide/04-persistent-cache-management/)
- [RadiantOne 7.4 Logging and Troubleshooting](https://developer.radiantlogic.com/idm/v7.4/logging-and-troubleshooting-guide/03-radiantone-universal-directory/)
- [RadiantOne 7.4 Caching Concepts](https://developer.radiantlogic.com/idm/v7.4/sys-admin-guide/cache/)

**Repository placement:** `guides/radiantone-persistent-cache-hdap-lab.md`  
**Repository:** `https://github.com/castillorm/radiantone-how-to`
