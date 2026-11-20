Watch me explain this lab: [LoomLink](https://www.loom.com/share/f6143e634aec479dba9c9790ec38575e)

# azure-sql-key-vault-managed-identity

Replaced a self-managed database VM with **Azure SQL Database** (PaaS), stored the database password in **Azure Key Vault**, and gave the web server a **Managed Identity** so it can read that password without one ever being stored on the server. Then I proved the whole chain from inside the VM: it got a token with no credentials, pulled the secret, logged in to Azure SQL and wrote a row, and was **denied (HTTP 403)** when it tried to write to Key Vault, which shows least privilege is enforced, not just configured.

# Azure Cloud Series, Lab 03: Modernizing to PaaS & Securing Secrets

**Azure SQL Database · Azure Key Vault · Managed Identity · Azure RBAC · Azure Monitor · Ubuntu · sqlcmd · Azure Portal**

[Lab 02](https://github.com/richardkmnguyen/azure-secure-two-tier-network) put a database on its own locked down VM. That works, but it means I own the OS, the patching, the backups and the availability, and the app still needs a password from somewhere. This lab fixes both: the database becomes a managed service, and the password moves into a vault that the web server reaches with its own identity. This is the third lab of a six week Azure series that ends in a capstone project.

| | |
|---|---|
| **Platform** | Azure Portal and PowerShell (built-in OpenSSH client) on a local Windows PC, against my personal Azure subscription. SQL and Key Vault in Central US, `vm-web-01` in East US from Lab 02 |
| **Time** | About 2 hours, including the end-to-end test from the VM |

---

## What this lab is for

The business problem: a web app talks to a database that runs on a VM someone has to patch, back up and keep alive. The app's connection string, with the database password in it, sits in a config file on the web server, where it can leak through a misconfigured backup, a copied file, or a commit to source control.

Four ideas fix that:

- **PaaS, not IaaS.** Azure SQL Database is SQL Server without the server. Microsoft handles the OS, patching, backups and availability; I only manage the database.
- **One home for secrets.** Key Vault stores the password in one audited place instead of in code or config files.
- **The server has its own identity.** A system-assigned Managed Identity lets `vm-web-01` prove who it is to Azure without a username or password. Azure manages the credential behind it.
- **Least privilege through RBAC.** I get **Key Vault Administrator** to set the vault up; the VM gets **Key Vault Secrets User**, which can read secret values and nothing else.

| Identity | Role on the vault | Can do |
|---|---|---|
| Me (my Azure account) | Key Vault Administrator | Create, read, update and delete secrets |
| `vm-web-01` (Managed Identity) | Key Vault Secrets User | Read secret values only |

---

## Architecture

```
vm-web-01 (Ubuntu, from Lab 02)
        │
        │  1. Asks the Instance Metadata Service (169.254.169.254)
        │     for a token as its Managed Identity. No password.
        │
        │  2. Uses the token to read SqlAdminPassword
        ▼
Azure subscription (personal)
        │
        ├── rg-lab02-richard (East US)
        │       ├── vm-web-01   System-assigned Managed Identity
        │       └── vm-db-01    DELETED, replaced by Azure SQL
        │
        └── rg-lab03-richardnguyen (Central US)
                │
                ├── kv-lab03-richardnguyen   Key Vault, Standard, RBAC
                │       └── Secret: SqlAdminPassword
                │             ▲
                │             │  Key Vault Secrets User → vm-web-01
                │
                └── sql-server-richardnguyen  (logical SQL server)
                        └── sqldb-app   Basic DTU tier (5 DTUs)
                              ▲
                              │  3. vm-web-01 logs in with the secret
```

<img width="1800" height="784" alt="lab03-architecture" src="https://github.com/user-attachments/assets/dd83da3f-5962-4fea-ba5e-972ae4da7435" />

| Component | Role |
|---|---|
| **`rg-lab03-richardnguyen`** | Resource group for everything new in this lab, so cleanup is one delete |
| **`sql-server-richardnguyen`** | Logical SQL server: holds the admin login, firewall rules and region |
| **`sqldb-app`** | Azure SQL Database on the Basic DTU tier, about $5/month |
| **`kv-lab03-richardnguyen`** | Key Vault, Standard tier, Azure RBAC permission model |
| **`SqlAdminPassword`** | The SQL admin password, stored as a secret |
| **`vm-web-01`** | Lab 02's web server, now with a system-assigned Managed Identity |

---

## What I built

### 1. Decommissioned the database VM
Deleted `vm-db-01`, its OS disk and its network interface from `rg-lab02-richard`, and left `vm-web-01` alone. A stopped VM still bills for its disk, so deleting it is the right move once it's replaced. The Activity log is the audit trail: each delete is listed with my account and a timestamp.

<img width="1817" height="748" alt="ModernizeSQLDecomAudit" src="https://github.com/user-attachments/assets/15e34b55-1560-48c2-87f2-f7247aae3f6a" />
<img width="1454" height="667" alt="ModernizeSQLDecomAfter" src="https://github.com/user-attachments/assets/daf00464-5cd7-4d9d-9063-420dcfc550f4" />

### 2. Hit a region wall and moved to Central US
Creating the SQL server in East US failed: *"Your subscription does not have access to create a server in the selected region."* Microsoft restricts new SQL capacity in some regions for some subscription types. I deployed the SQL server and Key Vault to **Central US** instead and kept them together. `vm-web-01` stayed in East US, which is fine: it reaches SQL over Azure's network, and the extra latency doesn't matter here.

<img width="560" height="481" alt="ModernizeSQLRegionError" src="https://github.com/user-attachments/assets/187ef13d-5ac7-47ec-822b-38e34b3a7463" />

### 3. Deployed Azure SQL Database without the cost traps
Created `sqldb-app` on a new logical server, `sql-server-richardnguyen`, with SQL authentication (`sqladmin`). The wizard has two settings that quietly get expensive:

- **Free offer banner.** While it's applied, the DTU tiers are hidden and you're locked to Serverless. I removed it first.
- **Workload environment.** It defaults to Production, which picks Hyperscale at $320+/month. I set it to **Development**, then chose **DTU-based → Basic** (5 DTUs, about $4.99/month).

<img width="851" height="509" alt="ModernizeSQLBasicTier" src="https://github.com/user-attachments/assets/96480297-d31c-40b2-9ea3-5e27196e837a" />

Networking: public endpoint, **Allow Azure services** on (so `vm-web-01` can connect), a firewall rule for my own IP, and minimum TLS 1.2. Microsoft Defender for SQL skipped for the lab ($15/server/month), and backups on locally-redundant storage.

<img width="1103" height="627" alt="ModernizeSQLNetworking1" src="https://github.com/user-attachments/assets/ba21c86c-566c-4e64-8114-ccb31f5a6b5f" />
<img width="1262" height="419" alt="ModernizeSQLNetworking2" src="https://github.com/user-attachments/assets/e576ca90-be47-4b81-8574-e65aaabf28b3" />
<img width="1435" height="274" alt="ModernizeSQLOverview" src="https://github.com/user-attachments/assets/ff1f8b8d-6915-4782-88a3-9215981f5c88" />

### 4. Deployed Key Vault with RBAC
Created `kv-lab03-richardnguyen` on the Standard tier with the **Azure role-based access control** permission model, and left purge protection off so the lab can be fully cleaned up. With RBAC, nobody can read or write secrets by default, not even the person who created the vault, so I assigned myself **Key Vault Administrator** before adding anything.

<img width="1885" height="712" alt="ModernizeSQLKeyVaultInputs" src="https://github.com/user-attachments/assets/5dcaa267-6813-464e-8b28-e9dc961ca3a6" />

### 5. Stored the password as a secret
Added the SQL admin password as `SqlAdminPassword`. From here on, Key Vault is the only place it lives.

<img width="1034" height="139" alt="ModernizeSQLKVSecrets" src="https://github.com/user-attachments/assets/440df168-1be7-402c-90e1-42ac012dbb8c" />

### 6. Gave the web server an identity and the minimum access
Turned on a **system-assigned Managed Identity** for `vm-web-01`, which registered it in Entra ID with its own Object ID. Then, on the vault, I assigned that identity **Key Vault Secrets User**.

<img width="391" height="257" alt="ModernizeSQLVMIdentity" src="https://github.com/user-attachments/assets/ae85413d-f1fd-4154-98a3-b39d385df688" />
<img width="1607" height="529" alt="ModernizeSQLKVRoleAssignments" src="https://github.com/user-attachments/assets/ad29c0c4-635b-48b8-979f-c095442d36a9" />

### 7. Proved it end to end from inside the VM
The original guide I planned to follow stopped at granting permissions. A permission you never use isn't proven, so I SSH'd into `vm-web-01` and walked the whole chain.

**Got a token with no credentials.** The Instance Metadata Service is only reachable from inside an Azure VM, and it hands out tokens for that VM's identity:

```bash
TOKEN=$(curl -s -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fvault.azure.net" \
  | jq -r .access_token)
```

**Read the secret straight into an environment variable.** `sqlcmd` reads `SQLCMDPASSWORD` on its own, so the password is never typed, printed, written to disk, or visible in the process list:

```bash
export SQLCMDPASSWORD=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://$KV.vault.azure.net/secrets/SqlAdminPassword?api-version=7.4" | jq -r .value)
echo "Retrieved secret, length: ${#SQLCMDPASSWORD} characters"
```

<img width="1004" height="94" alt="ModernizeSQLVMTokenSecret" src="https://github.com/user-attachments/assets/657e676b-38c6-4c55-9e45-0e6864c84f7f" />

**Logged in to Azure SQL and wrote a row:**

```bash
sqlcmd -S "$SQL.database.windows.net" -d sqldb-app -U sqladmin -Q "
CREATE TABLE dbo.LabCheck (Id INT IDENTITY PRIMARY KEY, Source NVARCHAR(50), CheckedAt DATETIME2 DEFAULT SYSUTCDATETIME());
INSERT INTO dbo.LabCheck (Source) VALUES ('vm-web-01 via Key Vault');
SELECT * FROM dbo.LabCheck;"
```

```
Id          Source                        CheckedAt
----------- ----------------------------- ---------------------------
          1 vm-web-01 via Key Vault       2026-11-16 05:31:39.8928136
```

<img width="709" height="217" alt="ModernizeSQLVMSQLQuery" src="https://github.com/user-attachments/assets/45017e24-dfa1-48de-90d1-1afadd01c7a8" />

**Tried to write a secret, which should fail:**

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" -X PUT \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"value":"test"}' "https://$KV.vault.azure.net/secrets/ShouldFail?api-version=7.4"
# HTTP 403
```

<img width="1163" height="41" alt="ModernizeSQLHTTP403" src="https://github.com/user-attachments/assets/eabe8dbe-8dac-48bb-8ae2-5a7433011e14" />

> Same token, two answers: reading the secret works, writing one is forbidden. That's least privilege doing its job.

### 8. Confirmed it in Azure Monitor
On `sqldb-app` → Metrics, **DTU percentage (Max)** showed the database idle at 0% after creation, then a spike to about 1% at 9:31 PM, which lines up with the `LabCheck` timestamp. Basic gives 5 DTUs, so 1% is a handful of small queries. In production I'd put an alert rule on this metric.

<img width="1661" height="797" alt="ModernizeSQLMonitorDTULastHour" src="https://github.com/user-attachments/assets/3e598721-9f0a-4821-a214-faed8b43a0a3" />
<img width="1902" height="395" alt="ModernizeSQLFinal" src="https://github.com/user-attachments/assets/a95457ec-5446-43bb-b889-7d5cc2eb9337" />

---

## Changes from the original guide I planned to follow

| Original guide | What I did instead | Why |
|---|---|---|
| Deploy in East US | Central US for SQL and Key Vault | My subscription can't create SQL servers in East US |
| Stops after granting the VM access | Added an end-to-end test from `vm-web-01`: token, secret, SQL login, and a write that should fail | A permission you never exercise isn't proven, and the 403 shows the restriction works |
| Completion checklist asks for a Key Vault access policy with Get and List | Verified the **Key Vault Secrets User** role assignment instead | The lab uses the RBAC permission model, so access policies don't apply |
| — | Password passed to `sqlcmd` through `SQLCMDPASSWORD`, never on the command line | Keeps the password out of shell history and the process list |

---

## Common issues

| Symptom | Cause | Fix |
|---|---|---|
| "Your subscription does not have access to create a server in the selected region" | SQL capacity restricted in that region for the subscription | Pick another region (I used Central US) and keep SQL and Key Vault together |
| No DTU-based option in Configure database | The free offer is still applied | Remove the "Free offer applied" banner, then reopen Configure database |
| "The operation is not allowed by RBAC" on the Secrets page | No role for your own account on the vault | Assign yourself Key Vault Administrator; wait 1–2 minutes |
| `Unable to handle repository shortcut` adding Microsoft's 24.04 repo | That list file's format isn't readable by `add-apt-repository` | Use the 22.04 repo; `sqlcmd` is a standalone binary and runs fine on 24.04 |
| Pasted commands half-run, with mangled lines like `t-get install -y sqlcmdcurl` | `apt-get` reads the rest of a multi-line paste as input while it runs | Paste one command at a time and wait for the prompt |
| `ssh: connect to host … port 22: Connection timed out` | I'd started `vm-test-01` instead of `vm-web-01` | Check the right VM is Running and its public IP matches your SSH config |
| Long dotted line on the Metrics chart | No data before the database existed | Set the time range to Last hour |

---

## Key takeaways

1. **Managed services remove whole categories of work.** No OS to patch, no backups to schedule, no VM to keep alive. The database is just a database.
2. **The best place for a password is nowhere on the server.** The VM reads it from Key Vault at the moment it needs it, using an identity Azure manages.
3. **RBAC is deny by default, even for the vault's creator.** Access exists only where a role grants it, and the roles can be split by job: admin for me, read-only for the app.
4. **Prove access in both directions.** The SQL login proves the VM *can* do what it should. The 403 proves it *can't* do what it shouldn't.
5. **Cloud capacity is a real constraint.** A region being unavailable isn't a broken lab; picking a sensible fallback and keeping dependent resources together is part of the job.

If I did this again, I'd switch the SQL server to **Microsoft Entra authentication** and let the Managed Identity log in to SQL directly with a token. That removes the SQL password entirely, so there's no secret left to store or rotate. I'd also check region availability before starting, and deploy the lab with Terraform after the portal run.

---

## Security notes

This is a lab configuration. Some of it is solid and some of it would not be acceptable in production:

**What the lab does well**
- **No credentials on the web server.** The VM authenticates with its Managed Identity and reads the password only when needed.
- **Least privilege on the vault.** The app's identity can read secrets and nothing else, and I tested the denial.
- **RBAC instead of access policies,** so vault permissions use the same model and audit trail as every other Azure resource.
- **TLS 1.2 minimum** on the SQL server, and Key Vault reached only over HTTPS.
- **Personal subscription, separate resource groups,** deleted after the lab.

**What I would change for production**
- **Entra-only authentication on Azure SQL,** with the Managed Identity as a database user, so there's no SQL password at all.
- **Private endpoints for SQL and Key Vault,** with public network access disabled, instead of a public endpoint and "All networks".
- **Narrow the SQL firewall.** "Allow Azure services" lets in traffic from any Azure subscription, not just mine.
- **Turn on purge protection and secret expiry** so the vault can't be permanently deleted by mistake and passwords get rotated.
- **Enable Microsoft Defender for SQL** and send SQL and Key Vault diagnostic logs to Log Analytics.
- **Alert on DTU percentage** and use zone- or geo-redundant backups for anything that matters.

---

## Verification

| Check | How I verified it |
|---|---|
| Old DB VM gone | `vm-db-01`, its disk and NIC deleted; Activity log shows the deletes |
| Web tier intact | `vm-web-01` still running in `rg-lab02-richard` |
| SQL database live | `sqldb-app` Overview: Online, Basic, Central US |
| Vault deployed | `kv-lab03-richardnguyen`, Standard, RBAC permission model |
| Secret stored | `SqlAdminPassword` listed as Enabled |
| VM has an identity | System-assigned Managed Identity On, Object ID present |
| Least-privilege role | Key Vault IAM: `vm-web-01` holds Key Vault Secrets User |
| VM can read the secret | Token from IMDS, secret length printed from inside the VM |
| VM can use the database | `sqlcmd` login as `sqladmin`, `LabCheck` row written and read back |
| VM can't write secrets | `PUT` to the vault returned HTTP 403 |
| Database is monitored | DTU percentage chart shows the spike from the test |

---

## Teardown

Deleting the Lab 03 resource group removes the SQL server, the database and the Key Vault in one step.

```
Resource groups → rg-lab03-richardnguyen → Delete resource group → type the name → Delete
```

Or with the Azure CLI:

```powershell
az group delete --name rg-lab03-richardnguyen --yes
az group show --name rg-lab03-richardnguyen   # expect ResourceGroupNotFound
```

Key Vault goes into soft-delete for 90 days. Because purge protection was off, I can purge it to free up the name right away:

```powershell
az keyvault purge --name kv-lab03-richardnguyen
```

`rg-lab02-richard` stays for now, since `vm-web-01` may be needed in Lab 04.

---

## Next steps

- Switch Azure SQL to **Entra-only authentication** and connect with the Managed Identity's token, no password
- Put SQL and Key Vault behind **private endpoints** inside the Lab 02 VNet
- Rebuild the lab with **Terraform** (`azurerm_mssql_server`, `azurerm_mssql_database`, `azurerm_key_vault`, `azurerm_role_assignment`)
- Add an **Azure Monitor alert** on DTU percentage
- Carry the pattern into the series capstone

---

**Status: complete.** Deleted the database VM → deployed Azure SQL Database (Basic) in Central US after East US was unavailable → deployed Key Vault with RBAC and stored the SQL password as a secret → gave `vm-web-01` a Managed Identity with Key Vault Secrets User → from the VM, got a token, read the secret, logged in to SQL and wrote a row, and was denied a secret write (403) → confirmed activity in Azure Monitor → resource group deleted.
