# Project 09: Distributed File & Object Storage Service

Build an S3-compatible distributed file storage service supporting chunked multipart uploads, SHA-256 content deduplication, presigned URL generation, and automatic storage tiering to S3 Glacier Deep Archive.

---

## 🗺️ System Architecture

```mermaid
flowchart TD
    Client["Client Browser"] -->|1. Request Presigned Upload URL| API["Storage Metadata API"]
    API -->|2. Generate HMAC Presigned URL| Client
    Client -->|3. Direct Binary Upload (Zero Backend JVM RAM!)| S3[("Amazon S3 Bucket")]
    
    S3 -->|4. S3 ObjectCreated Event| SQS["SQS Queue"]
    SQS --> Deduper["Deduplication & Metadata Worker"]
    Deduper --> Postgres[("PostgreSQL Metadata DB (SHA-256 Index)")]
```

---

## ⚡ Implementation: Direct S3 Presigned URL Generator

```java
package com.backend.engineering.projects.filestorage;

import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.PresignedPutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest;

import java.time.Duration;
import java.util.UUID;

@Service
public class S3PresignedUploadService {

    private final S3Presigner s3Presigner;
    private final String bucketName = "production-user-files";

    public S3PresignedUploadService(S3Presigner s3Presigner) {
        this.s3Presigner = s3Presigner;
    }

    public PresignedUploadDto generateUploadUrl(String contentType, long fileSize) {
        String objectKey = "uploads/" + UUID.randomUUID() + ".bin";

        PutObjectRequest objectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(objectKey)
                .contentType(contentType)
                .contentLength(fileSize)
                .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(15)) // URL expires in 15 minutes
                .putObjectRequest(objectRequest)
                .build();

        PresignedPutObjectRequest presignedPut = s3Presigner.presignPutObject(presignRequest);

        return new PresignedUploadDto(objectKey, presignedPut.url().toString(), 900L);
    }
}
```
