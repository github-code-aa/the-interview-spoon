# Z07 - Azure Blob Storage: Study + Interview Guide

**How to use:** Read each topic's study notes first, then work the difficulty ladder top to bottom (Easy -> Tricky); cover the Answer and try to say it aloud before reading.
Limits, pricing and SDK surface change over time - anything marked "verify in current docs" should be rechecked before quoting numbers in an interview.

## Table of Contents
1. Topic 1: Storage account types, services, kinds and tiers
2. Topic 2: Blob types, containers, naming, virtual directories and HNS
3. Topic 3: Access tiers and lifecycle management
4. Topic 4: Redundancy and failover
5. Topic 5: .NET SDK usage and large-file transfer
6. Topic 6: Authentication and authorization
7. Topic 7: SAS tokens deep dive
8. Topic 8: Concurrency control (ETag, leases)
9. Topic 9: Metadata vs index tags
10. Topic 10: Data protection (versioning, soft delete, snapshots, PITR, change feed, immutability)
11. Topic 11: Events and reacting to uploads
12. Topic 12: File ingestion pipeline, idempotency and dedupe
13. Topic 13: Static website, CDN / Front Door, client-direct uploads and CORS
14. Topic 14: Network security and encryption
15. Topic 15: Performance, throttling and retries
16. Topic 16: Cost model
17. Topic 17: Queue/Table storage, Azure Files vs Blob vs Data Lake
18. Topic 18: Tooling (AzCopy, Storage Explorer, Azurite), monitoring and production scenarios

---

## Topic 1: Storage account types, services, kinds and tiers

**Study notes**

A storage account is the top-level namespace and billing/config boundary (redundancy, network rules, encryption, keys). It hosts several data services under one account name:

```
<account>.blob.core.windows.net   Blob (objects)
<account>.dfs.core.windows.net    Data Lake Gen2 endpoint (same data, HNS enabled)
<account>.file.core.windows.net   Azure Files (SMB/NFS/REST shares)
<account>.queue.core.windows.net  Queue (simple messaging)
<account>.table.core.windows.net  Table (NoSQL key/attribute)
```

| Kind / SKU | Services | Notes |
|---|---|---|
| General-purpose v2 (Standard) | Blob, File, Queue, Table, ADLS Gen2 | Default; hot/cool/cold/archive tiers |
| Premium block blobs | Block/append blobs | SSD, low latency, high transaction rates |
| Premium file shares | Files only | SSD-backed shares |
| Premium page blobs | Page blobs | Legacy unmanaged disks scenarios |
| General-purpose v1 / BlobStorage | Legacy | Upgrade to GPv2 |

Performance tiers: Standard (HDD-based, cost-optimised) vs Premium (SSD, consistently low latency). Premium does not support archive tier (verify in current docs for tier support per account type).

### 1.1 [Easy] What is an Azure Storage account and which services does it contain?
**Answer**
A storage account is the container for Azure's storage services: Blob, Files, Queue, Table (and Data Lake Gen2 as Blob with a hierarchical namespace). It defines the unique DNS name, redundancy, networking, and encryption settings shared by those services.
**Explanation**
All services in the account share the same capacity/throughput limits, firewall rules and keys. Because settings like redundancy are per account, teams often split accounts by workload (e.g., hot app data vs. archive logs) to apply different policies and avoid noisy neighbours.
**Code / CLI / Bicep**
```bash
az storage account create -n mystorageacct001 -g rg-demo -l westeurope \
  --sku Standard_LRS --kind StorageV2 --access-tier Hot \
  --min-tls-version TLS1_2 --allow-blob-public-access false
```

### 1.2 [Follow-up] Which account kind should you choose for new workloads and why?
**Answer**
General-purpose v2 for almost everything, because it supports all services and all access tiers. Choose Premium block blob only when you need consistently low latency or very high transaction rates, since it costs more per GB but less per transaction.
**Explanation**
GPv1 lacks tiering and is legacy; BlobStorage kind is legacy too. GPv2 account default access tier (Hot/Cool) sets the tier for blobs that do not specify one. Premium is SSD-backed and offered with LRS/ZRS only (verify in current docs).
**Code / CLI / Bicep**
```bicep
resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'mystorageacct001'
  location: resourceGroup().location
  sku: { name: 'Standard_ZRS' }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    supportsHttpsTrafficOnly: true
  }
}
```

### 1.3 [Medium] When would you enable Data Lake Gen2 (hierarchical namespace)?
**Answer**
When you need real directories, atomic rename/delete of folders, POSIX-style ACLs, or analytics (Spark/Synapse/Databricks) workloads. For plain object storage of app files it is usually unnecessary.
**Explanation**
HNS is chosen at account creation (an existing account can be upgraded via a validated one-way migration; verify current docs). It adds the `dfs` endpoint and changes some feature support (some Blob features are limited with HNS; verify compatibility matrix). Rename of a folder with a million blobs is a metadata operation in HNS but a copy+delete per blob in flat namespace.
**Code / CLI / Bicep**
```bash
az storage account create -n mydatalake001 -g rg-demo --sku Standard_LRS \
  --kind StorageV2 --enable-hierarchical-namespace true
```

### 1.4 [Tricky] Your team has one storage account for everything and gets 503 ServerBusy and cross-team firewall conflicts. What do you do?
**Answer**
Split into multiple accounts by workload/team: scalability targets and network/auth/redundancy settings are per account. This isolates noisy neighbours and lets each account have its own firewall, tiers, and RBAC scope.
**Explanation**
Per-account limits (ingress/egress, request rate) are shared across all containers. Accounts are cheap (cost is by usage), so splitting is usually the right fix before heavy partition engineering. Also useful: separate accounts for diagnostic logs, backups, and public static sites.
**Code / CLI / Bicep**
```bash
az storage account list -g rg-demo --query "[].{n:name,sku:sku.name,kind:kind}" -o table
```
**Gotchas / follow-ups**
- Account names are globally unique, 3-24 lowercase letters/digits.
- Moving data between accounts: AzCopy server-side copy.
- Quota increases per subscription/region exist; verify in current docs.

**Interview traps**
- "Blob storage" is a service; "storage account" is the resource - don't conflate.
- Redundancy, firewall, shared-key policy are account-level, not container-level.
- HNS is not a free toggle on all accounts - plan it up front.
- Premium != "faster at everything"; it's lower latency and costlier per GB.

**Cheat sheet**
- Default: GPv2 Standard.
- Endpoints: blob, dfs, file, queue, table.
- Premium block blob for latency-sensitive/high TPS.
- HNS = ADLS Gen2.
- Split accounts for isolation and limits.

---

## Topic 2: Blob types, containers, naming, virtual directories and HNS

**Study notes**

```
Account -> Container -> Blob (name may include '/')
Block blob  : up to ~190.7 TiB (50,000 blocks x up to 4000 MiB), general files
Append blob : append-only blocks, logs
Page blob   : 512-byte pages, random R/W, VHD disks (up to 8 TiB)
```
(Verify limits in current docs.)

- Container names: 3-63 chars, lowercase letters, digits, hyphens; start with letter/number; no consecutive hyphens.
- Blob names: up to 1024 chars, case-sensitive; `/` is just a character in flat namespace; "folders" are virtual (prefix + delimiter listing).
- Public access level: private (default), blob, container - and can be disallowed at account level.

### 2.1 [Easy] What are the three blob types and when do you use each?
**Answer**
Block blobs for documents, images, media and most files; append blobs for append-only data such as logs; page blobs for random read/write like VM disks. In app development you use block blobs ~99% of the time.
**Explanation**
Block blobs are assembled from blocks committed via a block list. Append blobs allow only AppendBlock (no modify in place). Page blobs back unmanaged disks. The type is fixed at creation.
**Code / CLI / Bicep**
```csharp
var container = new BlobContainerClient(connStrOrUri, "docs");
await container.CreateIfNotExistsAsync();
var append = container.GetAppendBlobClient("logs/app.log");
await append.CreateIfNotExistsAsync();
using var ms = new MemoryStream(Encoding.UTF8.GetBytes("line\n"));
await append.AppendBlockAsync(ms);
```

### 2.2 [Follow-up] Are there real folders in Blob storage?
**Answer**
Not in a flat namespace: a "folder" is just a shared name prefix with `/` delimiter. Empty folders don't exist unless you use HNS (Data Lake Gen2) or a placeholder blob.
**Explanation**
Listing uses `GetBlobsByHierarchy(delimiter: "/")` to simulate directories. Renaming a "folder" in flat namespace means copy+delete of every blob. With HNS, directories are real and renames are atomic.
**Code / CLI / Bicep**
```csharp
await foreach (BlobHierarchyItem item in container.GetBlobsByHierarchyAsync(
    delimiter: "/", prefix: "invoices/2025/"))
{
    Console.WriteLine(item.IsPrefix ? $"[dir] {item.Prefix}" : item.Blob.Name);
}
```

### 2.3 [Medium] How does a block blob upload work internally?
**Answer**
Data is split into blocks (Put Block, each with a base64 block ID), then a Put Block List commits the ordered list making the blob visible. Small blobs go through a single Put Blob call.
**Explanation**
Uncommitted blocks aren't visible and are garbage-collected after about a week if never committed. Block IDs in one blob must be the same length. The SDK does this for you above a threshold (`InitialTransferSize`), parallelising blocks.
**Code / CLI / Bicep**
```csharp
var blob = container.GetBlockBlobClient("big.bin");
var ids = new List<string>();
int i = 0;
foreach (var chunk in chunks) // each chunk is a byte[]
{
    string id = Convert.ToBase64String(BitConverter.GetBytes(i++));
    await blob.StageBlockAsync(id, new MemoryStream(chunk));
    ids.Add(id);
}
await blob.CommitBlockListAsync(ids);
```

### 2.4 [Tricky] Blob names with special characters / trailing dots or case differences cause bugs. What are the traps?
**Answer**
Blob names are case-sensitive and may contain almost any character, but must be URL-encoded properly; names ending with a dot or `/`-heavy paths may behave unexpectedly across tools and the HNS. Reserved characters and path segments like `..` are problematic.
**Explanation**
The SDK encodes names for you if you pass the raw name to `GetBlobClient`; double-encoding when you build URIs manually is a common bug. In an HNS account, path segment rules are stricter. Also consider that a name using `invoice.pdf` and `Invoice.pdf` are two different blobs, which matters if you dedupe by name.
**Code / CLI / Bicep**
```csharp
// Pass the raw name; SDK encodes it
var blob = container.GetBlobClient("reports/Q1 2025/résumé #1.pdf");
Console.WriteLine(blob.Uri); // encoded
```
**Gotchas / follow-ups**
- Don't use `new Uri(...)` + string concat with unescaped names.
- Avoid sequential prefixes for hot-partition-sensitive workloads (see Topic 15).
- Max 1024 chars; verify the segment limits (254 path segments in flat namespace) in current docs.

**Interview traps**
- "Folders" don't exist (flat) - delete-folder is N operations.
- Container names cannot contain uppercase or underscores.
- Blob type can't be changed after creation.
- Uncommitted blocks are invisible.

**Cheat sheet**
- Block = files, Append = logs, Page = disks.
- Prefix + delimiter = virtual directory.
- HNS = true dirs + ACLs.
- Raw names into SDK; let it encode.

---

## Topic 3: Access tiers and lifecycle management

**Study notes**

| Tier | Storage cost | Access cost | Min retention | Latency |
|---|---|---|---|---|
| Hot | highest | lowest | none | ms |
| Cool | lower | higher | 30 days | ms |
| Cold | lower still | higher | 90 days | ms |
| Archive | lowest | highest | 180 days | offline - hours to rehydrate |

(Verify minimum durations and prices in current docs.) Archive blobs cannot be read until rehydrated: either change tier to Hot/Cool/Cold (Set Blob Tier) or Copy Blob to a new blob in an online tier. Rehydrate priority: Standard (up to ~15 hours) or High (<1 hour for small blobs <10 GB; verify). Moving or deleting before min retention incurs an early deletion charge (prorated).

Lifecycle management: rules at account level (JSON) with filters (blobTypes, prefixMatch, blobIndexMatch) and actions (tierToCool, tierToCold, tierToArchive, delete) based on days since modification/creation/last access (last-access tracking must be enabled), also for versions and snapshots. Runs about once a day; not instant.

### 3.1 [Easy] What are the blob access tiers?
**Answer**
Hot, Cool, Cold and Archive - trading lower storage cost for higher access cost and minimum storage durations. Archive is offline and needs rehydration before reading.
**Explanation**
Tier can be set at account default level or per blob. Hot for frequent access, Cool/Cold for infrequent (backups, older docs), Archive for long-term compliance data rarely read.
**Code / CLI / Bicep**
```csharp
await blobClient.SetAccessTierAsync(AccessTier.Cool);
```

### 3.2 [Follow-up] How do you read an archived blob?
**Answer**
Rehydrate it: either Set Blob Tier to an online tier in place, or Copy Blob to a new blob in an online tier (keeps the archive copy). Rehydration takes hours, choose Standard or High priority.
**Explanation**
While rehydrating, the blob's `ArchiveStatus` is `rehydrate-pending-to-hot/cool`. A GET on an archived blob returns 409 (BlobArchived). Use Event Grid (BlobTierChanged) or poll properties to know when it completes. High priority costs more.
**Code / CLI / Bicep**
```csharp
await blob.SetAccessTierAsync(AccessTier.Hot, rehydratePriority: RehydratePriority.High);
BlobProperties p = await blob.GetPropertiesAsync();
Console.WriteLine($"{p.AccessTier} / {p.ArchiveStatus}");
```
```bash
az storage blob set-tier --account-name acct -c arch -n old.zip --tier Hot \
  --rehydrate-priority Standard --auth-mode login
```

