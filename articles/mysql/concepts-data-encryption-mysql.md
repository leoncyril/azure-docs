---
title: Azure Database for MySQL data encryption with a customer-managed key
description: Azure Database for MySQL data encryption with a customer-managed key enables you to Bring Your Own Key (BYOK) for data protection at rest. It also allows organizations to implement separation of duties in the management of keys and data.
author: kummanish
ms.author: manishku
ms.service: mysql
ms.topic: conceptual
ms.date: 01/13/2020
---

# Azure Database for MySQL data encryption with a customer-managed key

> [!NOTE]
> At this time, you must request access to use this capability. To do so, contact AskAzureDBforMySQL@service.microsoft.com.

Data encryption with customer-managed keys for Azure Database for MySQL enables you to bring your own key (BYOK) for data protection at rest. It also allows organizations to implement separation of duties in the management of keys and data. With customer-managed encryption, you are responsible for, and in a full control of, a key's lifecycle, key usage permissions, and auditing of operations on keys.

Data encryption with customer-managed keys for Azure Database for MySQL, is set at the server-level. For a given server, a customer-managed key, called the key encryption key (KEK), is used to encrypt the data encryption key (DEK) used by the service. The KEK is an asymmetric key stored in a customer-owned and customer-managed [Azure Key Vault](../key-vault/key-Vault-secure-your-key-Vault.md) instance. The Key Encryption Key (KEK) and Data Encryption Key (DEK) is described in more detail later in this article.

Key Vault is a cloud-based, external key management system. It's highly available and provides scalable, secure storage for RSA cryptographic keys, optionally backed by FIPS 140-2 Level 2 validated hardware security modules (HSMs). It doesn't allow direct access to a stored key, but does provide services of encryption and decryption to authorized entities. Key Vault can generate the key, import it, or [have it transferred from an on-premises HSM device](../key-vault/key-Vault-hsm-protected-keys.md).

> [!NOTE]
> This feature is available in all Azure regions where Azure Database for MySQL supports "General Purpose" and "Memory Optimized" pricing tiers.

## Benefits

Data encryption for Azure Database for MySQL provides the following benefits:

* Data-access is fully controlled by you by the ability to remove the key and making the database inaccessible 
* Full control over the key-lifecycle, including rotation of the key to align with corporate policies
* Central management and organization of keys in Azure Key Vault
* Ability to implement separation of duties between security officers, and DBA and system administrators


## Terminology and description

**Data encryption key (DEK)**: A symmetric AES256 key used to encrypt a partition or block of data. Encrypting each block of data with a different key makes crypto analysis attacks more difficult. Access to DEKs is needed by the resource provider or application instance that is encrypting and decrypting a specific block. When you replace a DEK with a new key, only the data in its associated block must be re-encrypted with the new key.

**Key encryption key (KEK)**: An encryption key used to encrypt the DEKs. A KEK that never leaves Key Vault allows the DEKs themselves to be encrypted and controlled. The entity that has access to the KEK might be different than the entity that requires the DEK. Since the KEK is required to decrypt the DEKs, the KEK is effectively a single point by which DEKs can be effectively deleted by deletion of the KEK.

The DEKs, encrypted with the KEKs, are stored separately. Only an entity with access to the KEK can decrypt these DEKs. For more information, see [Security in encryption at rest](../security/fundamentals/encryption-atrest.md).

## How data encryption with a customer-managed key works

![Diagram that shows an overview of Bring Your Own Key](media/concepts-data-access-and-security-data-encryption/mysqloverview.png)

For a MySQL server to use customer-managed keys stored in Key Vault for encryption of the DEK, a Key Vault administrator gives the following access rights to the server:

* **get**: For retrieving the public part and properties of the key in the key vault.
* **wrapKey**: To be able to encrypt the DEK.
* **unwrapKey**: To be able to decrypt the DEK.

The key vault administrator can also [enable logging of Key Vault audit events](../azure-monitor/insights/azure-key-vault.md), so they can be audited later.

When the server is configured to use the customer-managed key stored in the key vault, the server sends the DEK to the key vault for encryptions. Key Vault returns the encrypted DEK, which is stored in the user database. Similarly, when needed, the server sends the protected DEK to the key vault for decryption. Auditors can use Azure Monitor to review Key Vault audit event logs, if logging is enabled.

## Requirements for configuring data encryption for Azure Database for MySQL

The following are requirements for configuring Key Vault:

