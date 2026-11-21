# RadiantOne 7.4 How-To: Persistent Cache with an HDAP Source

**Category:** Persistent cache and replication  
**Duration:** 60 minutes  
**Audience:** Identity engineers and RadiantOne administrators  
**Status:** Guided lab; version-specific UI steps require verification  
**Environment:** Isolated, nonproduction RadiantOne 7.4 installation

> **Publication safety:** All users, domains, and naming contexts in this guide are synthetic. Do not publish real credentials, server addresses, customer exports, or unsanitized logs. This independent educational guide is not official Radiant Logic documentation.

## Objective

Build a small HDAP-backed source, publish its entries through a Raw Layer virtual view, configure persistent caching for that view, and learn which logs to inspect when initialization or synchronization fails.

By the end, you should be able to explain the difference between a virtual view and a persistent cache; validate an initial cache population; test create, update, and delete propagation; and distinguish LDAP server, cache initialization, and synchronization evidence.

## Architecture

```text
LDAP Browser / Test Client
          |
          v
Raw namespace: ou=raw,o=pcache-lab
          |
   Persistent cache (materialized records)
          ^
          | refresh / initialization
Raw virtual view (LAB_HDAP_RAW.dvx)
          ^
          |
Source extraction (LAB_HDAP_SOURCE.orx)
          ^
          |
HDAP source: ou=people,o=lab-source
```

This is a **logical diagram**, not an assertion about the internal execution order of every RadiantOne component. The cache must be explicitly configured; simply creating a DVX does not enable caching.

## Prerequisites and example values

- RadiantOne 7.4 nonproduction instance with Main Control Panel access.
- Permission to create HDAP stores, namespace labels, data sources, virtual views, and cache configurations.
- LDAP Browser or Apache Directory Studio.
- PowerShell for the Windows log exercises (Linux users can use `tail` and `grep`).
- Confirm the actual LDAP listener port, installation directory, leader node, and available disk space.

| Item | Synthetic lab value |
| --- | --- |
| Source naming context | `o=lab-source` |
| Source container | `ou=people,o=lab-source` |
| Raw label | `ou=raw,o=pcache-lab` |
| Data source | `LAB_HDAP_SOURCE` |
| Extraction file | `LAB_HDAP_SOURCE.orx` |
| View file | `LAB_HDAP_RAW.dvx` |
| Test identities | `alice.smith`, `bob.jones`, `carol.lee` |

**Version verification:** Menu names, the supported cache-creation workflow, and the refresh mechanism can differ by 7.4 patch level. Verify these in the installed console before applying changes. Do not create overlapping cache roots or run initialization against production data.

## Lab 1 — Create and verify the HDAP source (10 minutes)

1. In the Main Control Panel, open **Directory Namespace**.
2. Create an isolated HDAP-backed naming context `o=lab-source` using the HDAP store creation workflow in your installed build.
3. Connect with LDAP Browser and verify the naming context is reachable.
4. Create `ou=people,o=lab-source` and the three synthetic entries below. If the root entry already exists, **do not re-import** its LDIF record.
5. Search the source container using `(objectClass=inetOrgPerson)` with subtree scope.

Save the following as `sample-users.ldif` (or copy its entries into LDAP Browser):

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

**Checkpoint:** Exactly three synthetic `inetOrgPerson` entries are returned under the source container.

## Lab 2 — Publish a Raw Layer virtual view (10 minutes)

1. Create the isolated Raw namespace `o=pcache-lab` and label `ou=raw,o=pcache-lab`.
2. Configure an LDAP-compatible source connection named `LAB_HDAP_SOURCE` pointing to the lab HDAP source through your actual RadiantOne LDAP listener. Use a dedicated test bind identity; do not store its password in the repository.
3. Test the connection and browse `ou=people,o=lab-source`.
4. In the view/extraction designer, create `LAB_HDAP_SOURCE.orx` to extract `inetOrgPerson` entries. Include `uid`, `cn`, `sn`, `givenName`, `mail`, and `employeeNumber`.
5. Create `LAB_HDAP_RAW.dvx`, associate the extraction, and map the entries into `ou=raw,o=pcache-lab`. Preserve the `uid` RDN and map the source container DN to the Raw label.
6. Save/deploy the view using the workflow supported by your build.
7. Search `ou=raw,o=pcache-lab` with `(objectClass=inetOrgPerson)`.

