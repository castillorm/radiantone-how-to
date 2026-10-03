# Querying RadiantOne ADAP with PowerShell

### Use PowerShell to authenticate to the RadiantOne REST interface and search a RadiantOne namespace

## Objective

This guide shows how to use PowerShell to connect to the RadiantOne REST interface, known as **ADAP (Adaptive Directory Access Protocol)**, authenticate with a bind identity, and perform read-only LDAP searches against a RadiantOne namespace.

The examples cover:

- Building a Basic Authentication header from a bind DN and password.
- Testing bind credentials before performing a search.
- Performing a subtree search with a base DN and LDAP filter.
- Requesting selected attributes.
- Packaging the search logic into a reusable PowerShell function.
- Troubleshooting common authentication, authorization, and query errors.

> This guide intentionally focuses on **read-only search operations**. Add, modify, delete, and other write operations should be tested separately and protected with appropriately scoped access controls.

---

## Applies to

- RadiantOne 7.4 REST/ADAP interface
- Windows PowerShell 5.1 or PowerShell 7+
- A RadiantOne namespace that is reachable from the PowerShell client

The REST endpoint, port, certificate configuration, and available naming contexts vary by environment. Use the ADAP endpoint configured for your RadiantOne deployment.

---

## How ADAP authentication works

ADAP exposes LDAP operations through an HTTP/HTTPS REST interface. For Basic Authentication, the request contains an `Authorization` header whose value is built from the RadiantOne bind DN and password:

```text
Authorization: Basic <base64(bindDN:password)>
```

The Base64 value is an encoding of the credentials; it is **not encryption**. Use HTTPS for real environments and ensure the client trusts the certificate presented by the ADAP endpoint.

A typical request flow looks like this:

```text
PowerShell
    |
    | HTTPS + Authorization header
    v
RadiantOne ADAP endpoint
    |
    | LDAP bind / LDAP search
    v
RadiantOne virtual namespace
    |
    v
Authorized search results
```

---

## Prerequisites

Before running the examples, confirm the following.

### 1. ADAP endpoint connectivity

You need the URL of the RadiantOne ADAP endpoint and network access to it.

Example:

```text
https://radiant.example.com:<port>/adap
```

Do not assume that the port used in another environment applies to yours. Confirm the configured REST/ADAP endpoint for the RadiantOne instance.

A basic TCP connectivity test from PowerShell is:

```powershell
Test-NetConnection -ComputerName "radiant.example.com" -Port <port>
```

Successful TCP connectivity does not prove that authentication or namespace access is configured correctly, but it is a useful first check.

### 2. Valid RadiantOne bind identity

Use a dedicated service or application identity where practical. You need:

- The full bind DN.
- The password for that identity.
- An identity that can successfully authenticate to RadiantOne.

Example bind DN:

```text
uid=svc-adap-query,ou=serviceaccounts,o=companydirectory
```

Avoid using `cn=directory manager` for routine application access. A dedicated identity makes permissions easier to scope, audit, and rotate.

### 3. Appropriate Access Control Instructions (ACIs)

Successful authentication and successful authorization are different checks.

RadiantOne uses the identity from the bind operation together with configured **Access Control Instructions (ACIs)** to determine what the caller may access in the virtual namespace.

For the read-only searches in this guide, the bind identity should have at least:

- **Search** permission on the target namespace/subtree.
- **Read** permission on the entries and attributes that should be returned.

RadiantOne evaluates Search permission for the search operation and then applies Read permission to entries and attributes returned by the search. As a result, an account may authenticate successfully but still be unable to retrieve the expected entries or attributes.

Apply least privilege. Scope the ACI to only the namespace, subtree, and attributes the service actually needs.

### 4. A valid base DN

You need the naming context or subtree that will be the starting point for the search.

Examples:

```text
o=companydirectory
```

or:

```text
ou=people,o=companydirectory
```

If you do not know the available naming contexts, ADAP can query the Root DSE. An example is included later in this guide.

### 5. A valid LDAP search filter

Examples:

```text
(objectClass=*)
```

```text
(uid=jdoe)
```

```text
(&(objectClass=person)(sn=Smith))
```

LDAP filters can contain characters that must be URL-encoded when they are placed in an HTTP query string. The PowerShell examples below use .NET URI encoding rather than requiring you to manually encode filters.

---

## Example values used in this guide

Replace all values with settings appropriate for your environment.

