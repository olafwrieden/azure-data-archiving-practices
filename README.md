# ![Banner](https://i.imgur.com/VlHw3qp.png)

**Purpose:** This resource is a collection of questions to ask and practices to consider for data archival and retention in [Microsoft Azure](https://azure.microsoft.com). This content is not officially endorsed by Microsoft and acts as a collection of learnings only.

**Definition of Archive:** An archive is an accumulation of historical records – in any media – or the physical facility in which they are located. Archives contain primary source documents that have accumulated over the course of an individual or organization's lifetime, and are kept to show the function of that person or organization.

<details>
  <summary>Expand Definition</summary>
  
  Professional archivists and historians generally understand archives to be records that have been naturally and necessarily generated as a product of regular legal, commercial, administrative, or social activities. They have been metaphorically defined as "the secretions of an organism", and are distinguished from documents that have been consciously written or created to communicate a particular message to posterity.
  
  In general, archives consist of records that have been selected for permanent or long-term preservation on grounds of their enduring cultural, historical, or evidentiary value. Archival records are normally unpublished and almost always unique, unlike books or magazines of which many identical copies may exist. This means that archives are quite distinct from libraries with regard to their functions and organization.

[Source](https://en.wikipedia.org/wiki/Archive)

</details>
<br />

**Why do organizations archive?**

- Performance: To alleviate performance issues (e.g. reducing table locking or offloading processing)
- Cost: Reduce costs to operational and analytical platforms
- Operational Alignment: To facilitate business requirements (e.g. reducing backup or restore times for critical business data sets)

<!-- ## Contributors

![Contributors](https://contrib.rocks/image?repo=olafwrieden/azure-data-archiving-practices) -->

## Table of Contents

- [Azure Storage Solutions](#)
- [Intro to Data Tiering](#)
- [Blob Lifecycle Management](#)
- [Immutable Blobs & Retention Policies](#)
- [Data Lake vs. Blob Storage](#)
- [Data Discovery and Classification Levels](#)
- [Azure Purview](#)

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

**⚡ Tip** Where possible, archive data in a compressed file format (e.g. avro, parquet, gzip).

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

**⚡ Tip** Where possible, archive data in a compressed file format (e.g. avro, parquet, gzip).

**⚡ Tip** When row level security is deployed, and the requirement exists to maintain it in the archive, data would be moved to a similar engine running at a lower performance tier and cost base which supports RLS.

## What are the Azure Storage Solutions?

Provided for overview purposes only, we will focus on _Blob Storage_ and _Data Lake Storage_ ([source](https://azure.microsoft.com/en-au/product-categories/storage/))

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

| Consideration                  | Azure Data Lake Storage Gen2                                                                                                           | Azure Blob Storage                                                                                                                                                                                                                                                                |
| :----------------------------- | :------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Structure                      | Hierarchical file system                                                                                                               | Object store with flat namespace                                                                                                                                                                                                                                                  |
| API                            | Works with both Blob and ADLS Gen2 APIs                                                                                                | Blob API only                                                                                                                                                                                                                                                                     |
| Authentication & Authorization | Supports ACL and POSIX permissions, plus additional granularity specific to ADLS Gen2                                                  | No granular access control                                                                                                                                                                                                                                                        |
| Geo-redundancy                 | Locally redundant (multiple copies of data in one Azure region)                                                                        | Locally redundant (LRS), zone redundant (ZRS), globally redundant (GRS), read-access globally redundant (RA-GRS)                                                                                                                                                                  |
| Encryption                     | Transparent, Server side <ul><li>With service-managed keys</li><li>With customer-managed keys in Azure KeyVault</li></ul>              | Transparent, Server side<ul><li>With service-managed keys</li><li>With customer-managed keys in Azure KeyVault</li></ul>Client-side encryption                                                                                                                                    |
| Object Management              | In ADLS Gen2, if a folder name is updated (for example) this is a metadata operation that is instantaneous and does not move the data. | Changes to folder names (for example) are physical write operations that copy data from one container to the new container name. This additional data movement could introduce higher risks of data loss (during movement) and result in longer operation times as data is moved. |

**Just-in-Time Archive Access**: What happens if we want to grant a user access to the archive or a section of the archive for specific period of time (eg. a day)? In Azure, we can leverage [Shared Access Signatures](https://docs.microsoft.com/en-us/azure/storage/common/storage-sas-overview) (preferred), or Access Keys. SAS provides secure delegated access to resources in your storage account, providing granular control over how data can be accessed. This includes what resources may be accessed, what permissions those resources can be accessed with, and for how long the SAS is valid.

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

## How do I read from an Archive Tier?

**Key message:** While a blob is in the archive access tier, it's considered offline and can't be read or modified. However, meta data is still online allowing us to list the blob and its properties.

- Data in the archive access tier is stored offline. The archive tier offers the lowest storage costs but also the highest access costs and latency. [SLA for Storage](https://azure.microsoft.com/support/legal/sla/storage/v1_5/).
- The hot and cool tiers support all redundancy options. The archive tier supports only LRS, GRS, and RA-GRS.
- Also see: [Rehydrating blob data from ab archive tier](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-rehydration).

### 🙋🏻‍♂️ Scenario: How do users request/read data?

Requesting data to an archived file is a common practice and one which which may involve two actors, a requestor and an approver.

#### The Idea

Design a basic PowerApp internally, to interact with the Azure Data Lake Gen2 API to scan the archive file metadata (cheap as this doesn't involve reading the file contents). If a Meta-data search is applied to the archive using PowerApps' native connectors to Cognitive Search, this approach provides a powerful archive search capability.

**Archivist:** An archivist is an information professional who assesses, collects, organizes, preserves, maintains control over records and archives determined to have long-term value. The records maintained by the archivist can consist of a variety of forms such as file systems and data stores.

**Requestor:** The requestor uses the PowerApp to browse the archive. Once the requestor has located one or more files in the archive to which they would like to request access, a request for access is lodged via the PowerApp to one or more approvers whose responsibility it is to approve or deny the file access (via the PowerApp).

**Approver:** Despite being allowed to approve/deny access requests, the approvers themselves cannot read or download the file(s) themselves. Approvers merely act as an approval gateway to permit read access to the files. If approved, the file(s) may now be downloaded or moved to an online tier.

## Archive Integrity: How do we safely delete source data?

A key question in the archiving equation is that of safely deleting source data. All too often the business worries (and rightfully so) about deleting the all important source data once it has been archived in the target destination.

An organization should have a documented reconciliation process in place to remove data from source systems. Part of this process should include an integrity verification step that checks whether the data written to the destination has the same "checksum" as the source data.
Checking for archive integrity during the archiving phase, provides piece of mind to the organization and its IT staff that a good copy of the data now exists in the archive. Thus allowing for the safe deletion of the source data.

**⚡ Tip** The Azure Copy `azcopy` command uses MD5 hashes to validate data integrity at the destination.

**⚡ Tip** If the risk of source data deletions is too great, you may decide not to delete the source data at all. This is however highly dependent on the organizational context.

## Data Lifecycle Management (Blob Storage)

### 🤔 Scenario

An insurance company stores data that they are required to hold on to for 7 years as per government regulations. On average, data is frequently accessed in the first two months after its creation. Then, however, it is only accessed occationally and after 6 months, only rarely.

### 💡 Conclusion

Because the data is accessed frequently (in the first 2 months), a hot storage tier is best suited. Between 2-6 months of the last modification date, this data is best moved to a cool storage tier. Later, 7 years after its last modification date, the blob is then deleted as it has served its purpose and is no longer required.

At the time of writing, this feature is still in preview in ADLS Gen2.

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

## Data Retention & Tamper-Proofing

### ⛔ Immutable Blobs: How do we prevent file changes?

Immutable Blob Storage enables organizations and their users to store business-critical data in a WORM (Write Once, Read Many) state at no additional cost. One benefit of configuring immutability is the added protection against data modifications or deletions, particularly in healthcare, financial, and broker-dealer organizations. This extends to any user, even those with account administrative priviledges.

Defining immutability policies allows organizations to comply with numerous industry regulations (incl. FINRA, SEC, CFTC) that call for tamper-proof storage of data. [Read more](https://azure.microsoft.com/en-au/blog/microsoft-azure-launches-tamper-proof-azure-immutable-blob-storage-for-financial-services/).

#### 1️⃣ Time-based Retention Policies

Store data for a specific interval. When set, objects can be created and read but not modified or deleted. Once expired, objects can be deleted but not overwritten.

- Minimum duration: 1 day, Maximum: 146,000 days (400 years)

#### 2️⃣ Legal Hold Policies

Store sensitive information that is critical to litigation or business use in a tamper-proof state for the desired duration until the hold is removed. This feature is not limited only to legal use cases but can also be thought of as an event-based hold or an enterprise lock, where the need to protect data based on event triggers or corporate policy is required.

#### 💭 Exercise: Reflecting on policies and logging

Suppose you work for a bank that is legally required to hold data for at least seven years. You may already have processes and procedures in place to comply with these obligations.

With the above listed examples of retention policies, you may recognise an advantage in enabling these optional set of tools at no additional cost. It may further prompt your organization to reflect on the following:

- Do we currently have a compliant set of technologies to support data retention policies?
- Could our existing data retention policies be strengthened?
- Could we benefit from better/automated logging when (and by whom) data is accessed?
- How do our data retention policies extend into our archiving considerations?

### WIP: Logging

Azure Storage Analytics Logs. Who is creating / reading document.

Mention it is in preview.
<https://docs.microsoft.com/en-us/azure/storage/common/storage-analytics-logging>

confidential, sensitive, internal only,

offical, sensitive, protected, public

high medium low none.

#### Immutable Blob Logs

Each container with a time-based retention policy enabled provides a policy audit log. The audit log includes up to seven time-based retention commands for locked time-based retention policies. Log entries include the user ID, command type, time stamps, and retention interval. The audit log is retained for the lifetime of the policy, in accordance with the SEC 17a-4(f) regulatory guidelines. [Logs for Time-based retention policies](https://docs.microsoft.com/en-us/azure/storage/blobs/immutable-time-based-retention-policy-overview#audit-logging)

Each container with a legal hold in effect provides a policy audit log. The log contains the user ID, command type, time stamps, and legal hold tags. The audit log is retained for the lifetime of the policy, in accordance with the SEC 17a-4(f) regulatory guidelines. [Logs for Legal Hold policies](https://docs.microsoft.com/en-us/azure/storage/blobs/immutable-legal-hold-overview#audit-logging)

## A word on Data Classification

Question: Do we archive the sensitive data or not? Answer: It depends!

The decision to archive sensitive data or not is one only your business can decide, perhaps it is driven by regulations outside your immediate control.

- Do we currently utilise any data classification labels (e.g. Official, Sensitive, Protected, Private ..)?

**⚡ Tip** Think about archiving sensitive file system data to different Storage Accounts / Blob Storage Containers in Azure, such that records are organized by `high` `medium` `low` `none` sensitivity levels - allowing for access to be controlled at the highest level (the storage medium itself).

## WIP: Data Discovery: Azure Purview

### 🔍 Introduction

Azure Purview is a unified data governance service that helps you manage and govern your on-premises, multicloud, and software-as-a-service (SaaS) data. Azure purview is Microsoft's data governance solution which helps you understand all data across your organisation. It's built on Apache Atlas, an open-source project for metadata management and governance for data assets.

Azure Blob Storage supports full and incremental scans to capture the metadata and schema. It also classifies the data automatically based on system and custom classification rules.

---

```text
NOTES: IDEAS FOR THIS DOC

Meta data tagging. Where did it come from? Properties -> Set metadata. Tool should support this. If not using Purview, highly recommend software adds metadata.

What are the SLAs for retrieving data from archive.
define meta data processing, outline naming convension (containers, folders, files) + tags, how do I ensure that the data archived is the same as in source system - md5 hash, hash locally, hash in azure - azcopy..

recommend small overlap,
recover data in source system,
flag as high-risk activity for archiving
```
