# Azure Data Factory -- Trusted Services Firewall Exception Retirement

> **Last updated:** April 2026

> **Effective Date:** August 1, 2026\
> **Impact:** Breaking change if relying on "Allow trusted Microsoft
> services" + Managed Identity

---

## 📧 Upcoming ADF Security Change

Microsoft has notified customers of an upcoming security change in Azure Data Factory.

Azure Data Factory will retire support for the trusted services firewall exception used with managed identities to access Azure Storage and Azure Key Vault. This change takes effect on August 1, 2026.

This document explains what this change means and how to remediate impacted architectures.

---

# 🚨 TL;DR

Today, Azure Data Factory can bypass Storage and Key Vault firewalls using the 'trusted Microsoft services' exception. After August 2026, that bypass is removed, so both identity and network access must be explicitly configured.

Azure Data Factory (ADF) is still a Microsoft service, but it will no longer be supported by the "Allow trusted Microsoft services" firewall exception for Azure Storage and Azure Key Vault.

This is a network behavior change, not an identity change. Managed Identity support remains.

👉 You now need:

- Managed Identity (who you are)
- Network access (how you get there)

👉 Both are now required. Identity without network access will fail.

👉 Previously, this bypass was enabled on the **Storage Account / Key Vault firewall**, not in ADF itself.


👉 If your pipeline works today without any network configuration, it is relying on the trusted services bypass and will fail after August 2026.

---

# 🧠 Background

Previously:

- Managed Identity + "Allow trusted Microsoft services" (on Storage/Key Vault firewall) = access

Now:

- Identity + Network must BOTH be explicitly configured
- Managed Identity behavior does not change; explicit network access is now required

---

## Important: ADF is not deployed into a VNet

- ADF is a PaaS control plane service and is not deployed into your customer VNet.
- The integration runtime (Azure IR, Managed VNet IR, SHIR, or SSIS IR) must still have a valid network path to each target data service.
- This deprecation enforces explicit network access, so identity alone is no longer sufficient.

> **Note:** Azure Synapse pipelines use the same integration runtime model and may be impacted by similar networking requirements. Validate Synapse environments separately. This document focuses on Azure Data Factory.

---

# 🧪 Pre-Validation

This feature flag simulates the post-retirement behavior by disabling the trusted services firewall bypass. It allows you to identify pipelines and linked services that will fail once the change is enforced.

Append to ADF Studio URL:

    feature.enableTrustMIToken=true

👉 Use this early in your migration planning to surface any gaps in your network configuration before the August 2026 deadline.

---

# 👥 Who May Be Affected

- Self-hosted Integration Runtime (SHIR) using managed identity with the trusted services firewall exception
- Azure-SSIS Integration Runtime using managed identity with the trusted services firewall exception
- REST linked services using managed identity
- Azure Function activities using managed identity
- Web activities accessing Storage or Key Vault with managed identity

If none of these apply, no action may be required.

---

# 🛠️ Migration Options

## 1. Private Endpoints (Recommended)

- ADF Managed VNet
- Private Endpoints to Storage / Key Vault

👉 See detailed implementation guidance below.

---

## 2. IP Allowlisting

- Add outbound IPs of SHIR / SSIS IR

👉 See detailed guidance and limitations below.

---

## 3. Service Endpoints (Customer VNet)

👉 See detailed guidance below.

### Requirements

- Requires a VNet + subnet

### Steps

1.  Enable service endpoints on subnet:
    - Microsoft.Storage
    - Microsoft.KeyVault
2.  Add subnet to firewall rules

### Pros

- Simpler than Private Endpoints
- Does not require private DNS setup
- Traffic stays on the Azure backbone (not traversing the public internet)

### Cons

- Endpoint remains publicly addressable
- Does not provide full isolation (not Zero Trust aligned)

---

## 🧭 Quick Decision Guide

| Scenario | Recommended Approach |
|----------|----------------------|
| No VNet | Managed VNet + Private Endpoints |
| Have VNet | Private Endpoints or Service Endpoints |
| Need quick fix | IP allowlisting (temporary) |
| Production workload | Private Endpoints |