Expected DN mapping:

```text
Source: uid=alice.smith,ou=people,o=lab-source
Raw:    uid=alice.smith,ou=raw,o=pcache-lab
```

**Checkpoint:** The Raw virtual view returns three identities **before caching**. If it does not, correct the data source, extraction, view mapping, or published namespace before continuing.

## Lab 3 — Configure and initialize persistent cache (15 minutes)

1. In **Directory Namespace**, locate the cache configuration for the Raw branch. Confirm that the selected cache root corresponds to the intended published view.
2. Configure the Raw view for persistent caching using the supported cache wizard or cache settings for your installed 7.4 patch.
3. Review the available refresh methods. For an HDAP-backed source, evaluate **HDAP Trigger** real-time capture if supported by this topology. If not available, document the actual mechanism selected rather than assuming real-time propagation.
4. Save the cache configuration and inspect its status.
5. Identify the RadiantOne **leader** before initialization.
6. Start initialization from the cache **Refresh Settings / Initialize** workflow (labels may vary). Select the supported source-view export/import option.
7. Monitor the initialization task in the server task interface and inspect any task-specific output.
8. Search `ou=raw,o=pcache-lab` again for `(objectClass=inetOrgPerson)`.

**Checkpoint:** The initialization task completes successfully, the cache reports an initialized/healthy state, and the Raw search returns three users.

**Important:** Three search results alone do **not** prove the cache is being used. Correlate results with cache configuration and initialization status.

## Lab 4 — Find the right logs (10 minutes)

RadiantOne log locations can be customized. Locate the actual files on your system rather than assuming every component is present.

| Evidence | Primary use |
| --- | --- |
| `vds_server/logs/vds_server.log` | LDAP request handling, server errors, source/backend issues |
| `vds_server/logs/sync_engine/sync_engine.log` | Sync Engine and HDAP Trigger processing, **when configured** |
| `vds_server/logs/periodiccache.log` | Periodic refresh activity, **if periodic refresh is used** |
| Connector or sync-agent logs | Only if the configured topology uses those components |
| Initialization task details | LDIF generation/import, initialization progress and failure context |
| Cache status / refresh history | Whether the cache is initialized, refreshing, or unhealthy |

### Windows PowerShell: discover and inspect logs

Replace the example path with the actual installation root:

```powershell
$RLI_HOME = 'C:\radiantone\vds'
$LogRoot = Join-Path $RLI_HOME 'vds_server\logs'

# Find candidate logs (read-only)
Get-ChildItem $LogRoot -Recurse -File |
    Where-Object { $_.Name -match 'sync|cache|vds_server|connector' } |
    Select-Object FullName, Length, LastWriteTime

# Find recent errors (read-only; may be slow on large log trees)
Get-ChildItem $LogRoot -Recurse -File -Filter '*.log' |
    Select-String -Pattern 'ERROR','WARN','cache','trigger','initialize' |
    Select-Object Path, LineNumber, Line

# Follow the server log (adjust the path if necessary)
Get-Content (Join-Path $LogRoot 'vds_server.log') -Tail 50 -Wait
```

If present, follow the Sync Engine log in a separate shell:

```powershell
Get-Content (Join-Path $LogRoot 'sync_engine\sync_engine.log') -Tail 50 -Wait
```

### Targeted DEBUG exercise

1. Record the existing Sync Engine logging level.
2. If the configured refresh topology uses the Sync Engine, temporarily enable its DEBUG logging in the Main Control Panel log settings.
3. Make one synthetic source change in Lab 5.
4. Record the timestamp, component, change type, and outcome; **redact all environment-specific data**.
5. Restore the original logging level.