### 3.3 [Medium] Write a lifecycle policy that tiers logs to cool after 30 days, archive after 90, delete after 365.
**Answer**
A lifecycle rule filtered to block blobs with prefix `logs/` and actions `tierToCool`, `tierToArchive`, `delete` keyed on `daysAfterModificationGreaterThan`. It is evaluated roughly daily.
**Explanation**
Rules can also manage versions and snapshots (`version`, `snapshot` blocks). Use `blobIndexMatch` to filter on tags. `daysAfterLastAccessTimeGreaterThan` requires enabling last access time tracking. Policies are asynchronous - don't rely on exact timing.
**Code / CLI / Bicep**
```bicep
resource lc 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = {
  parent: sa
  name: 'default'
  properties: {
    policy: {
      rules: [{
        name: 'logs-aging'
        enabled: true
        type: 'Lifecycle'
        definition: {
          filters: { blobTypes: ['blockBlob'], prefixMatch: ['logs/'] }
          actions: {
            baseBlob: {
              tierToCool: { daysAfterModificationGreaterThan: 30 }
              tierToArchive: { daysAfterModificationGreaterThan: 90 }
              delete: { daysAfterModificationGreaterThan: 365 }
            }
          }
        }
      }]
    }
  }
}
```

### 3.4 [Hard] Why did tiering millions of tiny blobs to Archive/Cool increase costs?
**Answer**
Tier-change operations are billed per operation and archive/cool have higher write and retrieval charges, plus minimum duration (early delete) and per-object overhead. For small blobs the transaction cost can exceed the storage savings.
**Explanation**
Archive also has per-blob metadata overhead (verify the billing overhead in docs). Better strategy: bundle small files into larger archives (zip/tar/parquet) before tiering, or keep small blobs in Hot/Cool. Compute break-even using actual retrieval frequency.
**Code / CLI / Bicep**
```bash
# See how much data per tier using Azure Monitor metric BlobCapacity with dimension Tier
az monitor metrics list --resource "/subscriptions/<sub>/resourceGroups/rg/providers/Microsoft.Storage/storageAccounts/acct/blobServices/default" \
  --metric BlobCapacity --dimension Tier --interval PT1H
```
**Gotchas / follow-ups**
- Overwriting a Cool blob resets its tier to the default (and may hit early delete).
- Lifecycle moves are billed operations.
- Blob versions/snapshots may keep consuming hot storage if rule excludes them.

### 3.5 [Tricky] An app GETs a blob and sometimes receives HTTP 409. Why?
**Answer**
The blob is in Archive tier (or mid-rehydration), so it is not readable: the error is `BlobArchived`. Lifecycle policies probably tiered it.
**Explanation**
Metadata and properties are still readable on archived blobs; data is not. Design the app to check `AccessTier`/`ArchiveStatus`, return 202-like "being retrieved" to users, trigger rehydration and subscribe to the tier-change event.
**Code / CLI / Bicep**
```csharp
try { await blob.DownloadToAsync(stream); }
catch (RequestFailedException ex) when (ex.ErrorCode == "BlobArchived")
{
    await blob.SetAccessTierAsync(AccessTier.Hot, rehydratePriority: RehydratePriority.Standard);
    // notify caller: retry later
}
```
**Gotchas / follow-ups**
- Copy-based rehydrate keeps archived original (billed twice).
- Lifecycle rule on "last access time" needs tracking enabled and has its own costs.

**Interview traps**
- Archive is offline, not "slow".
- Early deletion fee applies to tier changes as well as deletes.
- Lifecycle runs asynchronously (about daily).
- Default tier on account applies only to blobs without explicit tier.

**Cheat sheet**
- Hot > Cool(30d) > Cold(90d) > Archive(180d) (verify).
- Rehydrate = set tier or copy.
- Lifecycle = JSON rules, prefix/tag filters.
- Small-file tiering often a net loss.

---

## Topic 4: Redundancy and failover

**Study notes**

```
LRS   3 copies in one datacenter                  (11 nines durability, no zone/region protection)
ZRS   3 copies across 3 availability zones        (survives zone outage)
GRS   LRS + async copy to paired region (LRS there)
GZRS  ZRS primary + async copy to paired region
RA-GRS / RA-GZRS = same + read access to secondary endpoint (<acct>-secondary.blob.core.windows.net)
```
Geo-replication is asynchronous: RPO is typically <15 min but not guaranteed (check `LastSyncTime`). Customer-managed (planned/unplanned) failover switches the secondary to primary; after failover the account becomes LRS in the new primary until you re-enable geo-redundancy. Verify supported scenarios (HNS, etc.) in current docs.

### 4.1 [Easy] What is the difference between LRS, ZRS, GRS and GZRS?
**Answer**
LRS keeps three copies in one datacenter, ZRS spreads them across three zones in a region, GRS adds an async copy in the paired region, and GZRS combines ZRS in the primary with geo-copy. Higher levels cost more and protect against larger failures.
**Explanation**
Choose ZRS for zone-resilient regional apps with data residency needs; GZRS for the highest protection. LRS is fine for easily reproducible data.
**Code / CLI / Bicep**
```bash
az storage account update -n acct -g rg --sku Standard_GZRS
```

### 4.2 [Follow-up] What does the RA- prefix add?
**Answer**
Read access to the secondary region's endpoint at any time, not just after failover. Useful for read-heavy DR or fallback reads.
**Explanation**
The secondary is eventually consistent, so reads can be stale. The SDK supports `GeoRedundantSecondaryUri` in `BlobClientOptions` for automatic retry against secondary on read failures.
**Code / CLI / Bicep**
```csharp
var options = new BlobClientOptions
{
    GeoRedundantSecondaryUri = new Uri("https://acct-secondary.blob.core.windows.net")
};
var svc = new BlobServiceClient(new Uri("https://acct.blob.core.windows.net"),
    new DefaultAzureCredential(), options);
```

### 4.3 [Medium] How do you know replication lag and how does failover work?
**Answer**
Check the account's `geoReplicationStats.lastSyncTime`; data written after that time may be lost on unplanned failover. Failover is initiated by the customer (or Microsoft in a disaster) and re-points the primary DNS endpoint to the secondary.
**Explanation**
Because replication is async, unplanned failover can lose recent writes. After failover the account is LRS in the new region; you must reconfigure GRS/GZRS (which re-replicates data). Plan applications to use the account's primary endpoint name unchanged.
**Code / CLI / Bicep**
```bash
az storage account show -n acct -g rg --expand geoReplicationStats \
  --query "geoReplicationStats.{status:status,lastSync:lastSyncTime}"
az storage account failover -n acct -g rg --failover-type Planned   # verify flags in current docs
```

### 4.4 [Tricky] Your app uses GRS; the primary region goes down - can you read blobs immediately?
**Answer**
Not with plain GRS: the secondary is not readable until failover. Only RA-GRS/RA-GZRS allow reads from the secondary without failover.
**Explanation**
Many teams assume GRS means automatic read availability. You also can't write to the secondary until failover completes. Design: RA-GZRS + client secondary-endpoint fallback for reads, queued writes for outages.
**Code / CLI / Bicep**
```bash
az storage account update -n acct -g rg --sku Standard_RAGZRS
```
**Gotchas / follow-ups**
- Failover can cause data loss (RPO).
- Blob versioning/soft-delete state replicates too, but check feature restrictions with failover.
- Lifecycle policies and some features may need re-validation after failover.

**Interview traps**
- GRS != readable secondary.
- ZRS does not protect from a regional disaster.
- Geo replication is not a backup (deletes replicate).
- LRS-> ZRS migration is done via conversion/live migration (verify).

**Cheat sheet**
- LRS < ZRS < GRS < GZRS.
- RA = read secondary.
- Check LastSyncTime.
- Replication isn't backup; use soft delete/versioning.

---

## Topic 5: .NET SDK usage and large-file transfer

**Study notes**

Package: `Azure.Storage.Blobs` (v12). Client hierarchy:

```
BlobServiceClient  (account)
  -> BlobContainerClient  (container)
       -> BlobClient / BlockBlobClient / AppendBlobClient / PageBlobClient
```
Clients are thread-safe and designed to be reused (register as singletons). Use `DefaultAzureCredential` / `TokenCredential` where possible. Transfer tuning: `StorageTransferOptions { InitialTransferSize, MaximumTransferSize, MaximumConcurrency }` on `BlobUploadOptions`/`BlobDownloadToOptions`. `UploadAsync(Stream)` auto-chunks large streams. `OpenReadAsync`/`OpenWriteAsync` give stream semantics. Progress via `IProgress<long>` in options.

### 5.1 [Easy] How do you upload and download a blob in .NET?
**Answer**
Create a `BlobServiceClient` (with `DefaultAzureCredential`), get a container and blob client, then call `UploadAsync` / `DownloadToAsync` or `DownloadStreamingAsync`.
**Explanation**
`overwrite: true` is required to replace an existing blob (default fails with 409 BlobAlreadyExists). Prefer streams over byte arrays for large content.
**Code / CLI / Bicep**
```csharp
var svc = new BlobServiceClient(new Uri("https://acct.blob.core.windows.net"), new DefaultAzureCredential());
var blob = svc.GetBlobContainerClient("docs").GetBlobClient("a/b.pdf");

await using (var fs = File.OpenRead("b.pdf"))
    await blob.UploadAsync(fs, overwrite: true);

BlobDownloadStreamingResult r = await blob.DownloadStreamingAsync();
await using var outFile = File.Create("copy.pdf");
await r.Content.CopyToAsync(outFile);
```

### 5.2 [Follow-up] How do you set content type and metadata on upload?
**Answer**
Use `BlobUploadOptions` with `HttpHeaders = new BlobHttpHeaders { ContentType = ... }`, `Metadata`, `Tags`, and `AccessTier`. If you don't set ContentType it defaults to application/octet-stream, which breaks browser rendering.
**Explanation**
Setting headers at upload avoids a second Set Properties call. `Conditions` in the options provides conditional writes (e.g., `IfNoneMatch = ETag.All` for create-only).
**Code / CLI / Bicep**
```csharp
var opts = new BlobUploadOptions
{
    HttpHeaders = new BlobHttpHeaders { ContentType = "application/pdf", CacheControl = "max-age=3600" },
    Metadata = new Dictionary<string,string> { ["source"] = "api" },
    Tags = new Dictionary<string,string> { ["status"] = "new" },
    AccessTier = AccessTier.Hot,
    Conditions = new BlobRequestConditions { IfNoneMatch = ETag.All }
};
await blob.UploadAsync(stream, opts);
```

### 5.3 [Medium] How do you tune large file uploads and report progress?
**Answer**
Set `TransferOptions` (`MaximumTransferSize` block size, `MaximumConcurrency` parallel blocks, `InitialTransferSize` threshold for single-shot) and pass an `IProgress<long>`. Bigger blocks and more concurrency raise throughput at the cost of memory.
**Explanation**
Memory used is roughly block size x concurrency. Defaults are conservative (verify current defaults in docs). Concurrency above what your bandwidth/CPU can use gives diminishing returns and can cause timeouts.
**Code / CLI / Bicep**
```csharp
var opts = new BlobUploadOptions
{
    TransferOptions = new StorageTransferOptions
    {
        InitialTransferSize = 8 * 1024 * 1024,
        MaximumTransferSize = 16 * 1024 * 1024,
        MaximumConcurrency = 8
    },
    ProgressHandler = new Progress<long>(b => Console.WriteLine($"{b:N0} bytes"))
};
await blob.UploadAsync(File.OpenRead(path), opts);
```

### 5.4 [Medium] OpenReadAsync / OpenWriteAsync - when do you use them?
**Answer**
`OpenReadAsync` returns a seekable, buffered read stream that fetches ranges on demand; `OpenWriteAsync` returns a write stream that stages blocks and commits on dispose. Use them to stream through parsers (CSV/JSON/zip) without loading the whole blob.
**Explanation**
You must dispose/flush the write stream to commit. Reading via `OpenReadAsync` does ranged GETs with ETag condition by default, so if the blob changes during read you get a 412 (consistent read). `OpenWriteAsync(overwrite: true)` is required to replace.
**Code / CLI / Bicep**
```csharp
await using Stream read = await blob.OpenReadAsync(new BlobOpenReadOptions(allowModifications: false));
using var sr = new StreamReader(read);
while (await sr.ReadLineAsync() is { } line) { /* process */ }

await using Stream write = await outBlob.OpenWriteAsync(overwrite: true);
await JsonSerializer.SerializeAsync(write, bigObject);
```