```powershell
$AdapEndpoint = "https://radiant.example.com:<port>/adap"
$BindDn       = "uid=svc-adap-query,ou=serviceaccounts,o=companydirectory"
$BindPassword = "REPLACE-ME"
$BaseDn       = "ou=people,o=companydirectory"
$SearchFilter = "(objectClass=person)"
```

> Do not commit real passwords, API secrets, private certificates, or customer-specific credentials to GitHub. The literal password variable above is shown only to make the authentication mechanism easy to understand.

---

# Step 1 — Build the Authorization header

The following code converts the bind DN and password into the Basic Authentication header expected by ADAP.

```powershell
$BindDn       = "uid=svc-adap-query,ou=serviceaccounts,o=companydirectory"
$BindPassword = "REPLACE-ME"

$CredentialString = "{0}:{1}" -f $BindDn, $BindPassword
$CredentialBytes  = [System.Text.Encoding]::UTF8.GetBytes($CredentialString)
$EncodedCredential = [Convert]::ToBase64String($CredentialBytes)

$Headers = @{
    Authorization = "Basic $EncodedCredential"
    Accept        = "application/json"
}
```

You can display the header while working with synthetic credentials:

```powershell
$Headers
```

Do **not** display or log the header when it contains a real password. Anyone who obtains the Base64 value can decode the underlying `bindDN:password` string.

---

# Step 2 — Test the bind credentials

RadiantOne ADAP provides a simple-bind operation that is useful for validating credentials before troubleshooting a larger search request.

```powershell
$AdapEndpoint = "https://radiant.example.com:<port>/adap"

$BindUri = "${AdapEndpoint}?bind=simpleBind"

try {
    $BindResult = Invoke-RestMethod `
        -Uri $BindUri `
        -Method Get `
        -Headers $Headers `
        -ErrorAction Stop

    Write-Host "RadiantOne bind succeeded." -ForegroundColor Green
    $BindResult | ConvertTo-Json -Depth 10
}
catch {
    Write-Host "RadiantOne bind failed." -ForegroundColor Red
    Write-Host $_.Exception.Message
}
```

A successful simple-bind request should return an HTTP 200 result. Invalid credentials may result in HTTP `401 Unauthorized`.

### What this test proves

A successful result proves that:

- The ADAP endpoint is reachable through HTTP/HTTPS.
- The authorization header was accepted.
- The supplied bind credentials authenticated successfully.

It does **not** prove that the account has Search or Read access to a particular namespace. Test authorization with an actual namespace search next.

---

# Step 3 — Perform a basic namespace search

The basic ADAP search URL follows this pattern:

```text
<ADAP endpoint>/<baseDN>?filter=<LDAP filter>&scope=<scope>
```

The following PowerShell example searches all entries beneath the supplied base DN.

```powershell
$AdapEndpoint = "https://radiant.example.com:<port>/adap"
$BaseDn       = "ou=people,o=companydirectory"
$SearchFilter = "(objectClass=person)"
$Scope        = "sub"

$EncodedBaseDn = [System.Uri]::EscapeDataString($BaseDn)
$EncodedFilter = [System.Uri]::EscapeDataString($SearchFilter)

$SearchUri = "${AdapEndpoint}/${EncodedBaseDn}?filter=${EncodedFilter}&scope=${Scope}"

try {
    $SearchResult = Invoke-RestMethod `
        -Uri $SearchUri `
        -Method Get `
        -Headers $Headers `
        -ErrorAction Stop

    $SearchResult | ConvertTo-Json -Depth 10
}
catch {
    Write-Host "RadiantOne search failed." -ForegroundColor Red
    Write-Host $_.Exception.Message
}
```

The supported scope values are:

| Scope | Meaning |
| --- | --- |
| `base` | Search only the base DN entry. |
| `one` | Search entries one level below the base DN. |
| `sub` | Search the entire subtree beneath the base DN. |

For most directory queries against a people or account subtree, `sub` is the most common starting point. Use the smallest scope that satisfies the use case.

---

# Step 4 — Search for a specific identity

Change the LDAP filter to narrow the query.

```powershell
$BaseDn       = "ou=people,o=companydirectory"
$SearchFilter = "(uid=jdoe)"
$Scope        = "sub"

$EncodedBaseDn = [System.Uri]::EscapeDataString($BaseDn)
$EncodedFilter = [System.Uri]::EscapeDataString($SearchFilter)

$SearchUri = "${AdapEndpoint}/${EncodedBaseDn}?filter=${EncodedFilter}&scope=${Scope}"

$SearchResult = Invoke-RestMethod `
    -Uri $SearchUri `
    -Method Get `
    -Headers $Headers `
    -ErrorAction Stop

$SearchResult | ConvertTo-Json -Depth 10
```