---

The following sections provide more detailed guidance for specific deployment scenarios.

---

# 🔄 Existing ADF? How to Migrate to Managed VNet

Existing Azure Data Factory instances do NOT need to be redeployed to use Managed Virtual Network.

Managed VNet can be enabled after the factory is created, but this is not an automatic conversion. You must migrate workloads to use it.

## What changes

* You create a new **Managed Virtual Network Integration Runtime**
* You create **Managed Private Endpoints** to target services (Storage, Key Vault, etc.)
* You update **linked services** to use the new integration runtime
* You validate and test pipelines

## What does NOT change

* The Azure Data Factory resource itself
* Pipelines, datasets, and triggers
* Managed Identity configuration

## Migration pattern

Before:
ADF -> Azure IR -> Trusted Services -> Storage / Key Vault

After:
ADF -> Managed VNet IR -> Private Endpoint -> Storage / Key Vault

## Important

If linked services continue to use the default Azure Integration Runtime, they will still rely on the old network path and will fail after the trusted services exception is removed.

## ⚠️ Will this break existing pipelines?

Yes--if you enable Managed Virtual Network without updating your configuration.

Enabling Managed VNet does not automatically move existing workloads to the new network path. Any linked services that continue to use the default Azure Integration Runtime will still attempt to access resources using the old public path and will fail once the trusted services firewall exception is removed.

To avoid disruption:
- Create a Managed VNet integration runtime
- Configure private endpoints
- Update linked services to use the new runtime
- Test before cutover

👉 This should be treated as a migration, not a configuration toggle.

---

# 🧭 No Customer VNet? Your Options

## 🥇 Option 1 --- ADF Managed VNet + Private Endpoints (Recommended)

- No customer-managed VNet needed — Microsoft provisions and manages the VNet for you
- ADF connects to your **data sources** (Storage, SQL, ADLS, Key Vault, etc.) via Managed Private Endpoints (into the target service), not directly into your customer VNet.
- All traffic flows over the Microsoft backbone, never the public internet
- Someone must approve the private endpoint request on the data source side
- Your data sources must support private endpoints (most Azure PaaS services do)
- Best fit for production and Zero Trust network posture

Learn more:
- ADF Managed VNet + Managed Private Endpoints: https://learn.microsoft.com/en-us/azure/data-factory/managed-virtual-network-private-endpoint
- Azure Private Endpoint overview: https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview

> **What you do NOT need:** your own VNet, subnet, NSG, route table, VNet peering, or Self-Hosted IR

✔ Best long-term solution\
✔ Simplest for customers

---

## 🥈 Option 2 --- IP Allowlisting

- Allow outbound IPs of ADF IR / SHIR on each target service firewall
- Works as a short-term bridge, but expect periodic IP range updates
- Requires ongoing operational maintenance across every protected data source

Note: Azure Integration Runtime outbound IPs are not exposed in the Azure Data Factory resource. You must use Microsoft's published regional IP ranges. Observed IPs from logs or testing are not sufficient for firewall allowlisting.

Download Azure IP ranges (JSON): https://www.microsoft.com/en-us/download/details.aspx?id=56519

When using the JSON file, look for the AzureDataFactory.<region> service tag corresponding to your Data Factory region.

Learn more:
- Azure Integration Runtime IP addresses: https://learn.microsoft.com/en-us/azure/data-factory/azure-integration-runtime-ip-addresses
- Self-hosted Integration Runtime: https://learn.microsoft.com/en-us/azure/data-factory/create-self-hosted-integration-runtime

✔ Quick fix\
❌ Not scalable\
❌ Maintenance overhead

---

## ⚠️ Using Firewall Rules Instead of Private Access

This is a supported but operationally intensive approach and is not recommended for long-term production use.

If you choose not to use Managed VNet, Private Endpoints, or a customer VNet, you must explicitly allow network access using firewall rules.