### 5.5 [Hard] Why is MemoryStream a pitfall and how do you avoid OOM in Functions?
**Answer**
`MemoryStream` buffers the whole blob in memory (and LOH/array doubling), so large files cause OOM or GC pressure, particularly in Functions with limited memory. Stream directly: `DownloadStreamingAsync`, `OpenReadAsync`, and pass the stream through to processing/upload.
**Explanation**
Common anti-patterns: `DownloadToAsync(new MemoryStream())`, `ToArray()`, then base64. Instead pipe from source to target; for uploads from HTTP requests, use `Request.Body` straight to `UploadAsync`. Note the Functions blob trigger/binding with `Stream` is streamed, while `byte[]`/`string` bindings load all of it.
**Code / CLI / Bicep**
```csharp
// Bad
var ms = new MemoryStream(); await blob.DownloadToAsync(ms); var bytes = ms.ToArray();

// Good: stream copy blob -> blob
var src = await srcBlob.OpenReadAsync();
await dstBlob.UploadAsync(src, overwrite: true);
// or server-side, no data through your app:
await dstBlob.StartCopyFromUriAsync(srcBlob.Uri /* needs auth/SAS if cross-account */);
```
**Gotchas / follow-ups**
- Remember `MemoryStream.Position = 0` before upload, otherwise you upload zero bytes.
- Streams from `OpenWriteAsync` must be disposed asynchronously.
- Server-side copy avoids bandwidth entirely.

### 5.6 [Tricky] A 5-GB upload via an HTTP-triggered Function times out; what are your options?
**Answer**
Don't proxy large files through the Function: issue a user delegation SAS and let the client upload directly to Blob, then react via Event Grid. If you must proxy, stream and use background processing, since HTTP request timeouts (~230 s for the Azure front end) apply regardless of plan.
**Explanation**
Consumption plan execution timeout and the load-balancer idle timeout limit long requests. Direct-to-blob offloads bandwidth/CPU and scales. Chunked client upload (`StageBlock`/`CommitBlockList` using SAS) provides resume on failure.
**Code / CLI / Bicep**
```csharp
// Server returns a short-lived upload URL (see Topic 7 for user delegation SAS)
var sas = new BlobSasBuilder(BlobSasPermissions.Create | BlobSasPermissions.Write, DateTimeOffset.UtcNow.AddMinutes(15))
{ BlobContainerName = "uploads", BlobName = $"{Guid.NewGuid()}.bin" };
```
**Gotchas / follow-ups**
- Retry policy on the client can re-send blocks; blocks are idempotent by block ID.
- Set `NetworkTimeout`/retry options in `BlobClientOptions.Retry` for slow links.

**Interview traps**
- Creating a new BlobServiceClient per request (socket exhaustion / token cache loss).
- Forgetting `overwrite: true`.
- Forgetting to reset stream position.
- Missing ContentType.
- `byte[]` bindings in Functions load all data.

**Cheat sheet**
- Reuse clients (singleton / DI `AddAzureClients`).
- Tune: block size x concurrency = memory.
- OpenRead/OpenWrite to stream.
- StageBlock + CommitBlockList for manual chunking.
- Big files: direct-to-blob with SAS.

---

## Topic 6: Authentication and authorization

**Study notes**

| Method | Identity | Revocable | Recommended |
|---|---|---|---|
| Entra ID (RBAC) incl. managed identity | user/app/MI | yes (role removal) | Yes |
| Shared Key (account key / connection string) | the account | rotate key | No (disable) |
| SAS (account/service) | token holder | limited | delegated access |
| User delegation SAS | signed by Entra creds | yes | preferred SAS |
| Anonymous public | anyone | n/a | static public content only |

Built-in data-plane roles: **Storage Blob Data Reader**, **Storage Blob Data Contributor**, **Storage Blob Data Owner** (includes POSIX ACL management), **Storage Blob Delegator** (to get a user delegation key). Management roles (Owner/Contributor) do NOT grant data access (though Contributor can list keys, which then gives full access). RBAC propagation can take minutes. Disable shared key: `allowSharedKeyAccess=false`.

### 6.1 [Easy] How should an app authenticate to Blob storage?
**Answer**
Use Entra ID with a managed identity and an RBAC data role like Storage Blob Data Contributor, via `DefaultAzureCredential`. This removes secrets from config.
**Explanation**
In Azure, `DefaultAzureCredential` picks up the managed identity; locally it falls back to Visual Studio/Azure CLI credentials. For a user-assigned MI, set `ManagedIdentityClientId` in the options (or `AZURE_CLIENT_ID`).
**Code / CLI / Bicep**
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ManagedIdentityClientId = builder.Configuration["UamiClientId"] // only for UAMI
});
var svc = new BlobServiceClient(new Uri("https://acct.blob.core.windows.net"), cred);
```

### 6.2 [Follow-up] Which role would you assign to read vs write vs delete?
**Answer**
Storage Blob Data Reader for read/list, Contributor for read/write/delete, Owner for everything plus ACL/ownership management. Assign at the narrowest scope (container rather than account).
**Explanation**
Scope options: subscription, resource group, account, container (with HNS, ACLs for directory-level). Conditions (ABAC) can further limit by container name or blob path/tags (verify current support).
**Code / CLI / Bicep**
```bicep
// Storage Blob Data Contributor: ba92f5b4-2d11-453d-a403-e96b0029c9fe
resource ra 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: sa
  name: guid(sa.id, principalId, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: principalId
    principalType: 'ServicePrincipal'
  }
}
```

### 6.3 [Medium] Why disable shared key access, and what breaks?
**Answer**
Account keys are all-powerful, unauditable per caller, and hard to rotate; disabling them forces Entra auth and enables user-delegation SAS only. Things that break: connection-string clients, account/service SAS, and some tools/triggers still configured with keys.
**Explanation**
Azure Functions host storage (`AzureWebJobsStorage`) historically used connection strings; identity-based connections are supported (`AzureWebJobsStorage__accountName` + roles) but verify role requirements per trigger (Blob Data Owner, Queue Data Contributor, Table Data Contributor for the host). Portal browsing may default to key auth - switch to Entra auth.
**Code / CLI / Bicep**
```bash
az storage account update -n acct -g rg --allow-shared-key-access false
az storage blob list --account-name acct -c docs --auth-mode login
```

### 6.4 [Hard] You have the right role but get 403 AuthorizationPermissionMismatch. Why?
**Answer**
Usually: wrong principal (the app runs as a different identity than assumed), role assigned at a different scope, role propagation delay, or a Reader-only management role without a data role. Use the error code: `AuthorizationPermissionMismatch` means the identity authenticated but lacks permission, while `AuthorizationFailure` suggests a network/firewall issue.
**Explanation**
Check `az role assignment list --assignee <objectId> --scope <account>`. For UAMI, confirm the client ID is correctly passed, otherwise `DefaultAzureCredential` may pick another credential (e.g., a stale VS/CLI login). Also ABAC conditions can silently limit allowed actions.
**Code / CLI / Bicep**
```bash
az role assignment list --assignee <principalObjectId> --scope $(az storage account show -n acct -g rg --query id -o tsv) -o table
```
**Gotchas / follow-ups**
- Owner/Contributor on the account != blob data access via Entra.
- Wait up to several minutes after assignment.
- Use `ManagedIdentityCredential` explicitly in prod to avoid credential-chain surprises.

### 6.5 [Tricky] Is "Allow Blob anonymous access" the same as a public container?
**Answer**
No: the account setting only permits containers to be made public; each container's public access level still must be set. Turning off the account setting overrides all containers to private.
**Explanation**
Security baselines (Defender/Policy) recommend disabling it. Public access serves anonymous GET without auth, so data exfil risk is high.
**Code / CLI / Bicep**
```bash
az storage account update -n acct -g rg --allow-blob-public-access false
az storage container set-permission -n web --public-access off --account-name acct --auth-mode login
```
**Gotchas / follow-ups**
- Static website `$web` is served publicly via the web endpoint by design.

**Interview traps**
- Owner role at subscription != data access.
- Connection string in App Settings is a leaked-secret risk; use Key Vault references or MI.
- Role assignment delay causes "flaky" 403 after deployment.
- Account SAS can't be user-delegation-signed.

**Cheat sheet**
- Prefer MI + RBAC.
- Reader / Contributor / Owner + Delegator.
- allowSharedKeyAccess=false.
- AuthorizationPermissionMismatch = RBAC; AuthorizationFailure = network (usually).

---

## Topic 7: SAS tokens deep dive

**Study notes**

A Shared Access Signature is a signed query string granting limited, time-bound access.

| Type | Signed by | Scope | Revocation |
|---|---|---|---|
| User delegation SAS | user delegation key (Entra) | Blob/container only | revoke delegation key, role removal, short expiry |
| Service SAS | account key | one service (blob/container) | stored access policy or rotate key |
| Account SAS | account key | multiple services | rotate key |

Parameters: `sp` (permissions), `st/se` (start/expiry), `sip` (IP range), `spr` (https only), `sr` (resource), `si` (stored access policy id). Stored access policies (max 5 per container) allow changing/revoking without re-issuing. Best practice: user delegation SAS, short expiry, least permissions, HTTPS only, IP restrict where feasible. User delegation key lifetime max 7 days.

### 7.1 [Easy] What is a SAS and why use it?
**Answer**
A SAS is a URL token giving delegated, scoped, time-limited access to storage resources without sharing account keys. Typical use: let a browser upload or download a single blob.
**Explanation**
It encodes permissions, resource and times, signed with a key. Whoever holds the URL has the access ("bearer"), so treat it as a secret.
**Code / CLI / Bicep**
```bash
az storage blob generate-sas --account-name acct -c docs -n a.pdf --permissions r \
  --expiry 2026-12-31T00:00Z --https-only --auth-mode login --as-user -o tsv
```

### 7.2 [Follow-up] Generate a user delegation SAS in .NET.
**Answer**
Get a `UserDelegationKey` from `BlobServiceClient.GetUserDelegationKeyAsync` (requires Storage Blob Delegator or an equivalent action), then build a `BlobSasBuilder` and sign with `ToSasQueryParameters`.
**Explanation**
The SAS permissions are the intersection of what you sign and what the signing identity actually has by RBAC. Keep expiry short (minutes).
**Code / CLI / Bicep**
```csharp
var start = DateTimeOffset.UtcNow.AddMinutes(-5); // clock skew
var expiry = DateTimeOffset.UtcNow.AddMinutes(15);
UserDelegationKey key = await svc.GetUserDelegationKeyAsync(start, expiry);

var sas = new BlobSasBuilder
{
    BlobContainerName = "docs",
    BlobName = "a.pdf",
    Resource = "b",
    StartsOn = start,
    ExpiresOn = expiry,
    Protocol = SasProtocol.Https
};
sas.SetPermissions(BlobSasPermissions.Read);

