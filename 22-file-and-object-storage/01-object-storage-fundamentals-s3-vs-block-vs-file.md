# Object Storage Fundamentals: S3 vs Block vs File and Why DB BLOBs Are an Anti-Pattern

---

## 1. What Is It?
In cloud computing and backend engineering, **Object Storage** (e.g. AWS S3, Google Cloud Storage, MinIO) is a flat, non-hierarchical distributed storage architecture that manages unstructured data as discrete **Objects** addressed by unique **Keys** inside **Buckets**, accessible over standard HTTP REST APIs (`GET`, `PUT`, `DELETE`).

---

## 2. Why Storing Binary Files in Relational Databases Is an Anti-Pattern

```mermaid
flowchart LR
    subgraph AntiPattern["ANTI-PATTERN: Storing 10MB Video in PostgreSQL (BYTEA/BLOB)"]
        App1["App Thread"] -->|10MB BYTEA Write| DB[("PostgreSQL Buffer Pool")]
        DB -->|Pollutes 8KB Pages| Disks["Massive WAL & Disk Bloat"]
        Note over DB: Evicts Hot B+Tree Index Pages from RAM!
    end

    subgraph BestPractice["PRODUCTION BEST PRACTICE: Metadata in DB + Binary in S3"]
        App2["App Thread"] -->|1. Metadata: 120 bytes JSON| DB2[("PostgreSQL (Fast!)")]
        App2 -->|2. Raw Binary Stream| S3[("AWS S3 Object Storage")]
        Note over S3: High durability (11 9s); 0 DB Buffer Pool Pollution!
    end
```

### The 5 Catastrophic Flaws of Database BLOB Storage:
1. **Buffer Pool RAM Pollution**: Loading a $20\text{MB}$ PDF into a PostgreSQL table loads $2,500$ 8KB pages into the database **RAM Buffer Pool**, forcefully evicting hot B+Tree indexes and transactional tables from cache.
2. **Write-Ahead Log (WAL) Bloat**: Every single file insert writes raw binary bytes into the PostgreSQL WAL and replication stream, destroying replication throughput and saturating network bandwidth.
3. **Database Backup & Recovery Nightmares**: A 500GB database full of images takes 10 hours to `pg_dump` and restore during a disaster recovery scenario, violating RTO (Recovery Time Objective) SLAs.
4. **Expensive Storage**: AWS RDS database SSD storage costs $\approx \$0.115\text{/GB/month}$, whereas AWS S3 Standard costs $\approx \$0.023\text{/GB/month}$ ($5\times$ cheaper!).
5. **No Built-in CDN / HTTP Streaming**: Relational databases cannot natively stream chunked video bytes or integrate directly with CloudFront CDNs.

---

## 3. Storage Paradigms Comparison: Block vs File vs Object

| Feature | Block Storage (AWS EBS / SAN) | File Storage (AWS EFS / NFS) | Object Storage (AWS S3 / GCS) |
|---|---|---|---|
| **Storage Structure** | Raw sectors / disk blocks | Hierarchical directory tree | **Flat namespace (`Bucket/Key`)** |
| **Access Protocol** | Fibre Channel / iSCSI / NVMe | POSIX (NFS / SMB / CIFS) | **HTTP/HTTPS REST APIs (`GET`/`PUT`)** |
| **Max Capacity** | Up to $64\text{TB}$ per volume | Multi-terabyte shared volume | **Virtually Infinite (Exabytes)** |
| **Durability SLA** | $99.999\%$ ($5\text{ Nines}$) | $99.999999999\%$ ($11\text{ Nines}$) | **$99.999999999\%$ ($11\text{ Nines}$)** |
| **Cost per GB** | High ($\approx \$0.08 - \$0.15\text{/GB}$) | High ($\approx \$0.30\text{/GB}$) | **Ultra-Low ($\approx \$0.023\text{/GB}$)** |
| **Best Use Case** | OS boot drives, Database WAL storage | Legacy shared app file systems | **Images, videos, logs, backups, data lakes** |

---

## 4. How Does It Work?

### S3 Object Namespace & Key Naming
Unlike POSIX file systems with nested folder inodes:
- S3 has **no physical directories**.
- The string `users/avatars/2026/avatar_42.png` is simply an **atomic string Key** stored in a flat bucket dictionary.
- The forward slash (`/`) is treated as a delimiter purely for user interface visualization.

---

## 5. Implementation: Uploading Files to S3 via AWS SDK v2 in Java 21

```java
package com.backend.engineering.storage.s3;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;

import java.io.InputStream;
import java.util.Map;

@Service
public class S3ObjectStorageService {

    private static final Logger log = LoggerFactory.getLogger(S3ObjectStorageService.class);
    private final S3Client s3Client;
    private static final String BUCKET_NAME = "company-production-assets";

    public S3ObjectStorageService(S3Client s3Client) {
        this.s3Client = s3Client;
    }

    public void uploadFile(String objectKey, InputStream dataStream, long contentLength, String contentType) {
        PutObjectRequest putRequest = PutObjectRequest.builder()
                .bucket(BUCKET_NAME)
                .key(objectKey)
                .contentType(contentType)
                .metadata(Map.of("uploader-service", "asset-api", "retention", "permanent"))
                .build();

        // Stream raw bytes directly into S3 with ZERO intermediate disk buffering
        s3Client.putObject(putRequest, RequestBody.fromInputStream(dataStream, contentLength));
        log.info("Object [{}] successfully uploaded to bucket [{}]", objectKey, BUCKET_NAME);
    }
}
```

---

## 6. Performance

| Storage Architecture | Upload Latency ($10\text{MB}$ File) | DB Buffer Pool Impact | Storage Cost ($10\text{TB}$) |
|---|---|---|---|
| PostgreSQL `BYTEA` | $120\text{ms}$ | **Severe ($\approx 2,500$ pages evicted)** | $\$1,150\text{ / month}$ |
| **AWS S3 Object Storage** | **$35\text{ms}$** | **$0\text{ pages (Zero DB impact)}$** | **$\$230\text{ / month}$ ($80\%$ savings)** |

---

## 7. Interview Questions

### Q1: Why is storing large binary objects (images, videos, PDFs) inside a relational database considered a serious architectural flaw?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Buffer Pool Eviction & Cache Degradation**: Relational databases rely heavily on RAM buffer pools to cache frequently queried index B+Trees and table pages. Reading or writing large BLOB columns loads megabytes of contiguous binary pages into RAM, aggressively evicting critical relational data and destroying query cache hit ratios.
2. **Replication & WAL Saturation**: Every BLOB mutation must be recorded in the Write-Ahead Log (WAL) and streamed across the network to all read replicas, creating replication lag and saturating database I/O bandwidth.
3. **Backup & Recovery Inefficiency**: Daily snapshots and point-in-time recovery operations become massive and slow, violating RTO/RPO disaster recovery objectives.
4. **Cost & Scalability**: Database block storage is $5\times$ more expensive than S3, and relational databases cannot scale linearly to petabytes of unstructured media.
**Best Practice**: Store lightweight metadata (File ID, S3 Key, Size, MIME type, User ID) in the database, and stream the raw binary payload directly to an Object Store (AWS S3).
</details>

---

## 8. Quick Revision
- **Never Store BLOBs in DB**: Causes buffer pool thrashing, WAL bloat, and backup failures.
- **Object Storage (S3)**: Flat key-value namespace; virtually infinite scale; $11\text{ Nines}$ durability.
- **Block Storage (EBS)**: Raw disk volumes for OS and high-IOPS database engines.
- **Cost**: S3 is $> 5\times$ cheaper than database SSD storage.