Other useful filters include:

```text
(mail=jdoe@example.com)
```

```text
(sAMAccountName=jdoe)
```

```text
(&(objectClass=person)(employeeType=Employee))
```

The attribute names available for filtering depend on the namespace and view being queried.

---

# Step 5 — Request only selected attributes

By default, a search can return all attributes available to the caller. For automation, it is usually better to request only what the script needs.

Example:

```powershell
$BaseDn       = "ou=people,o=companydirectory"
$SearchFilter = "(uid=jdoe)"
$Scope        = "sub"
$Attributes   = "cn,uid,mail"

$EncodedBaseDn    = [System.Uri]::EscapeDataString($BaseDn)
$EncodedFilter    = [System.Uri]::EscapeDataString($SearchFilter)
$EncodedAttributes = [System.Uri]::EscapeDataString($Attributes)

$SearchUri = "${AdapEndpoint}/${EncodedBaseDn}?filter=${EncodedFilter}&scope=${Scope}&attributes=${EncodedAttributes}"

$SearchResult = Invoke-RestMethod `
    -Uri $SearchUri `
    -Method Get `
    -Headers $Headers `
    -ErrorAction Stop

$SearchResult | ConvertTo-Json -Depth 10
```

Requesting only required attributes reduces unnecessary data exposure and makes responses easier to process.

---

# Step 6 — Use a reusable PowerShell search function

Once the basic tests work, the following function provides a reusable wrapper for read-only ADAP searches.

```powershell
function Invoke-RadiantAdapSearch {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory)]
        [string]$AdapEndpoint,

        [Parameter(Mandatory)]
        [string]$BindDn,

        [Parameter(Mandatory)]
        [string]$BindPassword,

        [Parameter(Mandatory)]
        [string]$BaseDn,

        [Parameter(Mandatory)]
        [string]$Filter,

        [ValidateSet("base", "one", "sub")]
        [string]$Scope = "sub",

        [string[]]$Attributes
    )

    $CredentialString = "{0}:{1}" -f $BindDn, $BindPassword
    $CredentialBytes  = [System.Text.Encoding]::UTF8.GetBytes($CredentialString)
    $EncodedCredential = [Convert]::ToBase64String($CredentialBytes)

    $Headers = @{
        Authorization = "Basic $EncodedCredential"
        Accept        = "application/json"
    }

    $EncodedBaseDn = [System.Uri]::EscapeDataString($BaseDn)
    $EncodedFilter = [System.Uri]::EscapeDataString($Filter)

    $Parameters = @(
        "filter=$EncodedFilter"
        "scope=$Scope"
    )

    if ($Attributes -and $Attributes.Count -gt 0) {
        $AttributeList = $Attributes -join ","
        $EncodedAttributes = [System.Uri]::EscapeDataString($AttributeList)
        $Parameters += "attributes=$EncodedAttributes"
    }

    $QueryString = $Parameters -join "&"
    $SearchUri = "${AdapEndpoint}/${EncodedBaseDn}?${QueryString}"

    try {
        Invoke-RestMethod `
            -Uri $SearchUri `
            -Method Get `
            -Headers $Headers `
            -ErrorAction Stop
    }
    catch {
        $StatusCode = $null

        if ($_.Exception.Response -and $_.Exception.Response.StatusCode) {
            $StatusCode = [int]$_.Exception.Response.StatusCode
        }

        throw "ADAP search failed. HTTP status: $StatusCode. Error: $($_.Exception.Message)"
    }
}
```

Example usage:

```powershell
$result = Invoke-RadiantAdapSearch `
    -AdapEndpoint "https://radiant.example.com:<port>/adap" `
    -BindDn "uid=svc-adap-query,ou=serviceaccounts,o=companydirectory" `
    -BindPassword "REPLACE-ME" `
    -BaseDn "ou=people,o=companydirectory" `
    -Filter "(uid=jdoe)" `
    -Scope sub `
    -Attributes @("cn", "uid", "mail")

$result | ConvertTo-Json -Depth 10
```

---

# Step 7 — Avoid storing the password in the script

For interactive testing, PowerShell's credential prompt is preferable to placing a clear-text password in the script.

```powershell
$BindDn = "uid=svc-adap-query,ou=serviceaccounts,o=companydirectory"