var uri = new BlobUriBuilder(blob.Uri)
{
    Sas = sas.ToSasQueryParameters(key, svc.AccountName)
}.ToUri();
```

### 7.3 [Medium] How do you revoke a SAS?
**Answer**
Depends on type: a service SAS tied to a stored access policy is revoked by deleting/changing the policy; an ad-hoc account-key SAS requires rotating the key; a user delegation SAS requires revoking the user delegation keys (`az storage account revoke-delegation-keys`) or removing the signer's permissions.
**Explanation**
This is why ad-hoc long-lived SAS tokens are dangerous. Revocation via key rotation invalidates all SAS signed by that key and all key-based clients.
**Code / CLI / Bicep**
```bash
az storage account revoke-delegation-keys -n acct -g rg
az storage account keys renew -n acct -g rg --key key1
az storage container policy create -c docs --name readpol --permissions r --expiry 2026-12-31T00:00Z --account-name acct --auth-mode login
```

### 7.4 [Hard] What are the security risks of SAS and how do you mitigate?
**Answer**
SAS is a bearer token: leaks (logs, URLs, browser history, referrers) give access until expiry. Mitigate with user delegation SAS, short expiry, minimal permissions, HTTPS-only, IP ranges, per-blob scope, no logging of full URLs, and monitoring.
**Explanation**
Account SAS with broad permissions and far-future expiry is the worst case. A SAS can't be listed by Azure - there is no server inventory, so you cannot enumerate issued SAS. Azure Policy can require a SAS expiration policy (account-level "SAS expiration period", which creates log alerts rather than blocking, verify docs) and `allowSharedKeyAccess=false` blocks key-signed SAS.
**Code / CLI / Bicep**
```bicep
properties: {
  allowSharedKeyAccess: false
  sasPolicy: { sasExpirationPeriod: '00.01:00:00', expirationAction: 'Log' }
}
```
**Gotchas / follow-ups**
- Clock skew: set start a few minutes in the past or omit it.
- The `sip` limit may break behind NAT/proxy/CDN.
- Logs: StorageBlobLogs record authentication type (SAS) for audit.

### 7.5 [Tricky] SAS works from your laptop but gives 403 AuthenticationFailed in production. Why?
**Answer**
Common causes: URL got mangled (unencoded `+`/`&`, truncated), signature computed with wrong canonical resource or version, start time in the future vs skew, expiry, IP restriction (`sip`), or https-only vs http. `AuthenticationFailed` with a signature message usually means the string-to-sign didn't match.
**Explanation**
The error body includes the expected string-to-sign (`AuthenticationErrorDetail`) which helps debug. Also confirm no one appended a second query string onto a SAS URL incorrectly and that the account has not disabled shared key (for key-signed SAS).
**Code / CLI / Bicep**
```csharp
var b = new BlobUriBuilder(new Uri(sasUrl));
Console.WriteLine($"{b.Sas.StartsOn:u} - {b.Sas.ExpiresOn:u} perms={b.Sas.Permissions} ip={b.Sas.IPRange}");
```
**Gotchas / follow-ups**
- User delegation SAS fails if the signing identity lost RBAC after issuing.
- AuthorizationFailure (network) vs AuthenticationFailed (bad creds) are different.

**Interview traps**
- "SAS is secure because it expires" - only if short and scoped.
- Stored access policy only applies to service SAS (not user delegation/account SAS).
- Key rotation breaks everything signed by that key.
- SAS URL in logs = leak.

**Cheat sheet**
- Prefer user delegation SAS (Entra-signed, max 7 days key).
- sp/se/sip/spr/si.
- Revoke: policy / revoke keys / rotate key.
- Start 5 min back for skew.
- Short expiry + least privilege + HTTPS.

---

## Topic 8: Concurrency control (ETag, leases)

**Study notes**

Every blob has an **ETag** that changes on any modification (content, metadata, properties). Conditional headers: `If-Match`, `If-None-Match`, `If-Modified-Since`, `If-Unmodified-Since`. Failed condition: **412 Precondition Failed** (`ConditionNotMet`) for writes; **304** for conditional GET. **409 Conflict** covers state conflicts: `BlobAlreadyExists`, `LeaseIdMissing`/`LeaseAlreadyPresent` (lease problems return 409/412 depending on case), `BlobArchived`.

Leases: exclusive write/delete lock on a blob (or container). Duration 15-60 seconds or infinite (-1). Operations: acquire, renew, change, release, break. While leased, writes must supply the lease ID (otherwise 412 `LeaseIdMissing`).

```
Optimistic: read(ETag=E1) -> modify -> write If-Match:E1 -> 412 if changed
Pessimistic: acquire lease -> write(leaseId) -> release
```

### 8.1 [Easy] What is optimistic concurrency in Blob storage?
**Answer**
Read the blob's ETag, then write with `If-Match: <etag>`; if someone else changed it, the service returns 412 and you retry. No locks are held.
**Explanation**
Because the ETag changes on every modification, conflicts are detected reliably. This is the SDK's default advice for read-modify-write of small state blobs.
**Code / CLI / Bicep**
```csharp
var props = await blob.GetPropertiesAsync();
ETag etag = props.Value.ETag;
// ... modify content ...
try
{
    await blob.UploadAsync(newStream, new BlobUploadOptions
    { Conditions = new BlobRequestConditions { IfMatch = etag } });
}
catch (RequestFailedException ex) when (ex.Status == 412) { /* reload and retry */ }
```

### 8.2 [Follow-up] How do you create a blob only if it doesn't already exist?
**Answer**
Use `IfNoneMatch = ETag.All`; if the blob exists the call fails with 409 BlobAlreadyExists (SDK/service surfaces it as 409 for create-if-absent) - treat it as "already created".
**Explanation**
Useful for idempotent ingestion and for "first writer wins" semantics. Some operations return 412 instead of 409 with this condition; handle both (`ConditionNotMet` / `BlobAlreadyExists`).
**Code / CLI / Bicep**
```csharp
try
{
    await blob.UploadAsync(stream, new BlobUploadOptions
    { Conditions = new BlobRequestConditions { IfNoneMatch = ETag.All } });
}
catch (RequestFailedException ex) when (ex.Status is 409 or 412) { /* exists */ }
```

### 8.3 [Medium] How do leases work, and how do you use them as a distributed lock?
**Answer**
A lease gives one holder exclusive write/delete access for 15-60 s (renewable) or infinite. Use a small "lock" blob, acquire a lease, do the work while renewing, and release in `finally`; a failure to acquire (409 LeaseAlreadyPresent) means someone else holds the lock.
**Explanation**
This is a simple, cheap leader election / singleton job pattern (the Functions host and Durable Task use blob leases). Leases expire if the holder dies and doesn't renew, giving automatic recovery. Not suitable for high-frequency locking (each op is a REST call, latency ~10s of ms).
**Code / CLI / Bicep**
```csharp
var lockBlob = container.GetBlobClient("locks/job1");
if (!await lockBlob.ExistsAsync())
    await lockBlob.UploadAsync(BinaryData.FromString("lock"), overwrite: false); // handle 409 race

var lease = lockBlob.GetBlobLeaseClient();
try
{
    await lease.AcquireAsync(TimeSpan.FromSeconds(30));
    // do work; call lease.RenewAsync() periodically
}
catch (RequestFailedException ex) when (ex.ErrorCode == "LeaseAlreadyPresent") { return; }
finally { try { await lease.ReleaseAsync(); } catch { } }
```

### 8.4 [Hard] Leases vs ETag - when each, and what are the failure modes?
**Answer**
ETag (optimistic) is better for low-contention updates and needs no cleanup; leases fit exclusive long-running work or single-instance jobs. Lease pitfalls: process pauses past the TTL (GC pause, throttling) so two holders believe they own the lock; always validate by passing the lease ID on the writes.
**Explanation**
Writes made while holding the lease must include `Conditions.LeaseId`; the service then rejects writes from an expired holder if another has acquired a new lease. This "fencing" via lease ID gives safety for blob writes but not for external side-effects. Infinite leases can orphan locks (need `BreakAsync`).
**Code / CLI / Bicep**
```csharp
await blob.UploadAsync(stream, new BlobUploadOptions
{
    Conditions = new BlobRequestConditions { LeaseId = lease.LeaseId }
});
await lease.BreakAsync(TimeSpan.Zero); // admin recovery
```
**Gotchas / follow-ups**
- Lease on a blob blocks deletes (even soft-delete flows) until released/broken.
- Container leases block container deletion.
- A leased blob can still be read.

### 8.5 [Tricky] You get intermittent 412s on a blob you never leased. Why?
**Answer**
Most likely a conditional header: either your own `IfMatch` with a stale ETag, or `OpenReadAsync` detecting the blob changed mid-read, or the SDK retry re-sending a conditional write that actually succeeded the first time (the retry then fails ETag).
**Explanation**
The retry case: request succeeds server-side but response lost -> SDK retries with the old ETag -> 412 even though your data was written. Solve by making writes idempotent (compare content/hash after a 412) rather than blind retrying.
**Code / CLI / Bicep**
```csharp
catch (RequestFailedException ex) when (ex.Status == 412)
{
    var cur = await blob.GetPropertiesAsync();
    if (cur.Value.ContentHash.AsSpan().SequenceEqual(myHash)) return; // my earlier try succeeded
}
```
**Gotchas / follow-ups**
- Metadata updates also change the ETag.
- `OpenReadAsync` with `allowModifications: true` skips the check.

**Interview traps**
- 412 vs 409 vs 304 differences.
- Leases are not distributed transactions.
- Infinite lease left behind blocks deletes.
- ETag also changes with metadata/tier? (metadata yes; verify tier change behavior in docs).

**Cheat sheet**
- If-Match = update if unchanged.
- If-None-Match:* = create only.
- Lease 15-60s or infinite.
- Pass lease ID on writes.
- 412 = condition failed.

---

## Topic 9: Metadata vs index tags

**Study notes**

| | Metadata | Blob index tags |
|---|---|---|
| Searchable | No (must list + fetch) | Yes (service-side query across account) |
| Limits | ~8 KB total name-value | Up to 10 tags per blob; key 1-128, value 0-256 chars |
| Auth for read | Same as blob read | Separate permissions (Tags actions) |
| Cost | Free-ish | Indexed - billed (verify) |
| Use | descriptive info returned with blob | find/classify, lifecycle filters, ABAC |

Tags are eventually consistent in the index (small delay). Metadata keys must be valid C# identifiers; case preserved but treated case-insensitively.

### 9.1 [Easy] What is the difference between blob metadata and tags?
**Answer**
Metadata is user-defined name/value data stored with the blob that you can only read by fetching the blob (or listing with metadata); index tags are key/value pairs that the service indexes so you can query blobs by tag across containers.
**Explanation**
Use metadata for descriptive attributes; use tags for "find all blobs where status = 'pending'". Tags can also drive lifecycle rules and RBAC conditions.
**Code / CLI / Bicep**
```csharp
await blob.SetMetadataAsync(new Dictionary<string,string> { ["author"] = "arjun" });
await blob.SetTagsAsync(new Dictionary<string,string> { ["status"] = "pending", ["tenant"] = "contoso" });
```

### 9.2 [Follow-up] How do you find blobs by tag?
**Answer**
`BlobServiceClient.FindBlobsByTagsAsync` with a filter expression such as `"status" = 'pending' AND "tenant" = 'contoso'`; you can also scope to a container with `BlobContainerClient`.
**Explanation**
Tag names are quoted in the expression. Results are the matching blob names only. The caller needs the Storage Blob Data Owner role or the Blob index find permission (`Microsoft.Storage/storageAccounts/blobServices/containers/blobs/filter/action`) - verify role mapping in docs.
**Code / CLI / Bicep**
```csharp
await foreach (TaggedBlobItem item in svc.FindBlobsByTagsAsync("\"status\" = 'pending'"))
    Console.WriteLine($"{item.BlobContainerName}/{item.BlobName}");
```
```bash
az storage blob filter --account-name acct --tag-filter "\"status\"='pending'" --auth-mode login
```

### 9.3 [Medium] Can you use tags as a work queue (status = pending -> processing -> done)?
**Answer**
It works for low-volume cases, but it's a poor queue: no atomic claim or visibility timeout, and the index is eventually consistent. Use Service Bus/Queue for work distribution and tags just for classification.
**Explanation**
Two workers can both see "pending" and race. You can mitigate by conditional `SetTags` with `IfTags`/ETag conditions (`BlobRequestConditions.TagConditions`), turning the claim into compare-and-swap, but throughput and latency remain limited.
**Code / CLI / Bicep**
```csharp
try
{
    await blob.SetTagsAsync(new Dictionary<string,string>{["status"]="processing"},
        new BlobRequestConditions { TagConditions = "\"status\" = 'pending'" });
}
catch (RequestFailedException ex) when (ex.Status == 412) { /* someone else claimed it */ }
```

### 9.4 [Tricky] You set a tag and immediately query it, yet nothing returns. Why?
**Answer**
The tag index is eventually consistent, so there's a short lag between SetTags and its visibility to Find Blobs by Tags. Reading `GetTagsAsync` on the blob itself is immediately consistent.
**Explanation**
Don't use Find-by-tags as a read-your-writes mechanism. Also, tags set on upload via `BlobUploadOptions.Tags` are indexed the same way; and tag operations change the blob's tag state not its ETag (verify) - so don't rely on ETag for tag conflicts, use tag conditions.
**Code / CLI / Bicep**
```csharp
var tags = (await blob.GetTagsAsync()).Value.Tags; // immediate, per-blob
```
**Gotchas / follow-ups**
- Tags are not supported on HNS-enabled accounts in some configurations (verify).
- Tag values are visible to anyone with tag read permission; don't store secrets.

**Interview traps**
- Metadata isn't searchable.
- 10-tag limit.
- Tags aren't a queue.
- Eventual consistency of the index.

**Cheat sheet**
- Metadata = descriptive, tags = searchable.
- FindBlobsByTagsAsync.
- TagConditions for CAS.
- Tags can filter lifecycle rules.

---

## Topic 10: Data protection (versioning, soft delete, snapshots, PITR, change feed, immutability)

**Study notes**

| Feature | What it protects | Notes |
|---|---|---|
| Container soft delete | deleted containers | retention 1-365 days |
| Blob soft delete | deleted/overwritten blobs (without versioning) | retention days; undelete |
| Blob versioning | every overwrite/delete creates a previous version | auto, billed for stored versions |
| Snapshot | manual point-in-time read-only copy | billed on changed blocks |
| Change feed | ordered log of create/modify/delete events | stored in `$blobchangefeed` container |
| Point-in-time restore (PITR) | restore block blobs to a past time | requires versioning + change feed + soft delete; GPv2 only |
| Immutability (WORM) | time-based retention / legal hold | version-level or container-level policies |

Geo-replication does not protect from deletes - it replicates them.

### 10.1 [Easy] How do you recover an accidentally deleted blob?
**Answer**
If blob soft delete (or versioning) is enabled, undelete the blob or restore the previous version within the retention period. Without them, deletion is permanent.
**Explanation**
Enable soft delete for blobs and containers on every production account as baseline (e.g., 14-30 days). Soft-deleted data is billed until retention expires. With versioning, delete makes the current version a previous version.
**Code / CLI / Bicep**
```csharp
await blob.UndeleteAsync(); // soft-deleted blob
```
```bash
az storage account blob-service-properties update -n acct -g rg \
  --enable-delete-retention true --delete-retention-days 14 \
  --enable-container-delete-retention true --container-delete-retention-days 14 \
  --enable-versioning true
```

### 10.2 [Follow-up] Versioning vs snapshots - what's the difference?
**Answer**
Versioning is automatic: every write/delete preserves the prior state as a version with a version ID; snapshots are manually created, read-only point-in-time copies identified by timestamp. Versioning is the better default for protection against overwrites.
**Explanation**
Both bill only for unique blocks/changed data. Versions accumulate - pair with lifecycle rules to tier/delete old versions. Soft delete with versioning enabled keeps deleted versions too. You can promote an old version by copying it over the base blob.
**Code / CLI / Bicep**
```csharp
await foreach (var item in container.GetBlobsAsync(BlobTraits.None, BlobStates.Version, prefix: "a.pdf"))
    Console.WriteLine($"{item.VersionId} current={item.IsLatestVersion}");

