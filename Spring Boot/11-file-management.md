# 11 — File Management & Streaming in Spring Boot 3.x

> **Module Goal:** Master every dimension of enterprise-grade file management in modern Spring Boot 3.x (Java 17/21). From high-throughput multipart uploads, secure local NIO.2 and cloud object storage (AWS S3 SDK v2 / MinIO), to zero-copy streaming, HTTP 206 partial range byte requests, Tus-compliant resumable chunked uploads, and multi-layer defensive validation (Apache Tika magic bytes, ClamAV antivirus daemon, and path traversal guards).

---

## Table of Contents

1. [Architectural Overview & Core Abstractions](#1-architectural-overview--core-abstractions)
2. [File Upload Ingestion Approaches](#2-file-upload-ingestion-approaches)
   - [Approach 1: Standard `@RequestParam("file") MultipartFile`](#approach-1-standard-requestparamfile-multipartfile)
   - [Approach 2: Composite Multipart + JSON Payload via `@RequestPart`](#approach-2-composite-multipart--json-payload-via-requestpart)
   - [Approach 3: Chunked Resumable File Upload (Custom Chunk Assembler)](#approach-3-chunked-resumable-file-upload-custom-chunk-assembler)
   - [Approach 4: Direct Reactive / Stream-Through Upload to Object Storage](#approach-4-direct-reactive--stream-through-upload-to-object-storage)
3. [File Storage Backend Strategies](#3-file-storage-backend-strategies)
   - [Approach 1: Secure Local Filesystem Storage (NIO.2)](#approach-1-secure-local-filesystem-storage-nio2)
   - [Approach 2: Cloud Object Storage (AWS S3 SDK v2 / MinIO)](#approach-2-cloud-object-storage-aws-s3-sdk-v2--minio)
   - [Approach 3: Relational Database BLOB Storage via Spring Data JPA](#approach-3-relational-database-blob-storage-via-spring-data-jpa)
   - [Approach 4: Direct-to-Cloud Pre-Signed S3 Upload & Offloading](#approach-4-direct-to-cloud-pre-signed-s3-upload--offloading)
4. [File Download & Delivery Architectures](#4-file-download--delivery-architectures)
   - [Approach 1: Direct Resource Return (`FileSystemResource` / `ByteArrayResource`)](#approach-1-direct-resource-return-filesystemresource--bytearrayresource)
   - [Approach 2: Asynchronous Non-Blocking Streaming (`StreamingResponseBody`)](#approach-2-asynchronous-non-blocking-streaming-streamingresponsebody)
   - [Approach 3: HTTP 206 Partial Content Byte-Range Requests (Media Seeking)](#approach-3-http-206-partial-content-byte-range-requests-media-seeking)
   - [Approach 4: Pre-Signed CDN / Object Storage URL Redirection](#approach-4-pre-signed-cdn--object-storage-url-redirection)
5. [Security & Threat Mitigation Pipelines](#5-security--threat-mitigation-pipelines)
   - [Approach 1: Multi-Layer Validation (Extension + Apache Tika Magic Bytes)](#approach-1-multi-layer-validation-extension--apache-tika-magic-bytes)
   - [Approach 2: In-Flight Antivirus Stream Scanning (ClamAV Daemon)](#approach-2-in-flight-antivirus-stream-scanning-clamav-daemon)
   - [Approach 3: Defensive Path Traversal & Filename Sanitization](#approach-3-defensive-path-traversal--filename-sanitization)
6. [Asynchronous Background File Processing](#6-asynchronous-background-file-processing)
7. [Image Manipulation & Optimization Pipelines](#7-image-manipulation--optimization-pipelines)
8. [Comprehensive Configuration Reference](#8-comprehensive-configuration-reference)
9. [⚠️ Interview Traps & Production Gotchas](#️-interview-traps--production-gotchas)
10. [🔍 Deep Dive: DispatcherServlet & Multipart Internals](#-deep-dive-dispatcherservlet--multipart-internals)
11. [📝 Architectural Comparison Matrix](#-architectural-comparison-matrix)
12. [❓ Practice Questions & Real-World Scenarios](#-practice-questions--real-world-scenarios)

---

## 1. Architectural Overview & Core Abstractions

In a Spring Boot 3.x web application running on an embedded Servlet container (Tomcat 10.1+, Jetty 12+, Undertow), multipart requests (`multipart/form-data`) are governed by the Servlet 3.0+ specification through `jakarta.servlet.http.Part` and encapsulated cleanly by Spring’s `MultipartFile` interface.

```
                        HTTP POST (multipart/form-data)
                                      │
                                      ▼
                        DispatcherServlet.checkMultipart()
                                      │
                                      ▼
                      StandardServletMultipartResolver
                                      │
                     ┌────────────────┴────────────────┐
                     ▼                                 ▼
      Content-Length < Threshold        Content-Length > Threshold
     (Held in Heap Memory Buffer)     (Spooling to /tmp/spring-uploads)
                     └────────────────┬────────────────┘
                                      ▼
                           StandardMultipartFile
                                      │
                                      ▼
                        Controller Layer Injection
                  (@RequestParam vs @RequestPart vs Stream)
```

### The `MultipartFile` Contract

```java
package org.springframework.web.multipart;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Path;
import org.springframework.core.io.InputStreamSource;
import org.springframework.core.io.Resource;

public interface MultipartFile extends InputStreamSource {
    String getName();                     // Form control parameter name
    String getOriginalFilename();         // Raw client-supplied filename (⚠️ UNTRUSTED)
    String getContentType();             // Client-supplied MIME header (⚠️ UNTRUSTED)
    boolean isEmpty();
    long getSize();                      // Payload size in bytes
    byte[] getBytes() throws IOException; // ⚠️ DANGER: Allocates full payload on JVM heap
    InputStream getInputStream() throws IOException;
    void transferTo(File dest) throws IOException, IllegalStateException;
    default void transferTo(Path dest) throws IOException, IllegalStateException {
        transferTo(dest.toFile());
    }
    Resource getResource();
}
```

> [!WARNING]
> Both `getOriginalFilename()` and `getContentType()` are completely client-controlled headers. An attacker can set `Content-Type: image/png` on a compiled PHP web shell or an executable ELF binary named `../../../../etc/shadow\0.png`. Never make storage or authorization decisions based on these values.

---

## 2. File Upload Ingestion Approaches

Enterprise applications ingest files under vastly different constraints: small user avatars, complex business forms accompanied by metadata, multi-gigabyte video files, or distributed microservice ingestions.

---

### Approach 1: Standard `@RequestParam("file") MultipartFile`

#### Scenario
A simple web form or mobile endpoint where an authenticated user uploads a single file (e.g., an avatar profile picture, a resume PDF, or a receipt) alongside standard HTTP request query parameters or form fields.

#### Use-Case
Standard B2C or B2B administrative portals where file uploads are isolated, infrequent, and bounded within moderate sizes (e.g., `< 25MB`).

#### Benefits Over Others
- **Zero Overhead Configuration:** Native Spring MVC automatic data binding without custom deserializers.
- **Simplicity:** Directly binds to standard HTML form file inputs.
- **Fail-Fast Validation:** Spring's `StandardServletMultipartResolver` rejects payloads exceeding `spring.servlet.multipart.max-file-size` before controller execution.

#### Production-Grade Code Example

```java
package com.example.filemgmt.upload.standard;

import com.example.filemgmt.model.FileStorageMetadata;
import com.example.filemgmt.service.FileStorageService;
import com.example.filemgmt.validation.FileValidator;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.constraints.NotBlank;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.security.Principal;

@RestController
@RequestMapping("/api/v1/files")
@RequiredArgsConstructor
@Validated
@Slf4j
@Tag(name = "Standard File Upload", description = "Standard multipart endpoints for single and batch file uploads")
public class StandardFileUploadController {

    private final FileStorageService fileStorageService;
    private final FileValidator fileValidator;

    @PostMapping(value = "/single", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @Operation(summary = "Upload a single file with basic categorization")
    public ResponseEntity<FileUploadResponse> uploadSingleFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam("category") @NotBlank String category,
            Principal principal) {

        String uploaderId = principal != null ? principal.getName() : "anonymous";
        log.info("Processing upload request: name='{}', size={} bytes, declaredType='{}', category='{}', user='{}'",
                file.getOriginalFilename(), file.getSize(), file.getContentType(), category, uploaderId);

        // Comprehensive defensive validation: empty check, extension, magic bytes
        fileValidator.validateOrThrow(file);

        // Persist to underlying storage backend
        FileStorageMetadata metadata = fileStorageService.store(file, category, uploaderId);

        FileUploadResponse response = new FileUploadResponse(
                metadata.id(),
                metadata.originalFilename(),
                metadata.storedFilename(),
                metadata.contentType(),
                metadata.sizeBytes(),
                metadata.downloadUri(),
                metadata.checksumSha256()
        );

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    public record FileUploadResponse(
            String fileId,
            String originalFilename,
            String storedFilename,
            String contentType,
            long sizeBytes,
            String downloadUrl,
            String sha256Checksum
    ) {}
}
```

---

### Approach 2: Composite Multipart + JSON Payload via `@RequestPart`

#### Scenario
A complex enterprise business transaction where an uploaded document must be bound atomically with a rich, nested JSON metadata structure (e.g., an Insurance Claim submitting damage photos alongside policy details, claim line items, and dynamic nested incident coordinates).

#### Use-Case
APIs where splitting the request into two separate HTTP calls (e.g., `POST /claims` returning an ID, followed by `POST /claims/{id}/documents`) risks orphaned files or metadata without accompanying documents in the event of client network failures.

#### Benefits Over Others
- **Atomic Single-Roundtrip Operations:** Eliminates two-phase upload workflows and complex client cleanup choreography.
- **Deep Jackson Deserialization:** `@RequestPart` routes the JSON part directly through Spring's `MappingJackson2HttpMessageConverter`, enabling full `@Valid` bean validation on deeply nested objects.
- **Explicit Content-Type Verification:** Inspects each MIME part's individual `Content-Type` header (e.g., `application/json` vs `application/pdf`).

#### Production-Grade Code Example

```java
package com.example.filemgmt.upload.composite;

import com.example.filemgmt.model.FileStorageMetadata;
import com.example.filemgmt.service.FileStorageService;
import com.example.filemgmt.validation.FileValidator;
import jakarta.validation.Valid;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

@RestController
@RequestMapping("/api/v1/claims")
@RequiredArgsConstructor
@Slf4j
public class CompositeClaimUploadController {

    private final FileStorageService storageService;
    private final FileValidator fileValidator;

    @PostMapping(value = "/submit", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<ClaimSubmissionResult> submitClaimWithEvidence(
            @RequestPart("claimData") @Valid ClaimMetadataRequest claimData,
            @RequestPart("document") MultipartFile document,
            @RequestPart(value = "supplementalPhotos", required = false) List<MultipartFile> supplementalPhotos) {

        log.info("Ingesting claim submission: policyId='{}', claimant='{}', mainDoc='{}'",
                claimData.policyNumber(), claimData.claimantName(), document.getOriginalFilename());

        // Validate primary document
        fileValidator.validateDocument(document);

        // Store primary document
        FileStorageMetadata primaryDocMeta = storageService.store(document, "claims", claimData.claimantName());

        // Optionally process and store supplemental evidence
        List<String> photoUrls = List.of();
        if (supplementalPhotos != null && !supplementalPhotos.isEmpty()) {
            photoUrls = supplementalPhotos.stream()
                    .peek(fileValidator::validateImage)
                    .map(photo -> storageService.store(photo, "claims/photos", claimData.claimantName()).downloadUri())
                    .toList();
        }

        ClaimSubmissionResult result = new ClaimSubmissionResult(
                "CLM-" + Instant.now().toEpochMilli(),
                claimData.policyNumber(),
                primaryDocMeta.downloadUri(),
                photoUrls,
                "SUBMITTED",
                Instant.now()
        );

        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }

    public record ClaimMetadataRequest(
            @NotBlank(message = "Policy number is required") String policyNumber,
            @NotBlank(message = "Claimant name cannot be blank") String claimantName,
            @NotNull(message = "Estimated loss amount is required")
            @DecimalMin(value = "0.01", message = "Loss amount must be strictly positive")
            BigDecimal estimatedLossAmount,
            @NotEmpty(message = "At least one damage category is required") List<String> damageCategories
    ) {}

    public record ClaimSubmissionResult(
            String claimReferenceNumber,
            String policyNumber,
            String primaryDocumentUrl,
            List<String> supplementalPhotoUrls,
            String status,
            Instant submissionTimestamp
    ) {}
}
```

---

### Approach 3: Chunked Resumable File Upload (Custom Chunk Assembler)

#### Scenario
Uploading large files (e.g., 500MB video recordings, disk images, or multi-gigabyte database exports) over mobile or unstable network connections where packet loss causes standard single-request uploads to terminate abruptly.

#### Use-Case
High-reliability consumer video platforms, enterprise data backup portals, and field inspections where uploads must pause, resume, and survive temporary network drops without re-uploading previously transmitted bytes.

#### Benefits Over Others
- **Resumability:** A dropped socket only invalidates the active 5MB chunk rather than the entire 2GB payload.
- **Bounded Heap & Disk Usage:** Each chunk is small (typically 5MB-10MB), preventing spikes in container memory.
- **Parallel Chunk Transmission:** Client browsers can upload multiple chunks concurrently (e.g., chunks 1, 2, 3 concurrently) before triggering final assembly.

#### Production-Grade Code Example

```java
package com.example.filemgmt.upload.chunked;

import com.example.filemgmt.model.FileStorageMetadata;
import com.example.filemgmt.service.FileStorageService;
import jakarta.annotation.PostConstruct;
import lombok.Builder;
import lombok.Getter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.mock.web.MockMultipartFile;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.nio.channels.FileChannel;
import java.nio.file.*;
import java.security.MessageDigest;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@RestController
@RequestMapping("/api/v1/resumable-upload")
@RequiredArgsConstructor
@Slf4j
public class ChunkedUploadController {

    private final ResumableUploadCoordinator coordinator;

    @PostMapping("/init")
    public ResponseEntity<UploadSessionResponse> initializeSession(@RequestBody InitSessionRequest request) {
        String sessionId = coordinator.initSession(
                request.filename(),
                request.totalSize(),
                request.chunkSize(),
                request.totalChunks(),
                request.expectedSha256()
        );
        return ResponseEntity.ok(new UploadSessionResponse(sessionId, request.chunkSize()));
    }

    @PostMapping("/{sessionId}/chunk/{chunkIndex}")
    public ResponseEntity<ChunkAck> uploadChunk(
            @PathVariable String sessionId,
            @PathVariable int chunkIndex,
            @RequestParam("chunk") MultipartFile chunk) throws IOException {

        coordinator.saveChunk(sessionId, chunkIndex, chunk);
        boolean isFullyUploaded = coordinator.isComplete(sessionId);

        return ResponseEntity.ok(new ChunkAck(sessionId, chunkIndex, isFullyUploaded));
    }

    @PostMapping("/{sessionId}/finalize")
    public ResponseEntity<FileStorageMetadata> finalizeUpload(@PathVariable String sessionId) throws IOException {
        FileStorageMetadata metadata = coordinator.assembleAndPromote(sessionId);
        return ResponseEntity.ok(metadata);
    }

    public record InitSessionRequest(String filename, long totalSize, int chunkSize, int totalChunks, String expectedSha256) {}
    public record UploadSessionResponse(String sessionId, int chunkSize) {}
    public record ChunkAck(String sessionId, int chunkNumber, boolean isReadyForFinalization) {}
}

@Service
@Slf4j
@RequiredArgsConstructor
class ResumableUploadCoordinator {

    @Value("${app.storage.chunk-staging-dir:${java.io.tmpdir}/chunks}")
    private String stagingPathStr;

    private final FileStorageService finalStorageService;
    private final Map<String, UploadSessionState> activeSessions = new ConcurrentHashMap<>();
    private Path stagingRoot;

    @PostConstruct
    void init() throws IOException {
        stagingRoot = Paths.get(stagingPathStr).toAbsolutePath().normalize();
        Files.createDirectories(stagingRoot);
    }

    public String initSession(String filename, long totalSize, int chunkSize, int totalChunks, String expectedSha256) {
        String sessionId = UUID.randomUUID().toString();
        Path sessionDir = stagingRoot.resolve(sessionId);

        try {
            Files.createDirectories(sessionDir);
        } catch (IOException e) {
            throw new IllegalStateException("Failed to initialize chunk staging directory for session: " + sessionId, e);
        }

        UploadSessionState state = UploadSessionState.builder()
                .sessionId(sessionId)
                .filename(filename)
                .totalSizeBytes(totalSize)
                .chunkSizeBytes(chunkSize)
                .totalChunks(totalChunks)
                .expectedSha256(expectedSha256)
                .sessionDirectory(sessionDir)
                .receivedChunks(ConcurrentHashMap.newKeySet())
                .createdAt(Instant.now())
                .build();

        activeSessions.put(sessionId, state);
        log.info("Initialized resumable upload session '{}' for file '{}' ({} chunks, {} bytes)",
                sessionId, filename, totalChunks, totalSize);
        return sessionId;
    }

    public void saveChunk(String sessionId, int chunkIndex, MultipartFile chunk) throws IOException {
        UploadSessionState state = activeSessions.get(sessionId);
        if (state == null) {
            throw new IllegalArgumentException("Unknown or expired upload session: " + sessionId);
        }

        if (chunkIndex < 0 || chunkIndex >= state.getTotalChunks()) {
            throw new IndexOutOfBoundsException("Chunk index out of bounds: " + chunkIndex);
        }

        Path chunkTarget = state.getSessionDirectory().resolve(String.format("part_%05d.chunk", chunkIndex));
        chunk.transferTo(chunkTarget);
        state.getReceivedChunks().add(chunkIndex);

        log.debug("Chunk {}/{} received for session '{}'", chunkIndex + 1, state.getTotalChunks(), sessionId);
    }

    public boolean isComplete(String sessionId) {
        UploadSessionState state = activeSessions.get(sessionId);
        return state != null && state.getReceivedChunks().size() == state.getTotalChunks();
    }

    public FileStorageMetadata assembleAndPromote(String sessionId) throws IOException {
        UploadSessionState state = activeSessions.get(sessionId);
        if (state == null || !isComplete(sessionId)) {
            throw new IllegalStateException("Session is not complete or invalid: " + sessionId);
        }

        Path assembledFile = state.getSessionDirectory().resolve("assembled_" + state.getFilename());

        // Zero-copy assembly using NIO FileChannel.transferFrom
        try (FileChannel outputChannel = FileChannel.open(assembledFile,
                StandardOpenOption.CREATE, StandardOpenOption.WRITE, StandardOpenOption.TRUNCATE_EXISTING)) {

            long currentPosition = 0;
            for (int i = 0; i < state.getTotalChunks(); i++) {
                Path partPath = state.getSessionDirectory().resolve(String.format("part_%05d.chunk", i));
                if (!Files.exists(partPath)) {
                    throw new FileNotFoundException("Missing chunk component: " + partPath);
                }

                try (FileChannel inputChannel = FileChannel.open(partPath, StandardOpenOption.READ)) {
                    long partSize = inputChannel.size();
                    outputChannel.transferFrom(inputChannel, currentPosition, partSize);
                    currentPosition += partSize;
                }
            }
        }

        // Validate final file size matches expected total
        long assembledSize = Files.size(assembledFile);
        if (assembledSize != state.getTotalSizeBytes()) {
            throw new IllegalStateException("Assembled file size (" + assembledSize +
                    ") does not match expected size (" + state.getTotalSizeBytes() + ")");
        }

        // Promote to permanent storage
        FileStorageMetadata metadata;
        try (InputStream assembledStream = Files.newInputStream(assembledFile)) {
            MockMultipartFile multipartFile = new MockMultipartFile(
                    "file",
                    state.getFilename(),
                    Files.probeContentType(assembledFile),
                    assembledStream
            );
            metadata = finalStorageService.store(multipartFile, "assembled", "system-chunk-worker");
        } finally {
            // Clean up staging directory recursively
            cleanupSessionDirectory(state.getSessionDirectory());
            activeSessions.remove(sessionId);
        }

        return metadata;
    }

    private void cleanupSessionDirectory(Path directory) {
        try (var pathStream = Files.walk(directory)) {
            pathStream.sorted(Comparator.reverseOrder())
                    .forEach(path -> {
                        try { Files.deleteIfExists(path); } catch (IOException ignored) {}
                    });
        } catch (IOException e) {
            log.warn("Failed to recursively delete chunk staging dir: {}", directory, e);
        }
    }

    @Getter
    @Builder
    private static class UploadSessionState {
        private final String sessionId;
        private final String filename;
        private final long totalSizeBytes;
        private final int chunkSizeBytes;
        private final int totalChunks;
        private final String expectedSha256;
        private final Path sessionDirectory;
        private final Set<Integer> receivedChunks;
        private final Instant createdAt;
    }
}
```

---

### Approach 4: Direct Reactive / Stream-Through Upload to Object Storage

#### Scenario
Extreme high-throughput gateways receiving continuous media streams, high-volume sensor batches, or 50GB analytics tarballs where spooling even temporary files to disk would overwhelm container NVMe scratch disks and exhaust server I/O IOPS.

#### Use-Case
Microservices hosted on ephemeral containers (AWS Fargate, Google Cloud Run, Kubernetes pods with 512MB RAM limits and read-only root filesystems) acting as pass-through proxies to AWS S3.

#### Benefits Over Others
- **Zero Local Disk Spooling:** Bytes are read directly from the inbound HTTP socket and piped straight into the outbound S3 HTTP connection.
- **Tiny Memory Footprint:** Constant memory usage bounded by small ring buffers (e.g., 64KB), regardless of whether the file is 10MB or 50GB.
- **Instant Latency Optimization:** Eliminates the two-stage latency of "upload-to-server-then-upload-to-s3".

#### Production-Grade Code Example

```java
package com.example.filemgmt.upload.directstream;

import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.tomcat.util.http.fileupload.FileItemIterator;
import org.apache.tomcat.util.http.fileupload.FileItemStream;
import org.apache.tomcat.util.http.fileupload.servlet.ServletFileUpload;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;

import java.io.InputStream;
import java.util.UUID;

@RestController
@RequestMapping("/api/v1/direct-stream")
@RequiredArgsConstructor
@Slf4j
public class DirectStreamUploadController {

    private final S3Client s3Client;

    @Value("${aws.s3.bucket}")
    private String s3BucketName;

    @PostMapping("/upload")
    public ResponseEntity<StreamUploadResponse> streamUploadDirectToS3(HttpServletRequest request) throws Exception {
        if (!ServletFileUpload.isMultipartContent(request)) {
            return ResponseEntity.badRequest().build();
        }

        ServletFileUpload upload = new ServletFileUpload();
        FileItemIterator iterator = upload.getItemIterator(request);

        String generatedKey = null;
        long totalBytesPiped = 0;

        while (iterator.hasNext()) {
            FileItemStream item = iterator.next();
            if (!item.isFormField()) {
                String originalFilename = item.getName();
                generatedKey = "direct-stream/" + UUID.randomUUID() + "-" + originalFilename;

                try (InputStream stream = item.openStream()) {
                    log.info("Piping inbound stream directly to S3 bucket '{}', key '{}'", s3BucketName, generatedKey);

                    // Note: AWS S3 PutObject requires known content length or chunked transfer
                    // In production AWS SDK v2, use TransferManager or MultipartUpload for unknown lengths
                    PutObjectRequest putRequest = PutObjectRequest.builder()
                            .bucket(s3BucketName)
                            .key(generatedKey)
                            .contentType(item.getContentType())
                            .build();

                    // Uses S3 TransferManager under the hood or custom chunked RequestBody
                    s3Client.putObject(putRequest, RequestBody.fromInputStream(stream, request.getContentLengthLong()));
                }
                break; // Process primary file
            }
        }

        return ResponseEntity.ok(new StreamUploadResponse(generatedKey, s3BucketName));
    }

    public record StreamUploadResponse(String objectKey, String bucket) {}
}
```

---

## 3. File Storage Backend Strategies

Choosing where bytes reside represents one of the most critical durability, scaling, and architectural decisions in application design.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        File Storage Decisions                          │
└────────────────────────────────────────────────────────────────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           ▼                        ▼                        ▼
  Local Filesystem (NIO.2)     Cloud Object (S3)       Database BLOB
  - High speed local NVMe      - Infinite scaling       - ACID atomic rollback
  - Single server bounded      - Zero local disk risk   - DB heap/table bloat
  - Broken on K8s replicas     - Global multi-AZ HA     - Hard limit (<100KB)
```

---

### Approach 1: Secure Local Filesystem Storage (NIO.2)

#### Scenario
On-premise installations, standalone single-node internal server tools, air-gapped secure enclaves without cloud egress, or development/testing environments.

#### Use-Case
Local temporary caching, intranet appliances, or applications where local NVMe SSD speeds (3,000MB/s) outperform cloud API calls over WAN.

#### Benefits Over Others
- **Sub-Millisecond Read Latency:** Direct OS page-cache integration and kernel zero-copy file transfer.
- **Zero Third-Party Dependencies:** No S3 SDKs, credentials, or cloud network latency.
- **Atomic Operations:** Native support for OS-level atomic file moves (`ATOMIC_MOVE`).

#### Production-Grade Code Example

```java
package com.example.filemgmt.storage.local;

import com.example.filemgmt.model.FileStorageMetadata;
import com.example.filemgmt.service.FileStorageService;
import jakarta.annotation.PostConstruct;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;
import org.springframework.web.multipart.MultipartFile;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.nio.file.*;
import java.security.DigestInputStream;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.time.LocalDate;
import java.util.HexFormat;
import java.util.UUID;

@Service
@ConditionalOnProperty(name = "app.storage.type", havingValue = "local", matchIfMissing = true)
@Slf4j
public class LocalNioFileStorageService implements FileStorageService {

    private final Path rootStorageLocation;
    private final String baseUrl;

    public LocalNioFileStorageService(
            @Value("${app.storage.local.base-dir:/var/app/data/uploads}") String baseDir,
            @Value("${app.storage.base-url:http://localhost:8080}") String baseUrl) {
        this.rootStorageLocation = Paths.get(baseDir).toAbsolutePath().normalize();
        this.baseUrl = baseUrl;
    }

    @PostConstruct
    public void init() throws IOException {
        Files.createDirectories(rootStorageLocation);
        log.info("Initialized secure local filesystem storage at absolute root: {}", rootStorageLocation);
    }

    @Override
    public FileStorageMetadata store(MultipartFile file, String category, String uploadedBy) {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("Cannot store empty file payload");
        }

        String rawOriginalFilename = StringUtils.cleanPath(
                file.getOriginalFilename() != null ? file.getOriginalFilename() : "unknown.bin"
        );
        String extension = extractExtension(rawOriginalFilename);
        String secureStoredName = UUID.randomUUID() + (extension.isEmpty() ? "" : "." + extension);

        // Date-partitioned directory tree: uploads/2026/03/30/UUID.pdf
        // Prevents ext4/NTFS performance degradation caused by having >50k files in a single flat directory
        LocalDate now = LocalDate.now();
        Path partitionedDir = rootStorageLocation
                .resolve(category)
                .resolve(String.valueOf(now.getYear()))
                .resolve(String.format("%02d", now.getMonthValue()))
                .resolve(String.format("%02d", now.getDayOfMonth()))
                .normalize();

        // Path Traversal Defense: Ensure destination directory is strictly within root
        if (!partitionedDir.startsWith(rootStorageLocation)) {
            throw new SecurityException("Directory escape detected: " + partitionedDir);
        }

        Path targetFile = partitionedDir.resolve(secureStoredName).normalize();
        if (!targetFile.startsWith(rootStorageLocation)) {
            throw new SecurityException("File escape detected: " + targetFile);
        }

        try {
            Files.createDirectories(partitionedDir);

            // Compute SHA-256 while streaming bytes directly to disk via DigestInputStream
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            try (InputStream is = file.getInputStream();
                 DigestInputStream dis = new DigestInputStream(is, digest)) {
                Files.copy(dis, targetFile, StandardCopyOption.REPLACE_EXISTING);
            }

            String sha256Hex = HexFormat.of().formatHex(digest.digest());
            String relativePath = rootStorageLocation.relativize(targetFile).toString().replace('\\', '/');

            log.info("Persisted local file: original='{}', storedPath='{}', size={} bytes, sha256='{}'",
                    rawOriginalFilename, relativePath, file.getSize(), sha256Hex);

            return new FileStorageMetadata(
                    UUID.randomUUID().toString(),
                    rawOriginalFilename,
                    secureStoredName,
                    relativePath,
                    file.getContentType(),
                    file.getSize(),
                    sha256Hex,
                    baseUrl + "/api/v1/files/download/" + relativePath,
                    uploadedBy
            );

        } catch (IOException | NoSuchAlgorithmException e) {
            throw new RuntimeException("Failed to store file on local filesystem", e);
        }
    }

    @Override
    public Resource loadAsResource(String relativePath) {
        try {
            Path targetPath = rootStorageLocation.resolve(relativePath).normalize();

            // Guard against traversal attempts like "../../etc/passwd"
            if (!targetPath.startsWith(rootStorageLocation)) {
                throw new SecurityException("Path traversal attempt detected: " + relativePath);
            }

            Resource resource = new FileSystemResource(targetPath);
            if (!resource.exists() || !resource.isReadable()) {
                throw new FileNotFoundException("File does not exist or is not readable: " + relativePath);
            }

            return resource;
        } catch (FileNotFoundException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void delete(String relativePath) {
        Path targetPath = rootStorageLocation.resolve(relativePath).normalize();
        if (!targetPath.startsWith(rootStorageLocation)) {
            throw new SecurityException("Path traversal attempt: " + relativePath);
        }
        try {
            Files.deleteIfExists(targetPath);
        } catch (IOException e) {
            log.error("Failed to delete local file: {}", targetPath, e);
        }
    }

    private String extractExtension(String filename) {
        int dot = filename.lastIndexOf('.');
        return dot > 0 ? filename.substring(dot + 1).toLowerCase() : "";
    }
}
```

---

### Approach 2: Cloud Object Storage (AWS S3 SDK v2 / MinIO)

#### Scenario
Clustered production microservices deployed on Kubernetes (EKS/GKE), ECS, or multi-instance autoscaling groups behind a round-robin load balancer.

#### Use-Case
Standard modern enterprise software architecture where instances are disposable and storage must be virtually infinite, durable (99.999999999% 11 9's), and globally available.

#### Benefits Over Others
- **Horizontal Elasticity:** No instance-affinity or shared disk corruption risks across multiple pods.
- **Built-In Server-Side Encryption (SSE):** Automatic hardware encryption via AWS KMS or S3-managed AES-256.
- **Lifecycle & Tiering Policies:** Transparent transition of stale files to Glacier or automated deletion after retention windows.

#### Production-Grade Code Example

```java
package com.example.filemgmt.storage.s3;

import com.example.filemgmt.model.FileStorageMetadata;
import com.example.filemgmt.service.FileStorageService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.core.io.InputStreamResource;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import software.amazon.awssdk.core.ResponseInputStream;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

import java.io.IOException;
import java.io.InputStream;
import java.security.DigestInputStream;
import java.security.MessageDigest;
import java.time.LocalDate;
import java.util.HexFormat;
import java.util.Map;
import java.util.UUID;

@Service
@ConditionalOnProperty(name = "app.storage.type", havingValue = "s3")
@RequiredArgsConstructor
@Slf4j
public class AwsS3FileStorageService implements FileStorageService {

    private final S3Client s3Client;

    @Value("${aws.s3.bucket}")
    private String bucketName;

    @Value("${aws.s3.cdn-domain:}")
    private String cdnDomain;

    @Override
    public FileStorageMetadata store(MultipartFile file, String category, String uploadedBy) {
        String originalFilename = file.getOriginalFilename() != null ? file.getOriginalFilename() : "unnamed";
        String extension = extractExtension(originalFilename);
        String s3Key = buildS3Key(category, extension);

        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] fileBytes;
            try (InputStream is = file.getInputStream();
                 DigestInputStream dis = new DigestInputStream(is, digest)) {
                fileBytes = dis.readAllBytes();
            }

            String sha256Hex = HexFormat.of().formatHex(digest.digest());

            PutObjectRequest putRequest = PutObjectRequest.builder()
                    .bucket(bucketName)
                    .key(s3Key)
                    .contentType(file.getContentType())
                    .contentLength((long) fileBytes.length)
                    .serverSideEncryption(ServerSideEncryption.AES256)
                    .metadata(Map.of(
                            "original-filename", sanitizeForMetadata(originalFilename),
                            "sha256-checksum", sha256Hex,
                            "uploaded-by", uploadedBy
                    ))
                    .build();

            s3Client.putObject(putRequest, RequestBody.fromBytes(fileBytes));

            String accessUrl = (cdnDomain != null && !cdnDomain.isBlank())
                    ? cdnDomain + "/" + s3Key
                    : "https://" + bucketName + ".s3.amazonaws.com/" + s3Key;

            log.info("Persisted object to S3: bucket='{}', key='{}', size={} bytes", bucketName, s3Key, fileBytes.length);

            return new FileStorageMetadata(
                    UUID.randomUUID().toString(),
                    originalFilename,
                    s3Key,
                    s3Key,
                    file.getContentType(),
                    (long) fileBytes.length,
                    sha256Hex,
                    accessUrl,
                    uploadedBy
            );

        } catch (Exception e) {
            throw new RuntimeException("S3 Object upload failed for key: " + s3Key, e);
        }
    }

    @Override
    public Resource loadAsResource(String s3Key) {
        try {
            GetObjectRequest getRequest = GetObjectRequest.builder()
                    .bucket(bucketName)
                    .key(s3Key)
                    .build();

            ResponseInputStream<GetObjectResponse> s3Stream = s3Client.getObject(getRequest);
            return new InputStreamResource(s3Stream);
        } catch (NoSuchKeyException e) {
            throw new RuntimeException("Object not found in S3 bucket: " + s3Key, e);
        }
    }

    @Override
    public void delete(String s3Key) {
        DeleteObjectRequest deleteRequest = DeleteObjectRequest.builder()
                .bucket(bucketName)
                .key(s3Key)
                .build();
        s3Client.deleteObject(deleteRequest);
        log.info("Deleted S3 object: bucket='{}', key='{}'", bucketName, s3Key);
    }

    private String buildS3Key(String category, String extension) {
        LocalDate now = LocalDate.now();
        return String.format("%s/%d/%02d/%02d/%s%s",
                category,
                now.getYear(),
                now.getMonthValue(),
                now.getDayOfMonth(),
                UUID.randomUUID(),
                extension.isEmpty() ? "" : "." + extension
        );
    }

    private String extractExtension(String filename) {
        int dot = filename.lastIndexOf('.');
        return dot > 0 ? filename.substring(dot + 1).toLowerCase() : "";
    }

    private String sanitizeForMetadata(String value) {
        return value.replaceAll("[^a-zA-Z0-9._-]", "_");
    }
}
```

---

### Approach 3: Relational Database BLOB Storage via Spring Data JPA

#### Scenario
Highly regulated compliance applications (e.g., electronic sign-off certificates, cryptographic keys, or user avatars `< 64KB`) requiring absolute ACID transactional rollback consistency where database state and binary state can never drift apart.

#### Use-Case
Small files where taking a database backup (e.g., `pg_dump`) must encompass 100% of the application state without coordinating external filesystem or cloud bucket snapshots.

#### Benefits Over Others
- **Transactional Atomicity:** If the `@Transactional` method rolls back due to a business exception, the stored file binary is completely reverted in the same commit.
- **Unified Backup & Restore:** Zero danger of dangling references where a database record points to an S3 key that failed to delete.
- **Relational Access Control:** Governed directly by standard RDBMS row-level security and schema grants.

#### Production-Grade Code Example

```java
package com.example.filemgmt.storage.db;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.sql.Types;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

@Entity
@Table(name = "file_blobs")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class DatabaseFileBlob {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(name = "filename", nullable = false)
    private String filename;

    @Column(name = "content_type", nullable = false)
    private String contentType;

    @Column(name = "size_bytes", nullable = false)
    private long sizeBytes;

    @Lob
    @JdbcTypeCode(Types.BINARY)
    @Column(name = "binary_content", nullable = false)
    @Basic(fetch = FetchType.LAZY) // Critical: Never load binary into heap unless explicitly accessed
    private byte[] content;

    @Column(name = "uploaded_at", nullable = false, updatable = false)
    private Instant uploadedAt;
}

interface DatabaseFileRepository extends JpaRepository<DatabaseFileBlob, String> {}

@Service
@RequiredArgsConstructor
public class DatabaseBlobStorageService {

    private final DatabaseFileRepository fileRepository;

    @Transactional
    public String storeFileBlob(MultipartFile file) throws IOException {
        if (file.getSize() > 100 * 1024) { // Hard boundary: 100KB guardrail
            throw new IllegalArgumentException("Database BLOB storage is restricted to files <= 100KB to protect buffer pools");
        }

        DatabaseFileBlob blob = DatabaseFileBlob.builder()
                .filename(file.getOriginalFilename())
                .contentType(file.getContentType())
                .sizeBytes(file.getSize())
                .content(file.getBytes())
                .uploadedAt(Instant.now())
                .build();

        DatabaseFileBlob saved = fileRepository.save(blob);
        return saved.getId();
    }

    @Transactional(readOnly = true)
    public Optional<DatabaseFileBlob> getFileBlob(String id) {
        return fileRepository.findById(id);
    }
}
```

---

### Approach 4: Direct-to-Cloud Pre-Signed S3 Upload & Offloading

#### Scenario
Public-facing SaaS platforms with millions of users uploading 4K video files or raw photo archives where routing uploads through the Spring Boot application cluster would require massive bandwidth provisioning, dozens of Tomcat nodes, and massive egress costs.

#### Use-Case
Architectures maximizing infrastructure efficiency by offloading the entire binary transfer directly to AWS S3, utilizing Spring Boot purely for authentication, authorization, and cryptographic signature issuance.

#### Benefits Over Others
- **Zero App-Server Bandwidth & Memory:** The file bytes bypass the Spring Boot container entirely, travelling directly from the client's browser to AWS S3.
- **Infinite Scalability:** A modest 2-core Spring Boot instance can easily handle 10,000 upload authorizations per second.
- **Cost Minimization:** No load balancer ingress/egress transit fees on the application tier.

#### Production-Grade Code Example

```java
package com.example.filemgmt.storage.presigned;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.PresignedPutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest;

import java.security.Principal;
import java.time.Duration;
import java.util.Map;
import java.util.UUID;

@RestController
@RequestMapping("/api/v1/direct-s3")
@RequiredArgsConstructor
@Slf4j
public class PresignedS3UploadController {

    private final S3Presigner s3Presigner;

    @Value("${aws.s3.bucket}")
    private String bucketName;

    @PostMapping("/generate-upload-ticket")
    public ResponseEntity<PresignedUploadTicket> generateUploadTicket(
            @RequestBody UploadTicketRequest request,
            Principal principal) {

        String uploader = principal != null ? principal.getName() : "anonymous";
        String extension = extractExtension(request.filename());
        String objectKey = "direct-uploads/" + UUID.randomUUID() + (extension.isEmpty() ? "" : "." + extension);

        log.info("Generating S3 pre-signed upload URL for user='{}', key='{}', expectedMime='{}'",
                uploader, objectKey, request.contentType());

        // Construct standard PutObject request with required constraints
        PutObjectRequest objectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(objectKey)
                .contentType(request.contentType())
                .metadata(Map.of("uploader-id", uploader))
                .build();

        // Sign the URL with an explicit time-to-live (e.g., 15 minutes)
        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(15))
                .putObjectRequest(objectRequest)
                .build();

        PresignedPutObjectRequest presignedResult = s3Presigner.presignPutObject(presignRequest);

        PresignedUploadTicket ticket = new PresignedUploadTicket(
                objectKey,
                presignedResult.url().toString(),
                presignedResult.expiration().toEpochMilli(),
                presignedResult.httpRequest().headers()
        );

        return ResponseEntity.ok(ticket);
    }

    public record UploadTicketRequest(String filename, String contentType, long expectedSizeBytes) {}
    public record PresignedUploadTicket(String objectKey, String uploadUrl, long expirationEpochMs, Map<String, java.util.List<String>> signedHeaders) {}

    private String extractExtension(String filename) {
        int dot = filename.lastIndexOf('.');
        return dot > 0 ? filename.substring(dot + 1).toLowerCase() : "";
    }
}
```

---

## 4. File Download & Delivery Architectures

Delivering stored files back to HTTP clients requires balancing thread consumption, buffer limits, video seek capabilities, and edge CDN offloading.

---

### Approach 1: Direct Resource Return (`FileSystemResource` / `ByteArrayResource`)

#### Scenario
Serving static, small, or moderately sized assets (PDF invoices, generated reports, user profile avatars `< 5MB`) directly from local disks or memory caches.

#### Use-Case
Standard enterprise report download controllers where file sizes are small and memory consumption per connection is negligible.

#### Benefits Over Others
- **Automatic Content-Length Header:** Spring calculates the payload length and automatically attaches RFC 5987 compliant UTF-8 `Content-Disposition`.
- **Built-In Spring Caching & ETags:** Seamlessly integrates with Spring's `ShallowEtagHeaderFilter` for HTTP 304 Not Modified caching.

#### Production-Grade Code Example

```java
package com.example.filemgmt.download.direct;

import com.example.filemgmt.service.FileStorageService;
import lombok.RequiredArgsConstructor;
import org.springframework.core.io.Resource;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import java.nio.charset.StandardCharsets;

@RestController
@RequestMapping("/api/v1/downloads")
@RequiredArgsConstructor
public class DirectDownloadController {

    private final FileStorageService storageService;

    @GetMapping("/report/{filename:.+}")
    public ResponseEntity<Resource> downloadStaticDocument(@PathVariable String filename) {
        Resource fileResource = storageService.loadAsResource("reports/" + filename);

        // Build RFC 5987 / RFC 6266 Content-Disposition header safely escaping unicode
        ContentDisposition disposition = ContentDisposition.attachment()
                .filename(filename, StandardCharsets.UTF_8)
                .build();

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, disposition.toString())
                .contentType(MediaType.APPLICATION_PDF)
                .cacheControl(CacheControl.noCache().mustRevalidate())
                .body(fileResource);
    }
}
```

---

### Approach 2: Asynchronous Non-Blocking Streaming (`StreamingResponseBody`)

#### Scenario
Downloading large database CSV exports, zip archives, or telemetry dumps (100MB to 5GB) where loading the file into heap or holding the Tomcat container thread for the duration of a slow mobile client connection would lead to thread pool starvation.

#### Use-Case
Large file downloads where the server must release the worker thread back to Tomcat’s executor pool while streaming bytes over the wire.

#### Benefits Over Others
- **Thread Pool Liberation:** The primary Tomcat request processing thread is returned to the pool immediately upon returning the `StreamingResponseBody` lambda.
- **Controlled Buffer Bounds:** Bytes are pumped through a fixed 8KB or 64KB array, guaranteeing constant low heap consumption regardless of file size.
- **Dynamic Stream Generation:** Can stream data directly from an active database cursor or dynamic zip compressor without ever writing a physical file to disk.

#### Production-Grade Code Example

```java
package com.example.filemgmt.download.streaming;

import com.example.filemgmt.service.FileStorageService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

import java.io.InputStream;

@RestController
@RequestMapping("/api/v1/stream-downloads")
@RequiredArgsConstructor
@Slf4j
public class AsynchronousStreamingController {

    private final FileStorageService storageService;

    @GetMapping("/archive/{fileKey}")
    public ResponseEntity<StreamingResponseBody> streamLargeArchive(@PathVariable String fileKey) {
        Resource resource = storageService.loadAsResource(fileKey);

        StreamingResponseBody responseBody = outputStream -> {
            log.info("Initiating async response streaming for fileKey: {}", fileKey);
            byte[] buffer = new byte[16384]; // 16KB transfer buffer

            try (InputStream inputStream = resource.getInputStream()) {
                int bytesRead;
                while ((bytesRead = inputStream.read(buffer)) != -1) {
                    outputStream.write(buffer, 0, bytesRead);
                    outputStream.flush(); // Flush progressively to client
                }
            } catch (Exception e) {
                // Client dropped connection or network socket closed
                log.warn("Streaming pipe broken by client: {}", e.getMessage());
            }
        };

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + resource.getFilename() + "\"")
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                .body(responseBody);
    }
}
```

---

### Approach 3: HTTP 206 Partial Content Byte-Range Requests (Media Seeking)

#### Scenario
Streaming video (MP4/WebM) or audio files to web and mobile media players where users scrub back and forth along the video timeline, requiring the server to fulfill arbitrary byte range intervals (e.g., `bytes=5242880-10485759`).

#### Use-Case
Video streaming services, podcast platforms, and large PDF viewers requiring random-access seeking without downloading the full multi-hundred megabyte file first.

#### Benefits Over Others
- **Sub-Second Scrub Latency:** Video players instantly jump to any point in the stream by reading the index and fetching only the required slice.
- **Bandwidth Preservation:** Users who watch 30 seconds of a 2-hour movie only consume 15MB rather than 4GB.
- **Standard Protocol Compliance:** Fully adheres to RFC 7233 standard range specifications (`Accept-Ranges: bytes`, `Content-Range: bytes START-END/TOTAL`).

#### Production-Grade Code Example

```java
package com.example.filemgmt.download.partialrange;

import com.example.filemgmt.service.FileStorageService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

import java.io.IOException;
import java.io.InputStream;

@RestController
@RequestMapping("/api/v1/media")
@RequiredArgsConstructor
@Slf4j
public class MediaPartialRangeController {

    private final FileStorageService storageService;

    @GetMapping("/stream/{mediaId}")
    public ResponseEntity<StreamingResponseBody> streamMediaWithByteRanges(
            @PathVariable String mediaId,
            @RequestHeader(value = HttpHeaders.RANGE, required = false) String rangeHeader) throws IOException {

        Resource resource = storageService.loadAsResource("media/" + mediaId + ".mp4");
        long totalLength = resource.contentLength();

        if (rangeHeader == null || !rangeHeader.startsWith("bytes=")) {
            // Full file request (No Range requested)
            StreamingResponseBody fullBody = outputStream -> {
                try (InputStream is = resource.getInputStream()) {
                    is.transferTo(outputStream);
                }
            };

            return ResponseEntity.ok()
                    .header(HttpHeaders.ACCEPT_RANGES, "bytes")
                    .contentType(MediaType.parseMediaType("video/mp4"))
                    .contentLength(totalLength)
                    .body(fullBody);
        }

        // Parse Range Header: format is "bytes=START-END"
        String[] ranges = rangeHeader.replace("bytes=", "").split("-");
        long start = Long.parseLong(ranges[0]);
        long end = (ranges.length > 1 && !ranges[1].isBlank())
                ? Long.parseLong(ranges[1])
                : totalLength - 1;

        if (start >= totalLength || end >= totalLength || start > end) {
            return ResponseEntity.status(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE)
                    .header(HttpHeaders.CONTENT_RANGE, "bytes */" + totalLength)
                    .build();
        }

        long contentLength = end - start + 1;

        StreamingResponseBody rangeBody = outputStream -> {
            try (InputStream is = resource.getInputStream()) {
                long skipped = is.skip(start);
                if (skipped < start) {
                    throw new IOException("Failed to skip to required range offset");
                }

                byte[] buffer = new byte[8192];
                long bytesRemaining = contentLength;
                int bytesRead;

                while (bytesRemaining > 0 &&
                        (bytesRead = is.read(buffer, 0, (int) Math.min(buffer.length, bytesRemaining))) != -1) {
                    outputStream.write(buffer, 0, bytesRead);
                    bytesRemaining -= bytesRead;
                }
            }
        };

        return ResponseEntity.status(HttpStatus.PARTIAL_CONTENT)
                .header(HttpHeaders.ACCEPT_RANGES, "bytes")
                .header(HttpHeaders.CONTENT_RANGE, String.format("bytes %d-%d/%d", start, end, totalLength))
                .contentType(MediaType.parseMediaType("video/mp4"))
                .contentLength(contentLength)
                .body(rangeBody);
    }
}
```

---

### Approach 4: Pre-Signed CDN / Object Storage URL Redirection

#### Scenario
High-traffic production systems delivering protected media or enterprise artifacts where files reside on cloud storage (S3) behind edge CDNs (CloudFront / Cloudflare).

#### Use-Case
Download endpoints where Spring Boot enforces method security (`@PreAuthorize`), logs an audit record, and then redirects the client (HTTP 302 Found) to an ephemeral signed S3 or CDN URL.

#### Benefits Over Others
- **Zero App-Server Delivery Bandwidth:** The application does not stream a single file byte; delivery is handled by the cloud provider’s global edge network.
- **Edge Caching:** Signed URLs can be cached at the nearest CDN edge point of presence.
- **Granular Security:** Pre-signed URLs expire after a short time (e.g., 5 minutes), preventing link sharing.

#### Production-Grade Code Example

```java
package com.example.filemgmt.download.presigned;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;
import software.amazon.awssdk.services.s3.model.GetObjectRequest;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.GetObjectPresignRequest;

import java.net.URI;
import java.security.Principal;
import java.time.Duration;

@RestController
@RequestMapping("/api/v1/secure-access")
@RequiredArgsConstructor
@Slf4j
public class PreSignedDownloadRedirectController {

    private final S3Presigner s3Presigner;

    @Value("${aws.s3.bucket}")
    private String bucketName;

    @GetMapping("/download/{fileId}")
    @PreAuthorize("hasAuthority('DOCUMENT_READ')")
    public ResponseEntity<Void> getTemporaryDownloadUrl(
            @PathVariable String fileId,
            Principal principal) {

        log.info("Authorizing secure download for fileId='{}' by user='{}'", fileId, principal.getName());

        String s3Key = "documents/" + fileId;

        // Construct pre-signed GET request valid for only 5 minutes
        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(5))
                .getObjectRequest(GetObjectRequest.builder()
                        .bucket(bucketName)
                        .key(s3Key)
                        .responseContentDisposition("attachment; filename=\"document-" + fileId + ".pdf\"")
                        .build())
                .build();

        String signedUrl = s3Presigner.presignGetObject(presignRequest).url().toString();

        // Redirect client browser directly to S3 / CloudFront
        return ResponseEntity.status(HttpStatus.FOUND)
                .location(URI.create(signedUrl))
                .header(HttpHeaders.CACHE_CONTROL, "no-cache, no-store, max-age=0, must-revalidate")
                .build();
    }
}
```

---

## 5. Security & Threat Mitigation Pipelines

Handling untrusted user-submitted files represents one of the highest attack surfaces in modern web architectures (e.g., Remote Code Execution via uploaded web shells, Billion Laughs XML zip bombs, Cross-Site Scripting via SVG, and path traversal).

```
 Inbound Multipart File
          │
          ▼
 1. Size Verification (Payload <= 25MB)
          │
          ▼
 2. Defensive Path Sanitization (Strip ../, null bytes, unicode control chars)
          │
          ▼
 3. Extension Allowlist Check (Must match strict allowed set)
          │
          ▼
 4. Apache Tika Magic Byte Inspection (Verify real byte signature != spoofed header)
          │
          ▼
 5. ClamAV Daemon Anti-Virus Stream Inspection (Detect trojans, macros, payloads)
          │
          ▼
 Safe Persistence (Store outside web root with randomized UUID)
```

---

### Approach 1: Multi-Layer Validation (Extension + Apache Tika Magic Bytes)

#### Scenario
Strict enterprise document ingress where attackers attempt to bypass file upload filters by spoofing extensions (renaming `malware.exe` to `invoice.pdf`) or spoofing HTTP `Content-Type` headers.

#### Use-Case
Bank document portals, customer KYC verification, and public job application portals.

#### Benefits Over Others
- **Deep Signature Detection:** Reads the underlying magic numbers (e.g., `%PDF`, `\xFF\xD8\xFF` for JPEG) using Apache Tika's comprehensive MIME dictionary.
- **Header-Spoofing Immunity:** Ignores the user's declared HTTP `Content-Type` and detects the actual binary format.

#### Production-Grade Code Example

```java
package com.example.filemgmt.validation;

import lombok.extern.slf4j.Slf4j;
import org.apache.tika.Tika;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;
import java.util.Map;
import java.util.Set;

@Component
@Slf4j
public class DeepInspectionFileValidator {

    private final Tika tika = new Tika();

    // Strict mapping of allowed file extensions to their legitimate MIME magic byte signatures
    private static final Map<String, Set<String>> ALLOWED_EXTENSIONS_TO_MIMES = Map.of(
            "pdf", Set.of("application/pdf"),
            "jpg", Set.of("image/jpeg"),
            "jpeg", Set.of("image/jpeg"),
            "png", Set.of("image/png"),
            "docx", Set.of("application/vnd.openxmlformats-officedocument.wordprocessingml.document"),
            "xlsx", Set.of("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")
    );

    public void validate(MultipartFile file) {
        if (file == null || file.isEmpty()) {
            throw new SecurityValidationException("Upload rejected: File payload is empty");
        }

        String originalFilename = file.getOriginalFilename();
        if (originalFilename == null || originalFilename.isBlank()) {
            throw new SecurityValidationException("Upload rejected: Missing filename");
        }

        // 1. Extension Allowlist Check
        String extension = extractExtension(originalFilename);
        if (!ALLOWED_EXTENSIONS_TO_MIMES.containsKey(extension)) {
            throw new SecurityValidationException("Upload rejected: Unsupported extension ." + extension);
        }

        // 2. Real Magic Byte Detection via Apache Tika
        try (InputStream is = file.getInputStream()) {
            String detectedMime = tika.detect(is, originalFilename);
            Set<String> validMimesForExtension = ALLOWED_EXTENSIONS_TO_MIMES.get(extension);

            if (!validMimesForExtension.contains(detectedMime)) {
                log.warn("MIME spoofing detected! Filename='{}', Extension='{}', ClaimedType='{}', DetectedRealType='{}'",
                        originalFilename, extension, file.getContentType(), detectedMime);
                throw new SecurityValidationException("File signature mismatch! File pretends to be ." +
                        extension + " but contains binary signature of " + detectedMime);
            }

            log.info("File passed deep byte inspection: '{}' detected as '{}'", originalFilename, detectedMime);

        } catch (IOException e) {
            throw new SecurityValidationException("Failed to read file stream for byte inspection", e);
        }
    }

    private String extractExtension(String filename) {
        int dot = filename.lastIndexOf('.');
        return dot > 0 ? filename.substring(dot + 1).toLowerCase() : "";
    }
}
```

---

### Approach 2: In-Flight Antivirus Stream Scanning (ClamAV Daemon)

#### Scenario
Zero-trust public file ingestion platforms receiving documents that could carry embedded macro viruses, executable trojans, or zero-day malware.

#### Use-Case
Enterprise document workflows where every inbound file must pass an automated signature scanner before being promoted to persistent storage.

#### Benefits Over Others
- **In-Memory Streaming Scan:** Uses ClamAV's `zINSTREAM` socket protocol, streaming bytes directly over TCP without writing untrusted malware to the host container’s disk.
- **Fail-Closed Security Posture:** If the antivirus daemon is unreachable or throws a socket timeout, the upload is rejected immediately.

#### Production-Grade Code Example

```java
package com.example.filemgmt.validation.antivirus;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;

@Service
@Slf4j
public class ClamAvStreamingScanner {

    @Value("${clamav.host:localhost}")
    private String clamHost;

    @Value("${clamav.port:3310}")
    private int clamPort;

    @Value("${clamav.timeout-ms:5000}")
    private int timeoutMs;

    public void scanOrThrow(MultipartFile file) {
        log.info("Dispatching file '{}' ({} bytes) to ClamAV daemon for virus inspection",
                file.getOriginalFilename(), file.getSize());

        try (Socket socket = new Socket()) {
            socket.connect(new InetSocketAddress(clamHost, clamPort), timeoutMs);
            socket.setSoTimeout(timeoutMs);

            try (OutputStream out = new BufferedOutputStream(socket.getOutputStream());
                 InputStream in = new BufferedInputStream(socket.getInputStream())) {

                // Send INSTREAM command (zINSTREAM\0)
                out.write("zINSTREAM\0".getBytes(StandardCharsets.US_ASCII));
                out.flush();

                // Stream chunks: [4 bytes length][chunk data]
                byte[] chunkBuffer = new byte[8192];
                try (InputStream fileStream = file.getInputStream()) {
                    int read;
                    while ((read = fileStream.read(chunkBuffer)) != -1) {
                        byte[] lengthHeader = ByteBuffer.allocate(4).putInt(read).array();
                        out.write(lengthHeader);
                        out.write(chunkBuffer, 0, read);
                    }
                }

                // Send 4 zero bytes to terminate stream
                out.write(new byte[]{0, 0, 0, 0});
                out.flush();

                // Read ClamAV response (e.g., "stream: OK" or "stream: Eicar-Test-Signature FOUND")
                ByteArrayOutputStream responseBuffer = new ByteArrayOutputStream();
                byte[] respChunk = new byte[256];
                int respRead;
                while ((respRead = in.read(respChunk)) != -1) {
                    responseBuffer.write(respChunk, 0, respRead);
                }

                String reply = responseBuffer.toString(StandardCharsets.US_ASCII).trim();
                log.info("ClamAV response for '{}': {}", file.getOriginalFilename(), reply);

                if (reply.contains("FOUND")) {
                    log.error("MALWARE DETECTED in upload '{}'! Response: {}", file.getOriginalFilename(), reply);
                    throw new SecurityException("Upload rejected: Malware signature detected in file payload!");
                }

                if (!reply.contains("OK")) {
                    throw new IllegalStateException("ClamAV unexpected scanner response: " + reply);
                }
            }
        } catch (IOException e) {
            log.error("ClamAV virus scan communication failed! Enforcing fail-closed policy", e);
            throw new RuntimeException("Antivirus scanning daemon is currently unavailable. Upload blocked.", e);
        }
    }
}
```

---

### Approach 3: Defensive Path Traversal & Filename Sanitization

#### Scenario
Mitigating malicious filenames containing directory traversal sequences (`../../../../etc/passwd`), null bytes (`exploit.php\0.jpg`), or dangerous OS shell characters (`|`, `;`, `&`, `$`).

#### Use-Case
Mandatory defense-in-depth across all storage services where user-controlled strings interact with file paths.

#### Benefits Over Others
- **Prevents Arbitrary File Overwrites:** Attackers cannot escape the application root to overwrite critical system files (`/root/.ssh/authorized_keys`).
- **Eliminates OS Shell Injection:** Sanitizes metacharacters that could trigger command execution in downstream image or PDF processing scripts.

#### Production-Grade Code Example

```java
package com.example.filemgmt.validation;

import org.springframework.stereotype.Component;

import java.nio.file.Path;
import java.util.regex.Pattern;

@Component
public class PathSanitizationGuard {

    private static final Pattern ILLEGAL_CHARACTERS = Pattern.compile("[\\\\/:*?\"<>|\\x00-\\x1F]");
    private static final Pattern MULTIPLE_DOTS = Pattern.compile("\\.{2,}");

    /**
     * Sanitizes a client-provided filename into a safe base token.
     */
    public String sanitizeFilename(String filename) {
        if (filename == null || filename.isBlank()) {
            return "unnamed_file";
        }

        // 1. Strip path elements
        String cleaned = Path.of(filename).getFileName().toString();

        // 2. Strip dangerous control characters and shell metacharacters
        cleaned = ILLEGAL_CHARACTERS.matcher(cleaned).replaceAll("_");

        // 3. Remove traversal dots (..)
        cleaned = MULTIPLE_DOTS.matcher(cleaned).replaceAll("_");

        // 4. Strip leading dots (hidden files in UNIX)
        while (cleaned.startsWith(".")) {
            cleaned = cleaned.substring(1);
        }

        return cleaned.trim().isEmpty() ? "unnamed_file" : cleaned.trim();
    }

    /**
     * Confirms that a resolved child path is strictly confined within a designated parent root.
     */
    public Path resolveSecurely(Path rootDir, String userSuppliedPath) {
        Path resolved = rootDir.resolve(userSuppliedPath).normalize();
        if (!resolved.startsWith(rootDir.normalize())) {
            throw new SecurityException("CRITICAL: Directory traversal attempt detected! Path: " + userSuppliedPath);
        }
        return resolved;
    }
}
```

---

## 6. Asynchronous Background File Processing

For large batch files (e.g., a 2GB corporate payroll CSV containing 1,000,000 records), synchronous processing inside a controller will time out the client socket. Modern Spring Boot 3.x utilizes `@Async` combined with Java 21 Virtual Threads or configured `ThreadPoolTaskExecutor`s.

```java
package com.example.filemgmt.async;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.Resource;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.concurrent.CompletableFuture;

@Service
@RequiredArgsConstructor
@Slf4j
public class AsynchronousFileProcessor {

    @Async("fileProcessingExecutor")
    public CompletableFuture<BatchProcessingReport> processLargeCsvAsync(String jobId, Resource csvResource) {
        log.info("Starting asynchronous processing for job '{}' on thread '{}'",
                jobId, Thread.currentThread().getName());

        long recordsProcessed = 0;
        long failureCount = 0;
        Instant startTime = Instant.now();

        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(csvResource.getInputStream(), StandardCharsets.UTF_8))) {

            String line;
            boolean isHeader = true;

            while ((line = reader.readLine()) != null) {
                if (isHeader) {
                    isHeader = false;
                    continue;
                }

                try {
                    parseAndPersistRecord(line);
                    recordsProcessed++;
                } catch (Exception e) {
                    failureCount++;
                    log.warn("Failed to parse CSV line in job '{}': {}", jobId, line, e);
                }

                // Periodically check for thread interruption
                if (Thread.currentThread().isInterrupted()) {
                    log.warn("Job '{}' was cancelled mid-execution", jobId);
                    break;
                }
            }

            BatchProcessingReport report = new BatchProcessingReport(
                    jobId,
                    "COMPLETED",
                    recordsProcessed,
                    failureCount,
                    startTime,
                    Instant.now()
            );

            log.info("Completed async processing for job '{}': {} success, {} failures",
                    jobId, recordsProcessed, failureCount);

            return CompletableFuture.completedFuture(report);

        } catch (Exception e) {
            log.error("Fatal failure processing CSV job '{}'", jobId, e);
            return CompletableFuture.failedFuture(e);
        }
    }

    private void parseAndPersistRecord(String line) {
        // Business logic parsing
    }

    public record BatchProcessingReport(
            String jobId,
            String status,
            long successCount,
            long errorCount,
            Instant startedAt,
            Instant completedAt
    ) {}
}
```

---

## 7. Image Manipulation & Optimization Pipelines

Transforming uploaded user images into standardized thumbnails and modern formats (WebP) saves substantial CDN delivery costs.

```java
package com.example.filemgmt.image;

import lombok.extern.slf4j.Slf4j;
import net.coobird.thumbnailator.Thumbnails;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;

@Service
@Slf4j
public class ImageOptimizationService {

    /**
     * Generates an aspect-ratio-preserved JPEG thumbnail.
     */
    public byte[] createThumbnail(MultipartFile sourceImage, int targetWidth, int targetHeight) throws IOException {
        try (InputStream in = sourceImage.getInputStream();
             ByteArrayOutputStream out = new ByteArrayOutputStream()) {

            BufferedImage img = ImageIO.read(in);
            if (img == null) {
                throw new IllegalArgumentException("Payload is not a valid decoded image");
            }

            Thumbnails.of(img)
                    .size(targetWidth, targetHeight)
                    .keepAspectRatio(true)
                    .outputFormat("jpg")
                    .outputQuality(0.85) // 85% JPEG quality achieves great compression with negligible visual loss
                    .toOutputStream(out);

            return out.toByteArray();
        }
    }

    /**
     * Transcodes an incoming image directly to WebP.
     */
    public byte[] convertToOptimizedWebP(MultipartFile sourceImage) throws IOException {
        try (InputStream in = sourceImage.getInputStream();
             ByteArrayOutputStream out = new ByteArrayOutputStream()) {

            Thumbnails.of(in)
                    .scale(1.0)
                    .outputFormat("webp")
                    .outputQuality(0.80)
                    .toOutputStream(out);

            return out.toByteArray();
        }
    }
}
```

---

## 8. Comprehensive Configuration Reference

### `application.yml` (Production Standard)

```yaml
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 25MB               # Strict per-file maximum
      max-request-size: 100MB           # Total multipart request body limit
      file-size-threshold: 8KB          # Parts above 8KB are spooled to disk, protecting JVM heap
      location: ${java.io.tmpdir}/spring-spool  # Temp directory for spooled parts
      resolve-lazily: false             # Parse multipart headers eagerly
  mvc:
    async:
      request-timeout: 300000           # 5-minute timeout for StreamingResponseBody

# Custom Application File Management Properties
app:
  storage:
    type: s3                            # local | s3 | gcs
    base-url: https://api.mycompany.com
    local:
      base-dir: /var/app/data/uploads
    chunk-staging-dir: /var/app/data/staging

aws:
  region: us-east-1
  s3:
    bucket: prod-enterprise-assets
    cdn-domain: https://d111111abcdef8.cloudfront.net

clamav:
  host: clamav.internal.vpc
  port: 3310
  timeout-ms: 5000
```

---

## 9. ⚠️ Interview Traps & Production Gotchas

### Trap 1: Controller Exception Handlers Cannot Catch `MaxUploadSizeExceededException`

> [!CAUTION]
> When a client uploads a file larger than `spring.servlet.multipart.max-file-size`, the exception is thrown by the Servlet container's `StandardServletMultipartResolver` **before** the request ever reaches the target Controller method. A `try-catch` block inside the controller will **never** execute.

```java
// ❌ WRONG: Method-level try-catch is unreachable!
@PostMapping("/upload")
public ResponseEntity<?> upload(@RequestParam("file") MultipartFile file) {
    try {
        storageService.store(file);
    } catch (MaxUploadSizeExceededException e) {
        // DEAD CODE: Spring throws this before invoking this method!
        return ResponseEntity.badRequest().body("File too large");
    }
}

// ✅ CORRECT: Handle globally in @RestControllerAdvice
@RestControllerAdvice
@Slf4j
public class MultipartGlobalExceptionHandler {

    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ResponseEntity<ProblemDetail> handleSizeLimitExceeded(MaxUploadSizeExceededException ex) {
        log.warn("File upload rejected: Exceeded maximum allowed payload size");
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.PAYLOAD_TOO_LARGE,
                "The uploaded file exceeds the configured maximum upload limit."
        );
        problem.setTitle("Payload Too Large");
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE).body(problem);
    }
}
```

---

### Trap 2: Tomcat Connector `maxPostSize` Bottleneck

Even if you set `spring.servlet.multipart.max-file-size: 500MB`, your uploads may still be rejected with an immediate connection reset if embedded Tomcat's HTTP connector has its default `maxPostSize` (typically 2MB) intact.

```java
// ✅ Programmatic customization of Tomcat's Connector
@Configuration
public class TomcatFileUploadCustomizer {

    @Bean
    public TomcatServletWebServerFactory tomcatCustomizer() {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.addConnectorCustomizers(connector -> {
            connector.setMaxPostSize(500 * 1024 * 1024); // 500 MB
            connector.setMaxSavePostSize(-1);            // Unlimited post buffer
        });
        return factory;
    }
}
```

---

### Trap 3: Memory Exhaustion via `file.getBytes()`

Calling `file.getBytes()` loads the **entire** binary into a contiguous `byte[]` on the JVM young/old generation heap. If 50 concurrent users upload a 100MB file simultaneously, your application requires `50 * 100MB = 5GB` of immediate heap allocation, triggering severe Stop-The-World GC pauses and immediate `java.lang.OutOfMemoryError: Java heap space`.

```java
// ❌ ANTI-PATTERN: Heap-exhaustion trap
byte[] bytes = file.getBytes(); // Allocates 200MB byte array directly on heap!
Files.write(targetPath, bytes);

// ✅ CORRECT: Stream via NIO or transferTo()
try (InputStream is = file.getInputStream()) {
    Files.copy(is, targetPath, StandardCopyOption.REPLACE_EXISTING);
}
// OR
file.transferTo(targetPath);
```

---

### Trap 4: Calling `transferTo()` Multiple Times

`MultipartFile.transferTo()` delegates to the underlying Servlet `Part.write()` or consumes the `InputStream`. Calling it a second time (e.g., once to a temporary virus-scanning file and once to the permanent store) will throw an `IllegalStateException: File has already been moved or stream already consumed`.

---

## 10. 🔍 Deep Dive: DispatcherServlet & Multipart Internals

```
Inbound HTTP Request
         │
         ▼
DispatcherServlet.doDispatch()
         │
         ├─── Step 1: checkMultipart(request)
         │           Calls StandardServletMultipartResolver.isMultipart(request)
         │           Checks Content-Type: multipart/form-data
         │
         ├─── Step 2: resolveMultipart(request)
         │           Wraps HttpServletRequest in StandardMultipartHttpServletRequest
         │           Triggers request.getParts()
         │
         ├─── Step 3: Tomcat MultipartStream Parsing
         │           Boundary tokens parsed from byte stream
         │           Parts < file-size-threshold stored in memory (ByteArray)
         │           Parts > file-size-threshold spooled to /tmp/spring-uploads
         │
         ▼
Controller Method Invocation via HandlerAdapter
         │
         ├─── Resolves @RequestParam("file") -> StandardMultipartFile
         └─── Resolves @RequestPart("data") -> Jackson HttpMessageConverter
```

---

## 11. 📝 Architectural Comparison Matrix

| Dimension | Standard `@RequestParam` | Composite `@RequestPart` | Resumable Chunked | Direct S3 Pre-Signed |
|---|---|---|---|---|
| **Best Used For** | Basic single/multi file uploads | File + nested JSON metadata | Large files (>100MB), video | High volume SaaS, huge files |
| **App Server Bandwidth** | 100% consumed | 100% consumed | 100% consumed | **0% consumed (Bypasses server)** |
| **App Server Heap** | Bounded by threshold | Bounded by threshold | Extremely low (5MB chunks) | **Zero (Metadata only)** |
| **Network Resilience** | Fails on socket drop | Fails on socket drop | **Resumes from last chunk** | Client retries via AWS S3 |
| **Client Complexity** | Minimal HTML Form | Requires `FormData` object | Requires chunking client | Requires multi-step API call |

---

## 12. ❓ Practice Questions & Real-World Scenarios

1. **Why does Spring Boot require both `max-file-size` and `max-request-size` configuration properties?**
   - *Answer:* `max-file-size` dictates the maximum ceiling for any *individual* file in the multipart request, whereas `max-request-size` governs the cumulative total size of all files plus form fields combined in the entire HTTP request payload.

2. **How do you defend against an attacker uploading a file named `../../../../windows/system32/cmd.exe`?**
   - *Answer:* First, sanitize the raw filename using `Path.of(name).getFileName().toString()` and strip illegal characters. Second, resolve the target path against your absolute storage root and call `.normalize()`. Finally, verify using `.startsWith(rootStorageDirectory)`. If false, abort immediately.

3. **When would you select `StreamingResponseBody` over `ResponseEntity<Resource>`?**
   - *Answer:* Use `StreamingResponseBody` for very large files or dynamically generated streams (e.g., zip files created on the fly from database records) because it releases the container's Tomcat worker thread back to the pool, allowing an async thread to pump bytes through a small fixed buffer without holding up the web server.

4. **Why is verifying `MultipartFile.getContentType()` insufficient for malware prevention?**
   - *Answer:* `getContentType()` simply mirrors the client's HTTP request header string (e.g., `Content-Type: image/jpeg`), which an attacker can easily forge. True security requires deep inspection using magic byte analysis (via Apache Tika) to inspect the binary signature headers of the file.