**Do not publish raw log output.** Logs can contain identifiers, network details, and sensitive attribute values.

## Lab 5 — Test cache propagation (10 minutes)

Perform each action against the **source** `ou=people,o=lab-source`, then query the **Raw** branch `ou=raw,o=pcache-lab`.

| Test | Source action | Expected Raw result |
| --- | --- | --- |
| Update | Change Alice's `mail` to `alice.updated@example.com` | New value appears after the configured refresh |
| Add | Add `uid=david.miller` with `inetOrgPerson`, `cn`, `sn`, `uid`, and `mail` | User count becomes 4 |
| Delete | Delete the synthetic `uid=david.miller` entry | User count returns to 3 |

For the add test, use this synthetic LDIF:

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

Record for each operation: source modification timestamp, first observed Raw result, elapsed time, relevant cache status, and whether the expected log component recorded activity. **Do not assert that updates are instantaneous:** the result depends on the configured refresh method and system health.

## Troubleshooting decision table

| Symptom | Check first | Next evidence |
| --- | --- | --- |
| Raw view empty before caching | Source records, LDAP connection, ORX extraction, DVX mapping | `vds_server.log` |
| Cache initialization fails | Task status, source-view export, schema, disk space, leader | Initialization task details and server log |
| Source update not reflected in cache | Actual refresh method and pipeline status | `sync_engine.log` **or** `periodiccache.log`, as applicable |
| Add succeeds but delete remains stale | Change-capture and delete handling, refresh backlog | Sync/refresh logs and cache status |
| Cached search fails | Search base, permissions, cache health, server health | `vds_server.log` |
| No Sync Engine log activity | Confirm that the chosen topology actually uses Sync Engine | Refresh configuration; do not assume a missing log means failure |

**Diagnostic discipline:** Establish the exact failed operation and timestamp; validate the source independently; check cache configuration/status; then inspect only the relevant logs. Do not reinitialize a cache as the first troubleshooting action.

## Evidence worksheet

Use only synthetic or sanitized observations:

```text
RadiantOne version / patch: __________________
Platform: _________________________________
Refresh mechanism: ________________________
Source count before cache: _________________
Cache initialized successfully? ____________
Raw count after initialization: _____________
Update propagation time: __________________
Count after add: ___________________________
Count after delete: ________________________
Relevant log component: ___________________
Observed failure / resolution: _____________
```

## Cleanup and rollback

1. Restore any temporarily changed logging levels.
2. Remove the added test user if still present and restore Alice's original email address.
3. If the environment is dedicated to this lab, remove the lab cache and virtual view using the supported administrative workflow, then remove the isolated `o=pcache-lab` and `o=lab-source` namespaces **only after confirming they contain no other data**.
4. Do not use forced cache deletion or filesystem-level removal without vendor-approved procedures and a verified backup.

## Completion checklist

- [ ] Source HDAP contains three synthetic identities.
- [ ] Raw virtual view returns three identities before caching.
- [ ] Cache is explicitly configured and initialized.
- [ ] Cache status confirms successful initialization.
- [ ] Update, add, and delete behavior has been observed.
- [ ] Relevant log files and initialization evidence have been located.
- [ ] Logging levels have been restored.
- [ ] Any published screenshots or log snippets have been sanitized.

## Further reading

- [RadiantOne 7.4 documentation portal](https://developer.radiantlogic.com/idm/v7.4/)
- [RadiantOne 7.4 cache concepts](https://developer.radiantlogic.com/idm/v7.4/sys-admin-guide/cache/)
- [RadiantOne 7.4 persistent cache management](https://developer.radiantlogic.com/idm/v7.4/operations-guide/04-persistent-cache-management/)

> Validate documentation links and version-specific procedures against your installed 7.4 patch. This lab is intended for learning and should not be interpreted as a production change procedure.

---

**Repository:** [radiantone-how-to](https://github.com/castillorm/radiantone-how-to)  
**Suggested location:** `guides/radiantone-persistent-cache-hdap-lab.md`  
**License:** See the repository's root `LICENSE` file.