var old = blob.WithVersion(versionId);
await blob.StartCopyFromUriAsync(old.Uri); // restore (needs auth on source; same account OK with same creds)
```

### 10.3 [Medium] How does point-in-time restore work and what are its prerequisites?
**Answer**
PITR restores block blobs in containers (all or selected ranges) to a previous time by replaying the change feed. It requires soft delete, change feed and blob versioning enabled, and a restore window shorter than soft-delete retention.
**Explanation**
Only block blobs are restored; page/append are not. It operates on the whole account's containers (with optional blob ranges). Restore is asynchronous. Verify limits in current docs (e.g., not supported with HNS).
**Code / CLI / Bicep**
```bash
az storage account blob-service-properties update -n acct -g rg \
  --enable-restore-policy true --restore-days 7 --enable-change-feed true \
  --enable-versioning true --enable-delete-retention true --delete-retention-days 14
az storage blob restore -n acct -g rg --time-to-restore 2026-10-15T10:00:00Z   # verify args in docs
```

### 10.4 [Hard] How does immutable storage (WORM) work, and what are the pitfalls?
**Answer**
Time-based retention policies make blobs non-modifiable and non-deletable for N days; legal holds block changes until cleared; both can be set at container or version level. A *locked* time-based policy cannot be shortened or removed.
**Explanation**
Used for SEC 17a-4 / regulatory compliance. Container-level WORM requires policy on the container; version-level WORM requires versioning and the account setting. A policy in unlocked state is for testing; locking is irreversible (you can only extend). Cost: you keep paying for storage until retention ends; even deleting the account is blocked.
**Code / CLI / Bicep**
```bash
az storage container immutability-policy create --account-name acct -c legal --period 365
az storage container immutability-policy lock --account-name acct -c legal --if-match <etag>
az storage container legal-hold set --account-name acct -c legal --tags case123
```
**Gotchas / follow-ups**
- Lock policies only after verifying in a non-prod account.
- Legal holds use tags; one hold can be cleared independently of retention.
- Immutability + lifecycle delete: delete actions fail until retention ends.

### 10.5 [Tricky] A bad deploy overwrote 50,000 blobs 3 hours ago; versioning was on but not PITR. What do you do?
**Answer**
Script restoring each blob's previous version (list versions, copy the last good version over the current), since versioning keeps pre-overwrite states; if PITR/change feed was enabled you could do a bulk restore instead.
**Explanation**
Copy within the same account is a server-side operation and fast. Use parallelism and, if ETag/version conditions matter, copy by version ID. Afterward, enable PITR and test restores - recovery needs to be rehearsed. Note that soft delete alone (without versioning) only preserves snapshots of overwritten blobs ("soft deleted snapshots") - verify behavior in docs.
**Code / CLI / Bicep**
```csharp
await Parallel.ForEachAsync(names, new ParallelOptions { MaxDegreeOfParallelism = 16 }, async (n, ct) =>
{
    var versions = new List<BlobItem>();
    await foreach (var v in container.GetBlobsAsync(BlobTraits.None, BlobStates.Version, n, ct))
        versions.Add(v);
    var target = versions.Where(v => v.IsLatestVersion != true && v.VersionId != null)
                         .OrderByDescending(v => v.VersionId).First(); // pick a version older than the incident
    var cur = container.GetBlobClient(n);
    await cur.StartCopyFromUriAsync(cur.WithVersion(target.VersionId).Uri, cancellationToken: ct);
});
```
**Gotchas / follow-ups**
- Restoring creates another version (storage grows).
- Pick the version by timestamp (`target.Properties.CreatedOn`/version time), not just VersionId ordering.
- Test in non-prod first.

**Interview traps**
- Geo-redundancy is not backup.
- Soft delete retention costs money.
- Locked immutability can't be undone.
- PITR needs three prerequisites and covers block blobs only.

**Cheat sheet**
- Baseline: soft delete + versioning (+ container soft delete).
- PITR = versioning + change feed + soft delete.
- Snapshot = manual.
- WORM = retention + legal hold; lock is irreversible.
- Lifecycle to trim old versions.

---

## Topic 11: Events and reacting to uploads

**Study notes**

Two ways to react to blob creation in Azure Functions:

| | Blob trigger (polling/classic) | Blob trigger with `Source = EventGrid` | Event Grid trigger |
|---|---|---|---|
| Mechanism | host scans container + blob receipts | Event Grid push, host reads blob | Event Grid push, you handle the event payload |
| Latency | can be minutes on Consumption; scans for large containers | seconds | seconds |
| Scale | poor for huge containers | good | good |
| Reliability | at-least-once, receipts in `azure-webjobs-hosts` | at-least-once | retry + dead-letter |
| Notes | classic | recommended for modern setups (verify support per model/extension) | most flexible |

Event Grid `Microsoft.Storage.BlobCreated` (also BlobDeleted, BlobTierChanged, etc.). Fires after `PutBlob`, `PutBlockList`, `CopyBlob`, `FlushWithClose` (HNS). Events are at-least-once, not guaranteed order. System topic on the storage account; filter by subject prefix/suffix (`/blobServices/default/containers/uploads/blobs/`).

### 11.1 [Easy] How can a Function react when a file is uploaded?
**Answer**
Use a Blob trigger or, better, an Event Grid subscription on `BlobCreated` that triggers the Function. Event Grid pushes within seconds, whereas the classic blob trigger polls.
**Explanation**
Event payload has `data.url`, `data.contentLength`, `data.eTag`, `data.api`. You then use managed identity to read the blob.
**Code / CLI / Bicep**
```csharp
[Function("OnBlobCreated")]
public async Task Run([EventGridTrigger] EventGridEvent ev)
{
    var data = ev.Data!.ToObjectFromJson<StorageBlobCreatedEventData>();
    var blob = new BlobClient(new Uri(data.Url), new DefaultAzureCredential());
    // process
}
```

### 11.2 [Follow-up] Why is the classic Blob trigger sometimes slow or missing files?
**Answer**
It relies on periodic scans of the container (log-based scanning for Standard accounts is best-effort), so on Consumption plans latency can reach minutes, and for very large containers scanning is costly. Event Grid-based triggers avoid this.
**Explanation**
The host tracks processed blobs via receipts in the `azure-webjobs-hosts` container using ETags. Blobs modified later are reprocessed if ETag differs. Also, Consumption plan with cold start adds delay; Premium/Dedicated with always-on helps. Verify current docs on Flex Consumption/Event Grid source.
**Code / CLI / Bicep**
```csharp
[Function("Blob")]
public void Run([BlobTrigger("uploads/{name}", Source = BlobTriggerSource.EventGrid, Connection = "StorageConn")] Stream blob, string name) { }
```

### 11.3 [Medium] Create an Event Grid subscription for BlobCreated filtered to one container.
**Answer**
Create a system topic on the storage account and an event subscription with included event type `Microsoft.Storage.BlobCreated`, subject-begins-with the container path, and an endpoint (Function, Service Bus queue, Storage queue, webhook).
**Explanation**
Delivering to Service Bus gives buffering and decoupling; with a Function endpoint you rely on Event Grid retries (24 hours/30 attempts defaults, verify) plus a dead-letter container. Using a managed identity for delivery requires role assignment on the destination (e.g., Azure Service Bus Data Sender).
**Code / CLI / Bicep**
```bicep
resource topic 'Microsoft.EventGrid/systemTopics@2022-06-15' = {
  name: 'st-blob'
  location: location
  properties: { source: sa.id, topicType: 'Microsoft.Storage.StorageAccounts' }
}
resource sub 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2022-06-15' = {
  parent: topic
  name: 'uploads-to-sb'
  properties: {
    destination: {
      endpointType: 'ServiceBusQueue'
      properties: { resourceId: queue.id }
    }
    filter: {
      includedEventTypes: ['Microsoft.Storage.BlobCreated']
      subjectBeginsWith: '/blobServices/default/containers/uploads/blobs/'
    }
  }
}
```

### 11.4 [Hard] Duplicate or out-of-order events, and large files that fire early - how do you design for it?
**Answer**
Assume at-least-once and unordered delivery: make handlers idempotent (key on blob URL + ETag), and don't assume the content is final until the BlobCreated event; for chunked uploads it fires after the commit, not per block.
**Explanation**
Overwriting a blob generates another BlobCreated. Direct `Put Blob` events fire once complete. For HNS accounts SFTP/`FlushWithClose` produce events when the file closes; multiple "created" events can happen (CreateFile then Flush) so filter on `data.api`. Buffer through Service Bus with duplicate detection/sessions if ordering per entity matters.
**Code / CLI / Bicep**
```csharp
string key = $"{data.Url}|{data.ETag}";
if (!await dedupeStore.TryClaimAsync(key)) return; // already handled
```
**Gotchas / follow-ups**
- Event Grid validation handshake required for webhooks.
- Filter `data.api` in (`PutBlob`, `PutBlockList`, `CopyBlob`, `FlushWithClose`).
- Event size doesn't include the content; you must read it.

### 11.5 [Tricky] "Blobs not triggering my Function" - checklist?
**Answer**
Check: trigger type and Connection setting, MI/role on host storage, Event Grid subscription provisioning state/filters/validation, dead-letter, function app is running (Consumption cold, disabled function), blob in a container matching path, and receipts (`azure-webjobs-hosts`) if it was already processed.
**Explanation**
Practical order: (1) Event Grid metrics: delivered/failed/dropped; (2) Function invocations in App Insights; (3) filter mismatch (subject case, trailing slash); (4) network: firewall blocking Event Grid delivery - use private endpoint with trusted services/`AllowTrustedMicrosoftServices` and Event Grid managed identity; (5) the blob was uploaded using a path the trigger pattern doesn't match.
**Code / CLI / Bicep**
```bash
az eventgrid system-topic event-subscription show -g rg --system-topic-name st-blob -n uploads-to-sb \
  --query "{state:provisioningState,filter:filter}"
```
```kusto
requests | where timestamp > ago(1h) and name == "OnBlobCreated" | summarize count() by resultCode
```
**Gotchas / follow-ups**
- Overwriting a blob with the same ETag content doesn't re-trigger polling trigger.
- Poison blobs: classic trigger retries 5 times then writes to `webjobs-blobtrigger-poison` queue.

**Interview traps**
- Blob trigger is not real-time on Consumption.
- Event Grid is at-least-once; no ordering.
- Event doesn't contain the file.
- Firewall silently blocks delivery if not configured.
- Event can arrive before reads are consistent? (reads after commit are strongly consistent; the real risk is partial uploads by non-atomic clients on HNS).

**Cheat sheet**
- Prefer Event Grid (trigger or Source=EventGrid).
- BlobCreated -> filter by subject/api.
- Idempotency on URL+ETag.
- Poison queue for classic trigger.
- Deliver to Service Bus for buffering.

---

## Topic 12: File ingestion pipeline, idempotency and dedupe

**Study notes**

```
Producer --(SAS/MI)--> Blob (inbox/)
                          | BlobCreated
                     Event Grid (system topic)
                          | 
                    Service Bus queue  (buffer, DLQ, retries, sessions)
                          |
                    Function / worker  -> validate -> process -> write result
                          |
             move to processed/ (copy + delete) or tag status=done; failures -> quarantine/
