# ![Banner](https://i.imgur.com/VlHw3qp.png)

**Purpose:** This resource is a a collection of practices for data archival and retention in [Microsoft Azure](https://azure.microsoft.com). This content is not officially endorsed by Microsoft and acts as a collection of learnings only.

**Definition of Archive:** Will only use Cool, Performance is not an important factor in an archive. ADLS vs Blob - security and APIs. This is not a performance convo.

## Contributors

![Contributors](https://contrib.rocks/image?repo=olafwrieden/azure-data-archiving-practices)

## Table of Contents

- [Azure Storage Solutions](#)
- [Intro to Data Tiering](#)
- [Blob Lifecycle Management](#)
- [Immutable Blobs & Retention Policies](#)
- [Data Lake vs. Blob Storage](#)
- [Data Discovery and Classification Levels](#)
- [Azure Purview](#)

```text
NOTES: IDEAS FOR THIS DOC

What do we want to achieve?

blob vs data lake storage, performance when interacting with the data, services, blob only supports blob api, ADLS hdfs endpoint + blob api supported.
Management of objects -> change folder name in archive tier. In blob this is physical (copy data from one container to new one with new name). Moving files from one to another can intro failures + time consuming
In ADLS this is instantaneous (meta data operation).

Meta data tagging. Where did it come from? Properties -> Set metadata.
ADF to archive tag metadata.

What are the SLAs for retrieving data from archive.
define meta data processing, outline naming convension (containers, folders, files) + tags, how do I ensure that the data archived is the same as in source system - md5 hash, hash locally, hash in azure - azcopy..

how do we safely delete source data,
recommend small overlap,
recover data in source system,
flag as high-risk activity for archiving

sep storage account for each classification.
```

## Ask.. Where is my data?

Organisational data is typically spread throughout numerous storage mediums. These include but are not limited to **file system storage** and **data stores**. Understanding where your data is located, and what type of data it is, is the first step in addressing its archival practices.

### 🗃️ In a.. File System

> Think: SMB Storage, Network, S3 Buckets, Azure Storage, Google Storage, FTP, publicly available files

As we think about archiving this type of data. We should develop a set of business requirements that inform whether a particular file shall be archived or left alone.

**Your requirements may include:**

- When (date) was the file last accessed or modified?
- How is this file classified? Business rules may dictate that highly sensitive data shall not be archived.
- What tags (metadata) does the file have that we can filter by?
- Who is the owner of this file and is there a rule that says not to archive this person's files?
- What is the file type? We might only want to archive _.docx_ or ignore any _.py_ files.
- Are the files encrypted?

**⚡ Tip** Ultimately, it is the responsibility of the data owner to classify the file appropriately.

Depending on the nature of these files, a combination of business rules may need to be met before deeming the file fit for archiving. For example: A file may not have been accessed in while, but when it is required, it must be available without delay. In this scenario, perhaps we check if a custom tag (e.g. "no-archive") is present, and then ignore this file.

### 📚 In a.. Data Store

> Think: RDBMS, NoSQL, Hive, Key-Value Pair, Tuple Store

Similar to data in a file system, there should be a set of business rule that inform whether the a particular data store shall be archived or left alone.

**Your requirements may include:**

- When (date) was the row last accessed or modified?
- How is this data classified? Business rules may dictate that highly sensitive data shall not be archived.
- What tags (metadata) does the data have that we can filter by?
- Who is the owner of this data and is there a rule that says not to archive this person's data?
- What data storage engine is used? What is the source system / schema? For example: NoSQL vs Relational data stores like SQL Server or Oracle.
- Is the data store encrypted?
- Does the data store use row level security (RLS)?

**⚡ Tip** When row level security is deployed, and the requirement exists to maintain it in the archive, data would be moved to a similar engine running at a lower performance tier and cost base which supports RLS.

## What are the Azure Storage Solutions?

Provided for overview purposes, we will focus on _Blob Storage_ and _Data Lake Storage_ ([source](https://azure.microsoft.com/en-au/product-categories/storage/))

| Azure Product                                                                                               | Use it if you need..                                                                                                                           |
| :---------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------- |
| [Azure Disk Storage](https://azure.microsoft.com/en-au/services/storage/disks)                              | High-performance, durable block storage for Azure Virtual Machines                                                                             |
| [Azure Blob Storage](https://azure.microsoft.com/en-au/services/storage/blobs)                              | Massively scalable and secure object storage for cloud-native workloads, archives, data lakes, high-performance computing and machine learning |
| [Azure Data Lake Storage](https://azure.microsoft.com/en-au/services/storage/data-lake-storage)             | Massively scalable and secure data lake for your high-performance analytics workloads                                                          |
| [Azure Files](https://azure.microsoft.com/en-au/services/storage/files)                                     | Simple, secure and serverless enterprise-grade cloud file shares                                                                               |
| [Azure NetApp Files](https://azure.microsoft.com/en-au/services/netapp)                                     | Enterprise file storage, powered by NetApp                                                                                                     |
| [Azure Data Box](https://azure.microsoft.com/en-au/services/databox)                                        | Appliances and solutions for offline data transfer to Azure​                                                                                   |
| [Microsoft Azure Confidential Ledger](https://azure.microsoft.com/en-au/services/azure-confidential-ledger) | Store unstructured data that is completely tamper-proof and can be cryptographically verified                                                  |

## How to decide the archiving destination?

In this example we will be using Azure Storage and ADLS Gen2 as the repository for our archive. You are advised not to use ADLS Gen1 for any new projects - it is a legacy service. There is a 5 Pb limit per Storage Account, we will be using Standard storage tier, not Premium storage tier (read about the [differences](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-performance-tiers)).

Here are some differences you may find useful.

| Consideration                  | Azure Data Lake Storage Gen2                                                                                              | Azure Blob Storage                                                                                                                             |
| :----------------------------- | :------------------------------------------------------------------------------------------------------------------------ | :--------------------------------------------------------------------------------------------------------------------------------------------- |
| Structure                      | Hierarchical file system                                                                                                  | Object store with flat namespace                                                                                                               |
| API                            | Works with both Blob and ADLS Gen2 APIs                                                                                   | Blob API only                                                                                                                                  |
| Authentication & Authorization | Supports ACL and POSIX permissions, plus additional granularity specific to ADLS Gen2                                     | No granular access control                                                                                                                     |
| Geo-redundancy                 | Locally redundant (multiple copies of data in one Azure region)                                                           | Locally redundant (LRS), zone redundant (ZRS), globally redundant (GRS), read-access globally redundant (RA-GRS)                               |
| Encryption                     | Transparent, Server side <ul><li>With service-managed keys</li><li>With customer-managed keys in Azure KeyVault</li></ul> | Transparent, Server side<ul><li>With service-managed keys</li><li>With customer-managed keys in Azure KeyVault</li></ul>Client-side encryption |

**WIP: Just-in-Time Archive Access**: What happens if we want to grant a user access to the archive or a section of the archive for specific period of time (eg. a day)? _discuss options for Access Keys, SAS tokens.._

- [Data Ops for Auditing](https://docs.microsoft.com/en-us/azure/data-lake-store/data-lake-store-diagnostic-logs)

### WIP: Data Classification

MI access via tool to allow requestor to access data (via SAS token). Do we archive the sensitive data or not?

## Intro to Data Tiering

### 🔥 Hot Access Tier

The hot access tier has higher storage costs than cool and archive tiers, but the lowest access costs. Example usage scenarios for the hot access tier include:

- Data that's in active use or is expected to be read from and written to frequently
- Data that's staged for processing and eventual migration to the cool access tier

### 🧊 Cool Access Tier

The cool access tier has lower storage costs and higher access costs compared to hot storage. This tier is intended for data that will remain in the cool tier for at least 30 days. Example usage scenarios for the cool access tier include:

- Short-term backup and disaster recovery
- Older data not used frequently but expected to be available immediately when accessed
- Large data sets that need to be stored cost effectively, while more data is being gathered for future processing

### 📁 Archive Access Tier

The archive access tier has the lowest storage cost but higher data retrieval costs compared to hot and cool tiers. Data must remain in the archive tier for at least 180 days or be subject to an early deletion charge. Data in the archive tier can take several hours to retrieve depending on the specified rehydration priority. For small objects, a high priority rehydrate may retrieve the object from archive in under an hour.

Example usage scenarios for the archive access tier include:

- Long-term backup, secondary backup, and archival datasets
- Original (raw) data that must be preserved, even after it has been processed into final usable form
- Compliance and archival data that needs to be stored for a long time and is hardly ever accessed

🚨 **Beware:** The archive tier is not supported for ZRS, GZRS, or RA-GZRS accounts.

Read more about storage tiering: [Storage Blob Tiers](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-storage-tiers)

### What if we want to read from an Archive Tier?

Adapted from: [Rehydrate blob data from the archive tier](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-rehydration)

**Key message:** While a blob is in the archive access tier, it's considered offline and can't be read or modified. However, meta data is still online allowing us to list the blob and its properties.

- Data in the archive access tier is stored offline. The archive tier offers the lowest storage costs but also the highest access costs and latency. [SLA for Storage](https://azure.microsoft.com/support/legal/sla/storage/v1_5/).
- The hot and cool tiers support all redundancy options. The archive tier supports only LRS, GRS, and RA-GRS.

#### Two Options for Accessing Archived Data

1. Rehydate a blob to an online tier.
2. Copy an archive blob to an online tier.

[to be written]

#### 🙋🏻‍♂️ Scenario: How do users request/read data?

Requesting data to an archived file is a common practice and one which which may involve two actors, a requestor and an approver.

##### The Idea

Design a basic PowerApp internally, to interact with the Azure Data Lake Gen2 API to scan the archive file metadata (cheap as this doesn't involve reading the file contents). If a Meta-data search is applied to the archive using PowerApps' native connectors to Cognitive Search, this approach provides a powerful archive search capability.

**Requestor:** The requestor uses the PowerApp to browse the archive. Once the requestor has located one or more files in the archive to which they would like to request access, a request for access is lodged via the PowerApp to one or more approvers whose responsibility it is to approve or deny the file access (via the PowerApp).

**Approver:** Despite being allowed to approve/deny access requests, the approvers themselves cannot read or download the file(s) themselves. Approvers merely act as an approval gateway to permit read access to the files. If approved, the file(s) may now be downloaded or moved to an online tier until their

For structured data, this may be in the form of a Power BI report.
The Archive Manager is responsible for facilitating access to the archive.

## Data Lifecycle Management (Blob Storage)

### 🤔 Scenario

An insurance company stores data that they are required to hold on to for 7 years as per government regulations. On average, data is frequently accessed in the first two months after its creation. Then, however, it is only accessed occationally and after 6 months, only rarely.

### 💡 Conclusion

Because the data is accessed frequently (in the first 2 months), a hot storage tier is best suited. Between 2-6 months of the last modification date, this data is best moved to a cool storage tier. Later, 7 years after its last modification date, the blob is then deleted as it has served its purpose and is no longer required.

#### 💪🏻 Implementation

| Days since Last Modified | Action                                                          |
| :----------------------- | :-------------------------------------------------------------- |
| 0                        | Blob is stored in **hot** tier, as it is frequently accessed.   |
| 60 (2 months)            | Blob is moved to **cool** tier, as it is infrequently accessed. |
| 90 (3 months)            | Blob snapshot is **deleted** after 90 days.                     |
| 180 (6 months)           | Blob is moved to **archive** tier, as it is rarely accessed.    |
| 2,555 (7 years)          | Blob is **deleted** after 7 years.                              |

```json
{
  "rules": [
    {
      "name": "customerDataLifecycle",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["customer"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 60 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
            "delete": { "daysAfterModificationGreaterThan": 2555 }
          },
          "snapshot": {
            "delete": { "daysAfterCreationGreaterThan": 90 }
          }
        }
      }
    }
  ]
}
```

👉 Find out more: [Data Lifecyle Management](https://azure.microsoft.com/en-us/blog/azure-blob-storage-lifecycle-management-now-generally-available/)

## WIP: Immutable Blobs

We can lock access to blobs using Access Controls in

Add lifecycle management - bank example 7 years.

## WIP: Auditing

Auditing the archive process - validate data has not been tampered with. Sensitive data. 100% correct.

## Azure Data Lake Lifecycle Management

### Defining Action Sets

Similarly to our Blob Storage scenario, the `lastModified` property on a blob inside an Azure Data Lake can be used to trigger a series of action to move affected blobs to a different archive tier.

[continue.. action sets + filter sets]

[relate it back to archiving - to be written] (https://docs.microsoft.com/en-us/azure/azure-sql/database/data-discovery-and-classification-overview)

## Data Discovery and Classification Levels

### 🤔 Scenario

[relate it back to archiving - to be written]

## Azure Purview

Azure Purview is a unified data governance service that helps you manage and govern your on-premises, multicloud, and software-as-a-service (SaaS) data. Azure purview is Microsoft's data governance solution which helps you understand all data across your organisation. It's built on Apache Atlas, an open-source project for metadata management and governance for data assets.

Before registering data sources, you will need to create an Azure Purview account. For more information on creating a Purview account, go to (https://docs.microsoft.com/en-us/azure/purview/create-catalog-portal)

## Register and scan Azure Blob Storage with Azure Purview

Azure Blob Storage supports full and incremental scans to capture the metadata and schema. It also classifies the data automatically based on system and custom classification rules.

The following link will take you to the Microsoft document (how to register an Azure Blob Storage account in Purview and set up a scan).
(https://docs.microsoft.com/en-us/azure/purview/register-scan-azure-blob-storage-source)

## Register and scan Azure Data Lake Storage Gen1 and Gen2 with Azure Purview

The Azure Data Lake Storage Gen1 and Gen2 data source supports the following functionality:

- Full and incremental scans to capture metadata and classification in Azure Data Lake Storage Gen1 and Gen2.
- Lineage between data assets for ADF copy/dataflow activities.

The following link will take you to the Microsoft documentation that outlines how to register Azure Data Lake Storage Gen2 as data source in Azure Purview and set up a scan.

- Link to ADLS Gen1 (https://docs.microsoft.com/en-us/azure/purview/register-scan-adls-gen1)
- Link to ADLS Gen2 (https://docs.microsoft.com/en-us/azure/purview/register-scan-adls-gen2)