- For Azure Integration Runtime:
    - Allow Microsoft-managed outbound IP ranges (region-specific)
    - These ranges may change and require ongoing maintenance
    - Azure Integration Runtime IP ranges may change without notice, which can lead to unexpected connectivity failures if firewall rules are not kept up to date
    - The IP list can be large and region-specific, increasing operational overhead
    - For automation scenarios, consider using Azure service tags (for example: AzureDataFactory.<region>) where supported to simplify firewall rule management

- For Self-Hosted Integration Runtime:
    - Allow the outbound IP of the host machine
    - This is more stable and predictable

## 🔎 How to Identify Azure Integration Runtime IP Ranges

To allow Azure Data Factory (ADF) Azure Integration Runtime traffic through firewalls, you must allow the outbound IP ranges for the region where your Data Factory is deployed.

Steps:

1. Determine your Data Factory region (for example: East US, West US 2)
2. Go to the official Microsoft documentation:
    https://learn.microsoft.com/en-us/azure/data-factory/azure-integration-runtime-ip-addresses
3. Locate the section for your region
4. Allow all listed IP ranges in:
    👉 You must allow the full set of IP ranges for your region, not just those observed during testing.
    - Azure Storage firewall
    - Azure Key Vault firewall

### Important Notes

- IP ranges are **region-specific**
- The list can be **large**
- Microsoft may update these ranges periodically
- You must keep firewall rules up to date to avoid service disruptions

👉 For more predictable and secure connectivity, consider using Private Endpoints instead of IP allowlisting.

### 🔍 Optional — Validate Using Network Security Perimeter (NSP)

Network Security Perimeter (NSP) in audit mode can be used to observe access attempts and validate which source IP addresses Azure Data Factory is actively using.

This can help:
- Confirm that firewall rules are correctly configured
- Identify active IPs during testing or troubleshooting
- Provide visibility into real traffic patterns

⚠️ Important:
- NSP shows observed traffic, not all possible outbound IPs
- It should NOT be used as the sole source for firewall allowlisting

👉 Microsoft's published Azure Integration Runtime IP ranges are the authoritative source for firewall configuration.

👉 Managed Identity alone is not sufficient. Network access must be explicitly permitted.

---

## ⚠️ Important Note

Service Endpoints REQUIRE a VNet.\
If you have no VNet, you cannot use them.

---

# ⚠️ What happens if you do nothing?

If no action is taken before August 2026:

- Pipelines will fail once the trusted services firewall exception is removed
- Failures will occur at runtime when accessing Storage, Key Vault, or other protected services
- These failures may not be immediately obvious until pipelines execute

👉 This is a breaking change and requires proactive remediation.

Learn more:
- Azure updates (track service retirement/change notices): https://azure.microsoft.com/updates/
- Azure Service Health overview: https://learn.microsoft.com/en-us/azure/service-health/overview

---

# 💥 Common Failure Symptoms

- 403 Forbidden responses from Storage, SQL, or other target services
- "Client IP address is not authorized" firewall errors
- Key Vault firewall denies (for secrets/keys lookups)
- Storage firewall denies (blob, dfs, queue, or table access)
- Pipeline timeouts during linked service tests or activity runtime

👉 Authentication succeeds, but execution fails because network access is denied.

---

# 🏆 Best Practices

- Use Private Endpoints for production
- Separate identity and network controls
- Prefer Managed VNet for simplicity
- Avoid IP allowlisting long-term
- Test early with feature flag
- Monitor for firewall failures
- Align with CAF landing zone
- Treat trusted services as deprecated today

---

# 🔑 Key Takeaway

If your solution works today without any networking configuration, it is relying on the trusted services firewall exception and will break after August 2026.

---

# ✅ Final Recommendation

- Preferred: Managed Identity + Private Endpoints (Managed VNet if no VNet exists)
- Acceptable fallback: Service Endpoints (requires VNet)
- Temporary only: IP allowlisting
- Do not use: Trusted Services firewall exception (deprecated)