```
Key design decisions: claim (lock) strategy, idempotent processing, archive vs delete, poison handling, observability (correlation ID = blob name/etag), and cleaning inbox. Blob "move" is copy + delete (not atomic) except in HNS accounts where rename is atomic.

### 12.1 [Easy] Describe a standard blob ingestion pipeline on Azure.
**Answer**
Upload lands in an inbox container; Event Grid emits BlobCreated to Service Bus (or a Function); a worker processes the file; then it moves the file to processed/archive or quarantine and records status. Service Bus gives buffering, retries and a dead-letter queue.
**Explanation**
Decoupling with Service Bus protects from bursts and lets you scale consumers. Use Entra auth between all hops; correlation using blob URL/ETag in logs.
**Code / CLI / Bicep**
```csharp
[Function("Ingest")]
public async Task Run([ServiceBusTrigger("blob-events", Connection = "SbConn")] ServiceBusReceivedMessage msg)
{
    var ev = EventGridEvent.Parse(msg.Body);
    var data = ev!.Data!.ToObjectFromJson<StorageBlobCreatedEventData>();
    // process blob at data.Url
}
```

### 12.2 [Follow-up] How do you "move" a blob to a processed container?
**Answer**
Start a server-side copy to the destination and, once complete, delete the source. There's no native atomic move on flat namespace blobs (HNS has atomic rename).
**Explanation**
Same-account copies are usually near-instant (`StartCopyFromUriAsync` then poll `CopyStatus`, or `WaitForCompletionAsync`). Cross-account needs source auth. Order: copy -> verify (ETag/length/MD5) -> delete. If the process crashes between copy and delete you have duplicates; design retries to be safe.
**Code / CLI / Bicep**
```csharp
var dst = processed.GetBlobClient(src.Name);
var op = await dst.StartCopyFromUriAsync(src.Uri);
await op.WaitForCompletionAsync();
await src.DeleteIfExistsAsync(DeleteSnapshotsOption.IncludeSnapshots);
```

### 12.3 [Medium] How do you make processing idempotent and de-duplicate files?
**Answer**
Derive an idempotency key (blob name + ETag, or a content hash/MD5) and record it in a store (Table/Cosmos/SQL, or blob tags with conditions) before doing side effects; skip if already completed. Use conditional operations as the atomic claim.
**Explanation**
Blob name alone fails if producers reuse names; ETag changes with each overwrite so name+ETag identifies a version; `ContentHash` (MD5) is only set if the uploader provided it (for block blobs uploaded in multiple blocks, service may not compute it - verify), so compute SHA-256 yourself if dedupe-by-content matters. Service Bus duplicate detection (MessageId = ETag-based) adds a window-based dedupe.
**Code / CLI / Bicep**
```csharp
var props = await blob.GetPropertiesAsync();
string key = $"{blob.Name}:{props.Value.ETag}";
var marker = statusContainer.GetBlobClient($"done/{Uri.EscapeDataString(key)}");
try
{
    await marker.UploadAsync(BinaryData.FromString("claimed"),
        new BlobUploadOptions { Conditions = new BlobRequestConditions { IfNoneMatch = ETag.All } });
}
catch (RequestFailedException ex) when (ex.Status is 409 or 412) { return; }
```

### 12.4 [Hard] The consumer reads a blob while the producer is still uploading / overwriting. How do you avoid partial reads?
**Answer**
Block blobs are atomic from the reader's view: they appear only after commit, so Event Grid BlobCreated implies a complete blob. Partial data risk comes from overwrites during read (use `OpenReadAsync`, which pins to the ETag and fails with 412 if modified) or from producers writing temp names then renaming non-atomically.
**Explanation**
Convention: producers upload to a temp prefix and then copy/rename to the final location (or in HNS, atomic rename) -- or use a "manifest/ready" marker blob. Process by passing `IfMatch = etag` from the event so you process exactly the version that was announced.
**Code / CLI / Bicep**
```csharp
var resp = await blob.DownloadStreamingAsync(new BlobDownloadOptions
{ Conditions = new BlobRequestConditions { IfMatch = new ETag(data.ETag) } });
```
**Gotchas / follow-ups**
- If condition fails, a newer BlobCreated is coming; dead-letter the stale one or skip.
- Verify event `data.eTag` is the post-write ETag.

### 12.5 [Tricky] Your pipeline processed the same file twice and produced duplicate DB rows. Root causes and fixes?
**Answer**
At-least-once delivery from Event Grid/Service Bus (redelivery after lock expiry, crash before complete, overwritten blob re-fires event). Fix by making the DB write idempotent (unique key/upsert on blob URL+ETag), completing messages after durable success, and tuning lock duration.
**Explanation**
Service Bus lock expiry during long processing causes redelivery to another instance while the first continues; use auto lock renewal (`MaxAutoLockRenewalDuration`). Also a copy+delete move can re-trigger BlobCreated in the destination if its container is also subscribed - scope event filters by subject.
**Code / CLI / Bicep**
```sql
MERGE dbo.Imports AS t USING (SELECT @BlobUrl AS BlobUrl, @ETag AS ETag) s
ON t.BlobUrl = s.BlobUrl AND t.ETag = s.ETag
WHEN NOT MATCHED THEN INSERT (BlobUrl, ETag, ProcessedAt) VALUES (@BlobUrl, @ETag, SYSUTCDATETIME());
```
**Gotchas / follow-ups**
- Moving into `processed/` of the same container may fire another event; filter by prefix.
- Order of "mark done" vs "move" affects failure recovery.

**Interview traps**
- Assuming exactly-once.
- "Move" is not atomic.
- MD5 may be absent.
- Event filter loops (processed container re-triggering).
- Processing the latest blob instead of the announced ETag.

**Cheat sheet**
- Inbox -> Event Grid -> Service Bus -> worker -> processed/quarantine.
- Key = name + ETag (or SHA-256).
- Use IfNoneMatch=* / IfMatch for claims.
- Copy -> verify -> delete.
- Upsert in downstream DB.

---

## Topic 13: Static website, CDN / Front Door, client-direct uploads and CORS

**Study notes**

Static website hosting: enable on a GPv2 (or premium block blob) account; content served from the `$web` container via the web endpoint `https://<acct>.z<N>.web.core.windows.net`. Configure index document and 404 document. Only public/anonymous access; custom domains with HTTPS require CDN or Front Door (the web endpoint itself supports the custom domain over HTTP only via direct mapping; verify). For SPA routing, set error document to `index.html`.

CORS is a **browser** control configured per storage service (Blob/Queue/Table/File): allowed origins, methods, headers, exposed headers, max-age. It does not affect server-to-server calls and is not authorization.

```
Browser --1--> API (auth user) --2--> returns user-delegation SAS URL
Browser --3--> PUT blocks directly to Blob (CORS must allow origin + PUT + x-ms-* headers)
Blob --4--> Event Grid -> processing
```

### 13.1 [Easy] How do you host a static site on Blob storage?
**Answer**
Enable the static website feature, upload content to the `$web` container, and set index/error documents; the site is served from the account's web endpoint.
**Explanation**
Cheap and scalable; add Azure Front Door or CDN for custom domain, TLS, caching and WAF. It only serves static files.
**Code / CLI / Bicep**
```bash
az storage blob service-properties update --account-name acct --static-website \
  --index-document index.html --404-document index.html --auth-mode login
az storage blob upload-batch --account-name acct -d '$web' -s ./dist --auth-mode login
```

### 13.2 [Follow-up] Why put Front Door/CDN in front, and what changes?
**Answer**
For custom domain + managed TLS, edge caching/lower latency, WAF and routing, and to keep the storage origin protected. Set cache-control headers on blobs and purge cache after deployments.
**Explanation**
Origin is the static website endpoint (not the blob endpoint) so index/404 handling works. With Front Door Premium you can reach a private-endpoint origin via Private Link (verify). Stale caches after deploys are common: version file names or purge.
**Code / CLI / Bicep**
```bash
az storage blob upload-batch --account-name acct -d '$web' -s ./dist --auth-mode login \
  --content-cache-control "public, max-age=31536000, immutable" --pattern "assets/*"
az afd endpoint purge -g rg --profile-name afd --endpoint-name site --content-paths "/index.html" --domains www.contoso.com
```