* Key Vault and Azure Database for MySQL must belong to the same Azure Active Directory (Azure AD) tenant. Cross-tenant Key Vault and server interactions aren't supported. Moving resources afterwards requires you to reconfigure the data encryption.
* You must enable the soft-delete feature on the key vault, to protect from data loss if an accidental key (or Key Vault) deletion happens. Soft-deleted resources are retained for 90 days, unless the user recovers or purges them in the meantime. The recover and purge actions have their own permissions associated in a Key Vault access policy. The soft-delete feature is off by default, but you can enable it through PowerShell or the Azure CLI (note that you can't enable it through the Azure portal).
* Grant the Azure Database for MySQL access to the key vault with the get, wrapKey, and unwrapKey permissions by using its unique managed identity. In the Azure portal, the unique identity is automatically created when data encryption is enabled on the MySQL. See [Configure data encryption for MySQL](howto-data-encryption-portal.md) for detailed, step-by-step instructions when you're using the Azure portal.

* When you're using a firewall with Key Vault, you must enable the option **Allow trusted Microsoft services to bypass the firewall**.

The following are requirements for configuring the customer-managed key:

* The customer-managed key to be used for encrypting the DEK can be only asymmetric, RSA 2028.
* The key activation date (if set) must be a date and time in the past. The expiration date (if set) must be a future date and time.
* The key must be in the *Enabled* state.
* If you're importing an existing key into the key vault, make sure to provide it in the supported file formats (`.pfx`, `.byok`, `.backup`).

## Recommendations

When you're using data encryption by using a customer-managed key, here are recommendations for configuring Key Vault:

* Set a resource lock on Key Vault to control who can delete this critical resource and prevent accidental or unauthorized deletion.
* Enable auditing and reporting on all encryption keys. Key Vault provides logs that are easy to inject into other security information and event management tools. Azure Monitor Log Analytics is one example of a service that's already integrated.

* Ensure that Key Vault and Azure Database for MySQL reside in the same region, to ensure a faster access for DEK wrap and unwrap operations.

Here are recommendations for configuring a customer-managed key:

* Keep a copy of the customer-managed key in a secure place, or escrow it to the escrow service.

* If Key Vault generates the key, create a key backup before using the key for the first time. You can only restore the backup to Key Vault. For more information about the backup command, see [Backup-AzKeyVaultKey](https://docs.microsoft.com/powershell/module/az.keyVault/backup-azkeyVaultkey).

## Inaccessible customer-managed key condition

When you configure data encryption with a customer-managed key in Key Vault, continuous access to this key is required for the server to stay online. If the server loses access to the customer-managed key in Key Vault, the server begins denying all connections within 10 minutes. The server issues a corresponding error message, and changes the server state to *Inaccessible*. The only action allowed on a database in this state is deleting it.

### Accidental key access revocation from Key Vault

It might happen that someone with sufficient access rights to Key Vault accidentally disables server access to the key by:

* Revoking the key vault's get, wrapKey, and unwrapKey permissions from the server.
* Deleting the key.
* Deleting the key vault.
* Changing the key vault's firewall rules.

* Deleting the managed identity of the server in Azure AD.

## Monitor the customer-managed key in Key Vault

To monitor the database state, and to enable alerting for the loss of transparent data encryption protector access, configure the following Azure features:

* [Azure Resource Health](../service-health/resource-health-overview.md): An inaccessible database that has lost access to the customer key shows as "Inaccessible" after the first connection to the database has been denied.
* [Activity log](../service-health/alerts-activity-log-service-notifications.md): When access to the customer key in the customer-managed Key Vault fails, entries are added to the activity log. You can reinstate access as soon as possible, if you create alerts for these events.

* [Action groups](../azure-monitor/platform/action-groups.md): Define these to send you notifications and alerts based on your preferences.

## Restore and replicate with a customer's managed key in Key Vault

After Azure Database for MySQL is encrypted with a customer's managed key stored in Key Vault, any newly created copy of the server is also encrypted. You can make this new copy either through a local or geo-restore operation, or through read replicas. However, the copy can be changed to reflect a new customer's managed key for encryption. When the customer-managed key is changed, old backups of the server start using the latest key.

To avoid issues while setting up customer-managed data encryption during restore or read replica creation, it's important to follow these steps on the master and restored/replica servers:

* Initiate the restore or read replica creation process from the master Azure Database for MySQL.
* Keep the newly created server (restored/replica) in an inaccessible state, because its unique identity hasn't yet been given permissions to Key Vault.
* On the restored/replica server, revalidate the customer-managed key in the data encryption settings. This ensures that the newly created server is given wrap and unwrap permissions to the key stored in Key Vault.

## Next steps

Learn how to [set up data encryption with a customer-managed key for your Azure database for MySQL by using the Azure portal](howto-data-encryption-portal.md).