$Credential = Get-Credential -UserName $BindDn -Message "Enter the RadiantOne bind password"
$BindPassword = $Credential.GetNetworkCredential().Password
```

You can then pass `$BindDn` and `$BindPassword` to the examples above.

For unattended automation, use an approved secrets-management mechanism rather than storing the password in source code, shell history, configuration committed to Git, or CI/CD variables that are exposed to users who do not require access.

---

# Optional — Display available naming contexts

If the account is authorized to read the Root DSE, ADAP can return the naming contexts exposed by RadiantOne.

```powershell
$RootDseUri = "${AdapEndpoint}/rootdse"

$RootDse = Invoke-RestMethod `
    -Uri $RootDseUri `
    -Method Get `
    -Headers $Headers `
    -ErrorAction Stop

$RootDse | ConvertTo-Json -Depth 10
```

This can be useful when confirming the correct base DN before building a search.

---

## Troubleshooting

### `401 Unauthorized`

Check:

- Bind DN spelling and full DN syntax.
- Password.
- Whether the intended identity can authenticate to RadiantOne.
- Whether the Authorization header was built from the correct `bindDN:password` value.

Re-run the simple-bind test before troubleshooting the search itself.

### Bind succeeds, but the search does not return the expected entries

Check:

- Base DN.
- Search scope.
- LDAP filter.
- ACI Search permission on the requested subtree.
- ACI Read permission on the matching entries.
- ACI Read permission on the attributes being requested.

A successful bind proves authentication, not authorization.

### Some attributes are missing

Check the ACIs protecting those attributes. RadiantOne can allow the search while restricting which attributes the caller is authorized to read.

### Search returns too many entries

Narrow one or more of:

- Base DN.
- LDAP filter.
- Search scope.
- Returned attributes.

For large result sets, review ADAP size-limit and paging options rather than routinely requesting an entire namespace.

### HTTPS certificate error

Verify that:

- The endpoint hostname matches the certificate.
- The issuing CA is trusted by the PowerShell host.
- The certificate is not expired.

Do not make disabling certificate validation the normal fix. Correct the trust chain or endpoint configuration instead.

### Filter works in an LDAP client but fails through ADAP

Remember that the LDAP filter is also being transported inside a URL. Characters with special meaning in URLs must be encoded. The examples in this guide use:

```powershell
[System.Uri]::EscapeDataString($SearchFilter)
```

to handle URL encoding for the filter value.

---

## Validation checklist

Before using a script more broadly, verify all of the following:

- [ ] The ADAP endpoint is reachable from the PowerShell host.
- [ ] HTTPS certificate validation succeeds.
- [ ] The dedicated bind identity authenticates successfully.
- [ ] The identity has only the ACI permissions it requires.
- [ ] A base-level or narrowly scoped test query succeeds.
- [ ] The expected entries are returned.
- [ ] Only expected attributes are returned.
- [ ] Broad subtree searches have appropriate size/paging controls.
- [ ] Credentials are not stored in the repository or written to logs.

---

## Security considerations

For customer or production environments:

1. Prefer a dedicated, least-privileged bind identity.
2. Grant Search and Read only to the required namespace, entries, and attributes.
3. Use HTTPS.
4. Protect the bind password with an approved secrets-management solution.
5. Do not use `cn=directory manager` for routine scripts.
6. Do not commit real customer DNs, hostnames, identities, credentials, or response data to a public repository.
7. Begin with narrow filters and limited attributes before expanding a query.

---

## Next steps

Once read-only searches are working, useful extensions include:

- Adding `sizeLimit` controls.
- Working with ADAP paging.
- Converting returned JSON into PowerShell objects or CSV reports.
- Searching multiple naming contexts.
- Using OIDC/Bearer-token authentication where configured.
- Building controlled add/modify examples with a separately scoped service identity.

---

## References

- Radiant Logic, **RadiantOne 7.4 Web Services API Guide — REST/ADAP**: https://developer.radiantlogic.com/idm/v7.4/web-services-api-guide/05-rest/
- Radiant Logic, **RadiantOne 7.4 System Administration Guide — Access Controls**: https://developer.radiantlogic.com/idm/v7.4/sys-admin-guide/access-controls/

---

## Disclaimer

This guide is a personally maintained technical resource and is intended to supplement, not replace, official Radiant Logic documentation and support. Validate procedures in a non-production environment and adjust endpoint, authentication, access-control, and namespace values for your deployment.