### 13.3 [Medium] Browser-direct uploads: how do you implement them securely?
**Answer**
The API authenticates the user, then issues a short-lived user delegation SAS scoped to one new blob name (Create/Write permission only); the browser uploads directly, and the Blob event triggers processing. Configure CORS for your site origin only.
**Explanation**
Server picks the blob name (GUID/user prefix) to avoid overwrites and path abuse; never allow client-chosen container/blob. Enforce size limits by validating after upload (SAS can't cap size) via event handler, delete if oversize/invalid. Optionally upload to a quarantine container and scan with Defender malware scanning (verify) before promoting.
**Code / CLI / Bicep**
```csharp
app.MapPost("/upload-url", async (BlobServiceClient svc, ClaimsPrincipal user) =>
{
    var name = $"{user.FindFirstValue("oid")}/{Guid.NewGuid()}";
    var expiry = DateTimeOffset.UtcNow.AddMinutes(10);
    var key = await svc.GetUserDelegationKeyAsync(null, expiry);
    var b = new BlobSasBuilder(BlobSasPermissions.Create | BlobSasPermissions.Write, expiry)
    { BlobContainerName = "quarantine", BlobName = name, Protocol = SasProtocol.Https };
    var uri = new BlobUriBuilder(svc.GetBlobContainerClient("quarantine").GetBlobClient(name).Uri)
    { Sas = b.ToSasQueryParameters(key, svc.AccountName) }.ToUri();
    return Results.Ok(new { uri });
});
```

### 13.4 [Hard] Large file chunking from the browser and resuming
**Answer**
Use Put Block with base64 block IDs of equal length from the client (e.g., 4-8 MiB chunks, a few in parallel) and a final Put Block List; a failed chunk can be retried independently, and a resume lists uncommitted blocks.
**Explanation**
The JS SDK `@azure/storage-blob` `uploadData`/`uploadBrowserData` does this for you (with `blockSize`/`concurrency`). For resume, `getBlockList(‘uncommitted’)` tells which blocks exist; uncommitted blocks expire after about a week. The SAS needs Write permission (and Create for new). Max 50,000 blocks per blob so block size sets max file size.
**Code / CLI / Bicep**
```bash
# raw REST shape (illustrative): stage a block, then commit
curl -X PUT "$SAS_URL&comp=block&blockid=$(printf '%06d' 1 | base64)" -H "x-ms-version: 2023-11-03" --data-binary @chunk1
```
**Gotchas / follow-ups**
- Block ID must be base64 and same length for all blocks in a blob.
- Block size too small + big file = >50k blocks.
- Commit overrides all content: a replayed commit can overwrite.

### 13.5 [Tricky] Browser upload fails with "blocked by CORS policy" though SAS is valid. Why?
**Answer**
CORS is evaluated on the preflight OPTIONS request: the account's CORS rule for the Blob service must list the exact origin, the method (PUT, OPTIONS handled by service), and the headers the client sends (`x-ms-blob-type`, `x-ms-version`, `content-type`...). Any mismatch fails regardless of SAS.
**Explanation**
Common mistakes: wildcard missing for `x-ms-*` headers, scheme/port mismatch in origin, setting CORS on the wrong service (e.g., queue), or on the wrong account. Also a Front Door/CDN in between may strip/alter headers. Check "Exposed headers" for reading `ETag` or `x-ms-request-id` in JS.
**Code / CLI / Bicep**
```bash
az storage cors add --services b --methods GET PUT OPTIONS HEAD --origins https://app.contoso.com \
  --allowed-headers "*" --exposed-headers "*" --max-age 3600 --account-name acct --auth-mode login
```
**Gotchas / follow-ups**
- CORS isn't security; a non-browser client ignores it.
- Wildcard origin `*` with SAS is risky.

**Interview traps**
- CORS != authorization.
- Static website custom-domain HTTPS needs CDN/Front Door.
- SAS can't limit file size.
- Browser uploads to blob endpoint need `x-ms-blob-type: BlockBlob` header for Put Blob.

**Cheat sheet**
- $web + web endpoint.
- Front Door/CDN for TLS/domain/WAF.
- API issues user delegation SAS, Create/Write only, 10-15 min.
- CORS: origin, methods, x-ms-* headers.
- Chunk via Put Block + Put Block List.

---

## Topic 14: Network security and encryption

**Study notes**

Layers of defence:
```
Identity (Entra RBAC) -> Network (firewall/VNet rules/private endpoint) -> Encryption (at rest + in transit) -> Threat protection (Defender)
```
- **Firewall**: default action Deny + allow selected VNets (service endpoints), IP ranges (public IPv4 only, no private IPs), resource instance rules, and exceptions ("Allow trusted Azure services").
- **Private endpoint**: private IP in your VNet per sub-resource (`blob`, `dfs`, `file`, ...); needs Private DNS zone `privatelink.blob.core.windows.net`.
- Encryption at rest: always on, Microsoft-managed keys (AES-256) by default; optional customer-managed keys (CMK) in Key Vault/Managed HSM with a UAMI/SAMI accessing the key; infrastructure (double) encryption optional; encryption scopes per container/blob.
- In transit: `supportsHttpsTrafficOnly` (secure transfer required) and minimum TLS 1.2.
- Microsoft Defender for Storage: malware scanning, sensitive data threat detection, anomalous access alerts (verify pricing/features).

### 14.1 [Easy] How do you restrict Blob storage to a private network?
**Answer**
Set the firewall default action to Deny and allow specific VNets/IPs, or better create a private endpoint for the blob sub-resource and disable public network access. Private DNS must resolve the account to the private IP.
**Explanation**
Private endpoints give a private IP and let you turn public access off entirely. Service endpoints keep traffic on Azure backbone but use the public IP. On-prem users need DNS forwarding to the Azure private zone.
**Code / CLI / Bicep**
```bicep
resource pe 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-blob'
  location: location
  properties: {
    subnet: { id: subnetId }
    privateLinkServiceConnections: [{
      name: 'blob'
      properties: { privateLinkServiceId: sa.id, groupIds: ['blob'] }
    }]
  }
}
```
```bash
az storage account update -n acct -g rg --default-action Deny --public-network-access Disabled
```

### 14.2 [Follow-up] How do customer-managed keys work with managed identity?
**Answer**
The storage account uses a managed identity (user-assigned or system-assigned) with access to a Key Vault key (wrap/unwrap permissions) to protect the account's encryption keys. You can rotate the key version (auto-rotation if you omit version).
**Explanation**
Key Vault must have soft-delete and purge protection. If the identity loses access or the key is deleted/disabled, the account becomes inaccessible - a classic outage. Use a UAMI so permissions can be granted before the account is created. Role: Key Vault Crypto Service Encryption User (RBAC) (verify).
**Code / CLI / Bicep**
```bicep
properties: {
  encryption: {
    keySource: 'Microsoft.Keyvault'
    identity: { userAssignedIdentity: uami.id }
    keyvaultproperties: { keyname: 'cmk', keyvaulturi: kv.properties.vaultUri }
    services: { blob: { enabled: true, keyType: 'Account' } }
  }
}
```

### 14.3 [Medium] What does "Allow trusted Azure services" do and what doesn't it cover?
**Answer**
It lets specific Microsoft services (e.g., Backup, Event Grid with managed identity, Azure Monitor, some others) bypass the firewall with their own authorized identity. It does NOT allow arbitrary Azure resources such as your Function or Logic App - those need VNet integration + service endpoint/private endpoint or resource instance rules.
**Explanation**
Resource instance rules allow named resource types from a tenant/subscription (e.g., a specific Synapse workspace). Functions on Consumption have shared outbound IPs; adding those to the IP allow list is brittle. Use VNet integration (Premium/Flex/Dedicated) with private endpoint instead.
**Code / CLI / Bicep**
```bash
az storage account update -n acct -g rg --bypass AzureServices Logging Metrics
az storage account network-rule add -g rg --account-name acct --vnet-name vnet --subnet snet-func
```

### 14.4 [Hard] You get 403 AuthorizationFailure after deploy even though RBAC is correct. Debug.
**Answer**
`AuthorizationFailure` ("This request is not authorized to perform this operation") usually means the network layer blocked you: the firewall denies the caller's IP/subnet, the private DNS resolves to the public IP, or a private endpoint isn't in the right VNet. RBAC problems more often say `AuthorizationPermissionMismatch`.
**Explanation**
Steps: (1) check error code in the response/Storage logs (`StatusText`, `CallerIpAddress`); (2) `nslookup acct.blob.core.windows.net` from the caller - private IP expected; (3) review firewall rules and "Public network access"; (4) check VNet integration/route (WEBSITE_VNET_ROUTE_ALL / route all) for Function apps so traffic goes via VNet; (5) NSG/UDR; (6) cross-subscription private endpoint approvals. A new deployment may have changed outbound IPs or removed the subnet delegation.
**Code / CLI / Bicep**
```kusto
StorageBlobLogs
| where TimeGenerated > ago(1h) and StatusCode == 403
| project TimeGenerated, OperationName, StatusText, CallerIpAddress, AuthenticationType, Uri
| order by TimeGenerated desc
```
**Gotchas / follow-ups**
- Portal data browsing fails when your client IP isn't allowed, even if you are Owner.
- Firewall changes take up to a few minutes to propagate.
- Storage firewall rules don't apply to requests from the same region in some legacy cases? (verify) - don't rely on it.

### 14.5 [Tricky] After enabling private endpoint, on-prem and some Azure services can't reach the account. Why?
**Answer**
Name resolution: clients must resolve the account FQDN (via the `privatelink` CNAME) to the private IP; if public access is disabled and DNS still returns the public IP you get 403. Also other Azure services connecting from outside your VNet are blocked unless exceptions/their own private endpoints are configured.
**Explanation**
Fix by linking Private DNS zones to all relevant VNets, conditional forwarders from on-prem DNS to Azure DNS (168.63.129.16 via a resolver), and per-service managed private endpoints (e.g., Data Factory/Synapse). Also create separate private endpoints for `dfs` when using ADLS Gen2 and `web` for static sites.
**Code / CLI / Bicep**
```bash
nslookup acct.blob.core.windows.net
# expect: acct.privatelink.blob.core.windows.net -> 10.x.x.x
```
**Gotchas / follow-ups**
- Each sub-resource (blob, dfs, queue, table, file, web) needs its own endpoint + DNS zone.
- Private endpoint in another region is possible but mind latency.

**Interview traps**
- Trusted services != all Azure services.
- Firewall IP rules don't accept private IP ranges.
- CMK key loss = data unavailable.
- AuthorizationFailure (network) vs AuthorizationPermissionMismatch (RBAC).

**Cheat sheet**
- Private endpoint + disable public access + private DNS.
- Secure transfer + TLS 1.2.
- CMK needs soft delete + purge protection.
- Defender for Storage for malware/anomaly.
- 403 + AuthorizationFailure -> check network/DNS first.

---

## Topic 15: Performance, throttling and retries

**Study notes**

Scalability targets (per account; verify current numbers in docs): roughly 20,000 requests/second per standard account, ingress/egress in the tens of Gbps (region dependent), and per-blob target of ~500 requests/s / up to ~60 MiB/s throughput for a single block blob. Premium block blobs have higher request rates.

Throttling: HTTP **503 ServerBusy** (or 500 OperationTimedOut) means back off and retry. The SDK's default retry policy is exponential with jitter (defaults: 3 retries, ~0.8 s delay, max 1 minute; verify).

Partitioning: Storage partitions by account+container+blob name (range partitioning); sequential names (timestamps, incrementing IDs) concentrate load on one partition server, which can cause throttling at high scale. Add a hash prefix to distribute (though modern storage auto-balances, still a recommendation for extreme scale).

### 15.1 [Easy] What does a 503 ServerBusy from Blob storage mean?
**Answer**
You hit a scalability target (account, partition, or blob) and the service asks you to back off. Retry with exponential backoff and reduce request rate/concurrency.
**Explanation**
Check metrics `Transactions` by `ResponseType` (`ServerBusyError`, `ClientThrottlingError`). The SDK retries automatically a few times; persistent errors need load distribution or splitting accounts.
**Code / CLI / Bicep**
```csharp
var options = new BlobClientOptions
{
    Retry = { Mode = RetryMode.Exponential, MaxRetries = 6, Delay = TimeSpan.FromSeconds(1), MaxDelay = TimeSpan.FromSeconds(30), NetworkTimeout = TimeSpan.FromSeconds(100) }
};
```

### 15.2 [Follow-up] How do blob names affect throughput?
**Answer**
Range partitioning by name means lexically sequential names (e.g., `2025-10-16T10:00:01.log`) land on the same partition, limiting throughput; prefixing with a short hash/random value spreads load across partitions.
**Explanation**
Trade-off: hashed prefixes hurt listing by date. Choose by workload: for heavy parallel ingest use `hash/yyyy/MM/dd/...`; for browse-by-date use tag/metadata or an index. Azure also auto-splits hot ranges but with delay.
**Code / CLI / Bicep**
```csharp
static string Name(string id) =>
    $"{Convert.ToHexString(SHA1.HashData(Encoding.UTF8.GetBytes(id)))[..3]}/{DateTime.UtcNow:yyyy/MM/dd}/{id}";
```

### 15.3 [Medium] How do you maximise download/upload throughput for large blobs?
**Answer**
Parallelise ranges/blocks (`MaximumConcurrency`, larger `MaximumTransferSize`), use a region-collocated client, reuse connections, raise `ServicePointManager`/`HttpClient` limits as needed, and use AzCopy for bulk. Single-stream transfer is limited by per-blob targets.
**Explanation**
Check bottleneck: client NIC, CPU (MD5/CRC), disk, or service throttling. In Functions, outbound connection limits (Consumption: ~600 active connections per instance) and SNAT port exhaustion limit parallelism; reuse a singleton client.
**Code / CLI / Bicep**
```csharp
await blob.DownloadToAsync(path, new BlobDownloadToOptions
{
    TransferOptions = new StorageTransferOptions { MaximumConcurrency = 16, InitialTransferSize = 8 * 1024 * 1024, MaximumTransferSize = 8 * 1024 * 1024 }
});
```

### 15.4 [Hard] Latency is high only for small-object workloads; what do you check?
**Answer**
Per-request overhead dominates: check client-side (cold TLS handshakes, new clients per call, DNS), cross-region distance, Nagle/`Expect: 100-continue`, throttling, and consider Premium block blobs for single-digit-ms latency. Batch/bundle small objects where possible.
**Explanation**
Use `StorageBlobLogs` `DurationMs` (E2E) vs `ServerLatencyMs`: large gap implies client/network, small gap means service. Metrics `SuccessE2ELatency` vs `SuccessServerLatency` give the same insight.
**Code / CLI / Bicep**
```kusto
StorageBlobLogs
| where TimeGenerated > ago(1h) and OperationName == "GetBlob"
| summarize p50=percentile(DurationMs,50), p95=percentile(DurationMs,95), srv95=percentile(ServerLatencyMs,95) by bin(TimeGenerated, 5m)
```
**Gotchas / follow-ups**
- E2E - server latency large => client/network.
- Test in same region first.

### 15.5 [Tricky] Parallel uploads made things slower. Why?
**Answer**
Over-parallelism causes memory pressure (block size x concurrency x files), thread/connection pool starvation, SNAT exhaustion, and throttling, leading to retries that reduce effective throughput. Tune down and measure.
**Explanation**
Double parallelism (parallel files x per-file concurrency) multiplies. Use a bounded `Parallel.ForEachAsync` for files and moderate per-blob concurrency. Watch for `ClientThrottlingError`, GC time, and `TaskCanceledException` from NetworkTimeout.
**Code / CLI / Bicep**
```csharp
await Parallel.ForEachAsync(files, new ParallelOptions { MaxDegreeOfParallelism = 4 }, async (f, ct) =>
{
    await container.GetBlobClient(Path.GetFileName(f)).UploadAsync(f, new BlobUploadOptions
    { TransferOptions = new StorageTransferOptions { MaximumConcurrency = 4 } }, ct);
});
```
**Gotchas / follow-ups**
- Retries can re-send entire blocks.
- Verify the current defaults for retry/timeouts.

**Interview traps**
- 503 is not a bug - it's backpressure.
- Sequential names cause hot partitions.
- Creating a client per request.
- Not distinguishing E2E vs server latency.

**Cheat sheet**
- Backoff + jitter on 503/500.
- Hash-prefix names for extreme ingest.
- Concurrency x block size = memory.
- E2E vs server latency in logs.
- Premium block blob for low latency.

---

## Topic 16: Cost model

**Study notes**

Blob cost components (verify prices in current docs):
1. **Capacity** (GB-month) by tier and redundancy.
2. **Transactions** (per 10,000 ops) - write, read, list, other; cost per op grows as tier gets colder.
3. **Data retrieval** (per GB) for Cool/Cold/Archive.
4. **Egress** (data leaving the Azure region/to internet; ingress is free; inter-region replication has bandwidth costs).
5. **Features**: versions/snapshots/soft-delete stored data, index tags, last access time tracking, change feed, geo-replication data transfer, Defender for Storage, SFTP and HNS endpoints (hourly).
6. **Early deletion** and tier-change operations.

### 16.1 [Easy] What drives Blob storage cost?
**Answer**
Mainly stored GB by tier/redundancy, number and type of transactions, data retrieval for cooler tiers, and egress. Ingress is free.
**Explanation**
Redundancy multiplies capacity cost (GRS/GZRS more than LRS). Premium has higher GB price but lower transaction cost.
**Code / CLI / Bicep**
```bash
az consumption usage list --start-date 2026-09-01 --end-date 2026-09-30 \
  --query "[?contains(instanceName,'acct')].{m:meterName,c:pretaxCost}" -o table
```

### 16.2 [Follow-up] Why can millions of small files be expensive?
**Answer**
Each file requires separate transactions (create, list, read) and minimum-size/overhead per object; transaction cost can exceed capacity cost. Hot tier suits many small, frequently accessed objects better than cool/archive.
**Explanation**
Consider bundling (zip/Parquet/tar), batching writes (append blob or buffering), and using larger block sizes (fewer PutBlock calls per big blob). For listing-heavy workloads use blob inventory instead of repeated List Blobs.
**Code / CLI / Bicep**
```bicep
resource inv 'Microsoft.Storage/storageAccounts/inventoryPolicies@2023-01-01' = {
  parent: sa
  name: 'default'
  properties: { policy: { enabled: true, type: 'Inventory', rules: [{
    name: 'daily', enabled: true, destination: 'inventory'
    definition: { filters: { blobTypes: ['blockBlob'] }, format: 'Csv', schedule: 'Daily', objectType: 'Blob'
      schemaFields: ['Name','Content-Length','AccessTier','Last-Modified'] }
  }] } }
}
```

### 16.3 [Medium] Storage cost suddenly spiked. How do you investigate?
**Answer**
Cost analysis by meter (capacity vs transactions vs egress vs data retrieval), then Azure Monitor metrics (`UsedCapacity`, `Transactions` by `ApiName`), and logs (`StorageBlobLogs`) to find top callers/operations. Typical culprits: runaway list loops, polling blob triggers, lifecycle moves, versions/snapshots growth, soft-delete retention, or egress from a public SAS.
**Explanation**
Compare day-over-day; use diagnostic settings to Log Analytics for per-operation breakdowns. Check lifecycle policies (tier-change operations), a misconfigured loop that lists whole containers, or a leaked SAS used for downloads.
**Code / CLI / Bicep**
```kusto
StorageBlobLogs
| where TimeGenerated > ago(1d)
| summarize ops=count(), bytesOut=sum(ResponseBodySize) by OperationName, CallerIpAddress, AuthenticationType
| top 20 by ops desc
```

### 16.4 [Hard] Estimate cost-optimal tiering for 100 TB where 5% is read monthly.
**Answer**
Model each tier: storage cost + (reads x read-op cost) + (GB retrieved x retrieval cost) + tier-change/early-deletion costs, using current price sheets; usually Cool/Cold is best if access is infrequent and objects are large, while Hot stays cheaper for high read ratios. Use last-access-time-based lifecycle to tier automatically.
**Explanation**
Don't assume - compute break-even months for tier transitions: (Hot-Cool storage delta x months) vs (write/transition ops + retrieval). The Cool 30-day / Cold 90-day / Archive 180-day minimums mean transient data should not be tiered down. Use the Azure Pricing Calculator (verify prices).
**Code / CLI / Bicep**
```csharp
double hot = tb * hotPerGb * 1024 + reads * hotReadPer10k / 1e4;
double cool = tb * coolPerGb * 1024 + reads * coolReadPer10k / 1e4 + retrievedGb * coolRetrievalPerGb;
Console.WriteLine(hot < cool ? "Stay Hot" : "Move to Cool"); // plug in current prices
```
**Gotchas / follow-ups**
- Smart tiering/last access time tracking has its own cost.
- Cross-region reads add egress.

**Interview traps**
- Ingress free, egress not.
- Cold tiers have higher per-op cost.
- Versions and soft-deleted data are billed.
- Polling triggers & list loops generate silent transaction costs.

**Cheat sheet**
- Capacity + transactions + retrieval + egress + features.
- Inventory over repeated List.
- Bundle small files.
- Cost analysis -> metrics -> logs.
- Use lifecycle to age data.

---

## Topic 17: Queue/Table storage, Azure Files vs Blob vs Data Lake

**Study notes**

| Service | Model | Use when | Alternatives |
|---|---|---|---|
| Queue Storage | simple queue, <=64 KB messages, 7-day default TTL (configurable, up to infinite), at-least-once, no ordering/FIFO guarantee | cheap simple buffering, huge volume | Service Bus (ordering, sessions, topics, DLQ, transactions, duplicate detection, 256 KB-100 MB) |
| Table Storage | NoSQL key/attribute with PartitionKey+RowKey, cheap | simple lookups at low cost | Cosmos DB Table API / NoSQL for global distribution, indexing, SLAs |
| Azure Files | SMB/NFS/REST file shares | lift-and-shift apps, shared mounts | Blob for objects; Azure NetApp for high performance |
| Blob | objects over HTTP | unstructured data, app files | |
| Data Lake Gen2 | Blob + HNS + ACLs | analytics | |

Queue messages: default visibility timeout 30 s (up to 7 days), dequeue count for poison handling (the Functions Queue trigger moves to `<queue>-poison` after 5 attempts).

### 17.1 [Easy] Storage Queue vs Service Bus - when to choose which?
**Answer**
Storage Queue for simple, cheap, very large-scale buffering where ordering, DLQ, sessions and transactions are not needed; Service Bus for enterprise messaging: FIFO via sessions, topics/subscriptions, dead-letter, duplicate detection, scheduled delivery, larger messages.
**Explanation**
Storage queues have no built-in dead-letter queue (you implement via `DequeueCount`) and no topics. Service Bus costs more but offers richer semantics. Auth with Entra for both.
**Code / CLI / Bicep**
```csharp
var q = new QueueClient(new Uri("https://acct.queue.core.windows.net/work"), new DefaultAzureCredential());
await q.CreateIfNotExistsAsync();
await q.SendMessageAsync("hello");
QueueMessage[] msgs = (await q.ReceiveMessagesAsync(maxMessages: 10, visibilityTimeout: TimeSpan.FromMinutes(2))).Value;
foreach (var m in msgs) { /* process */ await q.DeleteMessageAsync(m.MessageId, m.PopReceipt); }
```

### 17.2 [Follow-up] Table Storage vs Cosmos DB?
**Answer**
Table Storage is a cheap key-value/entity store with PartitionKey+RowKey indexing only, single-region (with geo-replication), limited query features. Cosmos DB (NoSQL or Table API) offers global distribution, automatic indexing of all properties, SLAs, and richer querying at higher cost.
**Explanation**
Table query performance depends on PartitionKey/RowKey; non-key filters scan. Batch transactions are limited to one partition (up to 100 entities).
**Code / CLI / Bicep**
```csharp
var table = new TableClient(new Uri("https://acct.table.core.windows.net"), "Orders", new DefaultAzureCredential());
await table.CreateIfNotExistsAsync();
await table.UpsertEntityAsync(new TableEntity("cust1", "order1") { ["Total"] = 42.5 });
```

### 17.3 [Medium] Azure Files vs Blob vs Data Lake Gen2 - how do you choose?
**Answer**
Azure Files when apps need SMB/NFS file-system semantics (shared mounts, lift-and-shift); Blob for cloud-native object storage via HTTP/SDK (cheapest, tiering, events); Data Lake Gen2 when you need hierarchical namespace/ACLs/atomic directory ops for analytics. Blob and ADLS are the same service family with HNS toggled.
**Explanation**
Files has different pricing (provisioned for premium; pay-as-you-go for standard), snapshots/Azure Backup, and identity-based SMB auth (Entra Kerberos/AD DS). Blob events are available for blob only (Files has no Event Grid BlobCreated). Mounting Blob via NFS 3.0/BlobFuse is possible but with limitations (verify).
**Code / CLI / Bicep**
```bash
az storage share-rm create -g rg --storage-account acct -n appshare --quota 100
```

### 17.4 [Tricky] Your Function app's Storage Queue messages keep reappearing. Why?
**Answer**
The processing didn't complete/delete before the visibility timeout expired, or it threw an exception; the message becomes visible again and `DequeueCount` increments; after maxDequeueCount (default 5 in the Functions host) it moves to `<queue>-poison`.
**Explanation**
Fix: make processing idempotent, extend timeouts (use `UpdateMessageAsync` to extend visibility), or move long work to Durable Functions/Service Bus with lock renewal. Check `host.json` `queues.maxDequeueCount`, `visibilityTimeout`, `batchSize`.
**Code / CLI / Bicep**
```json
{ "version": "2.0", "extensions": { "queues": { "maxDequeueCount": 5, "visibilityTimeout": "00:05:00", "batchSize": 8 } } }
```
**Gotchas / follow-ups**
- Message size limit 64 KB (Base64 encoding overhead; configure message encoding in SDK).
- No ordering guarantees.

**Interview traps**
- Storage queue lacks DLQ/topics/sessions.
- Table isn't relational; queries without keys scan.
- Files != Blob; no blob events for Files.
- Cosmos Table API is not the same service as Table Storage.

**Cheat sheet**
- Queue: simple/cheap; Service Bus: rich.
- Table: PK+RK; Cosmos for scale/indexes.
- Files: SMB/NFS; Blob: objects; ADLS: HNS.
- Poison after 5 dequeues.

---

## Topic 18: Tooling (AzCopy, Storage Explorer, Azurite), monitoring and production scenarios

**Study notes**

- **AzCopy v10**: high-performance CLI for copy/sync to/from/between accounts; server-side copy between accounts; auth via `azcopy login` (Entra), `--identity` (managed identity), or SAS. Commands: `copy`, `sync`, `list`, `remove`, `jobs`, `login`.
- **Storage Explorer**: GUI for browsing, SAS generation, ACLs, uploads.
- **Azurite**: local emulator (blob 10000, queue 10001, table 10002); connection `UseDevelopmentStorage=true`; install via npm/Docker/VS Code. Not 100% parity (verify supported API version and features).
- **Monitoring**: Azure Monitor metrics (Transactions, Availability, SuccessE2ELatency, SuccessServerLatency, Ingress/Egress, BlobCapacity), diagnostic settings per service (`blobServices/default`) -> Log Analytics (`StorageBlobLogs`), Storage account/Event Hub/Archive. Logs are not enabled by default.

### 18.1 [Easy] How do you bulk copy data into Blob quickly?
**Answer**
Use AzCopy: `azcopy copy` for one-off and `azcopy sync` for incremental; it parallelises, resumes failed jobs and can copy server-side between accounts.
**Explanation**
Authenticate with `azcopy login` (needs Storage Blob Data Contributor) or a SAS. For HNS accounts use the same blob endpoint or dfs. Use `--dry-run` to preview and `--delete-destination` carefully with `sync`.
**Code / CLI / Bicep**
```bash
azcopy login
azcopy copy "C:\data\*" "https://acct.blob.core.windows.net/docs" --recursive
azcopy sync "C:\data" "https://acct.blob.core.windows.net/docs" --recursive --dry-run
azcopy copy "https://src.blob.core.windows.net/c?<SAS>" "https://dst.blob.core.windows.net/c" --recursive
```

### 18.2 [Follow-up] How do you develop and test locally without a real account?
**Answer**
Run Azurite and use `UseDevelopmentStorage=true`; for integration tests start it in Docker or via Testcontainers and create a container per test.
**Explanation**
Azurite supports Blob, Queue, Table (some features missing, e.g., not all newer REST versions/some Event Grid, lifecycle not supported). Functions local settings: `AzureWebJobsStorage=UseDevelopmentStorage=true`. Use `--skipApiVersionCheck` if SDK is newer than Azurite.
**Code / CLI / Bicep**
```bash
docker run -p 10000:10000 -p 10001:10001 -p 10002:10002 mcr.microsoft.com/azure-storage/azurite
```
```csharp
var svc = new BlobServiceClient("UseDevelopmentStorage=true");
var c = svc.GetBlobContainerClient("test-" + Guid.NewGuid().ToString("N"));
await c.CreateAsync();
```

### 18.3 [Medium] Enable diagnostics and write KQL for 403, 404 and latency.
**Answer**
Create a diagnostic setting for the blob service sending StorageRead/Write/Delete logs to Log Analytics, then query `StorageBlobLogs` by `StatusCode`, `StatusText`, and latency columns.
**Explanation**
Check `AuthenticationType` (OAuth, SAS, AccountKey, Anonymous) to detect leaked/unexpected callers. Useful alerts: 403 spikes, 5xx/ServerBusy, availability < 99%.
**Code / CLI / Bicep**
```bash
az monitor diagnostic-settings create --name blobdiag \
  --resource "/subscriptions/<sub>/resourceGroups/rg/providers/Microsoft.Storage/storageAccounts/acct/blobServices/default" \
  --workspace <logAnalyticsWorkspaceId> \
  --logs '[{"category":"StorageRead","enabled":true},{"category":"StorageWrite","enabled":true},{"category":"StorageDelete","enabled":true}]'
```
```kusto
StorageBlobLogs
| where TimeGenerated > ago(24h) and StatusCode in (403, 404, 409, 412, 503)
| summarize count() by StatusCode, StatusText, OperationName, AuthenticationType, bin(TimeGenerated, 1h)
```

### 18.4 [Hard] Production scenario drill: SAS leaked, accidental deletes, 403 after deploy
**Answer**
SAS leak: identify via logs (AuthenticationType == SAS, CallerIpAddress), revoke (stored policy / revoke delegation keys / rotate key), reduce exposure, and disable shared key. Accidental delete: undelete/restore versions/PITR. 403 after deploy: distinguish AuthorizationFailure (network/DNS/identity switch) from PermissionMismatch (RBAC) via logs.
**Explanation**
Runbook mindset: contain -> scope impact via logs (who/what/when) -> recover -> prevent. Preventive controls: user delegation SAS, short TTL, Defender alerts, soft delete + versioning + resource locks (`CanNotDelete` lock on the account), RBAC least privilege, deployment pipelines that assign roles before app start.
**Code / CLI / Bicep**
```kusto
StorageBlobLogs
| where TimeGenerated > ago(7d) and AuthenticationType == "SAS" and OperationName == "GetBlob"
| summarize downloads=count(), bytes=sum(ResponseBodySize) by CallerIpAddress, Uri
| top 20 by downloads desc
```
```bash
az lock create --name nodelete --lock-type CanNotDelete --resource-group rg --resource-name acct \
  --resource-type Microsoft.Storage/storageAccounts
```
**Gotchas / follow-ups**
- Resource locks don't prevent data-plane blob deletion; soft delete/immutability do.
- Rotating the key also breaks apps using it - rotate secondary first, update consumers, rotate primary.

### 18.5 [Tricky] Slow uploads and 412s in production - quick triage?
**Answer**
Slow uploads: compare E2E vs server latency, region distance, client concurrency/block size, firewall/proxy inspection, throttling (503) and Functions SNAT limits. 412s: find the conditional header (IfMatch/lease/tag condition) and whether concurrent writers or SDK retries caused it.
**Explanation**
Collect the `x-ms-client-request-id` / `x-ms-request-id` from exceptions (`RequestFailedException.ErrorCode`, `GetRawResponse().ClientRequestId`) and correlate to `StorageBlobLogs` `CorrelationId`/`RequestId`. Enable SDK logging via `AzureEventSourceListener` for headers/timings.
**Code / CLI / Bicep**
```csharp
using var listener = AzureEventSourceListener.CreateConsoleLogger(EventLevel.Informational);
try { await blob.UploadAsync(stream, opts); }
catch (RequestFailedException ex)
{
    Console.WriteLine($"{ex.Status} {ex.ErrorCode} reqId={ex.GetRawResponse()?.Headers.RequestId}");
}
```
**Gotchas / follow-ups**
- Retry after a lost response can produce 412/409 even though the first write succeeded.
- Verify SDK/logging API names against the current package version.

**Interview traps**
- Diagnostic logs are off by default - no history if you didn't enable them.
- Azurite != full parity.
- AzCopy `sync --delete-destination` can delete data.
- Resource lock doesn't protect blobs.
- Rotating keys without staging breaks consumers.

**Cheat sheet**
- AzCopy copy/sync; login via Entra/identity.
- Azurite for local; `UseDevelopmentStorage=true`.
- Diagnostics -> StorageBlobLogs; filter StatusCode/AuthenticationType.
- Incident flow: contain -> scope -> recover -> prevent.
- Capture request IDs for support.



