# 11 — File Management in Spring Boot

> **Module Goal:** Master every aspect of file handling in production Spring Boot applications — from simple uploads to S3 streaming, security hardening, and resumable chunked uploads.

---

## Table of Contents

1. [File Upload Fundamentals](#1-file-upload-fundamentals)
2. [File Storage Strategies](#2-file-storage-strategies)
3. [AWS S3 Integration](#3-aws-s3-integration)
4. [File Download & Streaming](#4-file-download--streaming)
5. [Async File Processing](#5-async-file-processing)
6. [File Metadata Management](#6-file-metadata-management)
7. [Image Processing](#7-image-processing)
8. [Temporary Files](#8-temporary-files)
9. [Security Considerations](#9-security-considerations)
10. [Production Patterns](#10-production-patterns)
11. [Configuration Reference](#11-configuration-reference)
12. [⚠️ Interview Traps & Gotchas](#️-interview-traps--gotchas)
13. [🔍 Deep Dive — Internals](#-deep-dive--internals)
14. [📝 Summary & Key Takeaways](#-summary--key-takeaways)
15. [❓ Practice Questions](#-practice-questions)

---

## 1. File Upload Fundamentals

### 1.1 The MultipartFile API

`MultipartFile` is Spring's abstraction over a file received in a multipart/form-data HTTP request. It wraps the underlying servlet `Part` (Servlet 3.0+) or Commons FileUpload part.

```java
public interface MultipartFile extends InputStreamSource {

    String getName();                     // form field name (e.g. "file")
    String getOriginalFilename();         // browser-supplied filename  ⚠️ UNSAFE
    String getContentType();             // MIME type from Content-Type header ⚠️ UNSAFE
    boolean isEmpty();
    long getSize();                      // size in bytes
    byte[] getBytes() throws IOException; // load ALL bytes into heap – careful with large files
    InputStream getInputStream() throws IOException;
    void transferTo(File dest) throws IOException, IllegalStateException;
    void transferTo(Path dest) throws IOException, IllegalStateException; // Spring 5.1+
    Resource getResource();
}
```

> [!WARNING]
> `getOriginalFilename()` and `getContentType()` are **client-controlled** values. Never trust them for security decisions. A malicious client can set `Content-Type: image/jpeg` on a `.exe` file.

### 1.2 Multipart Configuration

**application.yml**
```yaml
spring:
  servlet:
    multipart:
      enabled: true                    # default: true
      max-file-size: 10MB              # per-file limit (default: 1MB)
      max-request-size: 50MB           # total request body limit (default: 10MB)
      file-size-threshold: 2KB         # size above which file is spooled to disk (default: 0)
      location: /tmp/spring-uploads    # temp directory for spooled parts
      resolve-lazily: false            # parse multipart eagerly (default: false)
```

> [!NOTE]
> `file-size-threshold` is critical for memory management. Files smaller than this threshold are kept in memory; larger ones are written to `location` on disk as temp files. Set this to a low value (e.g., 4KB) in production.

**Programmatic Configuration** (when you need dynamic limits):
```java
@Configuration
public class MultipartConfig {

    @Bean
    public MultipartConfigElement multipartConfigElement() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        factory.setMaxFileSize(DataSize.ofMegabytes(10));
        factory.setMaxRequestSize(DataSize.ofMegabytes(50));
        factory.setFileSizeThreshold(DataSize.ofKilobytes(4));
        return factory.createMultipartConfig();
    }
}
```

**Tomcat-specific limits** (must also be configured at the connector level for very large uploads):
```java
@Configuration
public class TomcatConfig {

    @Bean
    public TomcatServletWebServerFactory tomcatFactory() {
        return new TomcatServletWebServerFactory() {
            @Override
            protected void customizeConnector(Connector connector) {
                super.customizeConnector(connector);
                // Default is 2MB — increase for large file uploads
                connector.setMaxPostSize(100 * 1024 * 1024); // 100 MB
                connector.setMaxSavePostSize(-1); // unlimited
            }
        };
    }
}
```

### 1.3 `@RequestParam` vs `@RequestPart`

| Feature | `@RequestParam` | `@RequestPart` |
|---|---|---|
| Handles `MultipartFile` | ✅ Yes | ✅ Yes |
| Content-Type awareness | ❌ No | ✅ Yes (inspects part's `Content-Type`) |
| Complex types via `HttpMessageConverter` | ❌ No | ✅ Yes (e.g., JSON body part) |
| Simple string params | ✅ Yes | ❌ No |
| Best for | Simple file uploads | Mixed multipart (file + JSON metadata) |

```java
// @RequestParam — simple file upload
@PostMapping("/upload/simple")
public ResponseEntity<String> uploadSimple(
    @RequestParam("file") MultipartFile file,
    @RequestParam("description") String description) {
    // ...
}

// @RequestPart — file + JSON metadata in single request
@PostMapping(value = "/upload/rich", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<FileResponse> uploadRich(
    @RequestPart("file") MultipartFile file,
    @RequestPart("metadata") FileMetadataRequest metadata) {  // Deserialized via Jackson
    // metadata is automatically deserialized from the JSON part
    // ...
}
```

### 1.4 Single File Upload

```java
@RestController
@RequestMapping("/api/files")
@RequiredArgsConstructor
@Slf4j
public class FileUploadController {

    private final FileStorageService fileStorageService;
    private final FileValidator fileValidator;

    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<FileUploadResponse> uploadFile(
            @RequestParam("file") MultipartFile file) {

        log.info("Received upload request: filename={}, size={}, contentType={}",
                file.getOriginalFilename(), file.getSize(), file.getContentType());

        // Validate before processing
        fileValidator.validate(file);

        // Store the file
        StoredFile stored = fileStorageService.store(file);

        return ResponseEntity.status(HttpStatus.CREATED)
                .body(FileUploadResponse.builder()
                        .fileId(stored.getId())
                        .filename(stored.getStoredName())
                        .size(stored.getSize())
                        .url(stored.getAccessUrl())
                        .build());
    }
}
```

### 1.5 Multiple File Upload

```java
@PostMapping(value = "/upload/multiple", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<List<FileUploadResponse>> uploadMultiple(
        @RequestParam("files") List<MultipartFile> files) {

    if (files.isEmpty()) {
        throw new IllegalArgumentException("No files provided");
    }
    if (files.size() > 10) {
        throw new IllegalArgumentException("Maximum 10 files per request");
    }

    List<FileUploadResponse> responses = files.stream()
            .map(file -> {
                fileValidator.validate(file);
                StoredFile stored = fileStorageService.store(file);
                return FileUploadResponse.from(stored);
            })
            .collect(Collectors.toList());

    return ResponseEntity.status(HttpStatus.CREATED).body(responses);
}
```

### 1.6 File Validation

```java
@Component
public class FileValidator {

    private static final long MAX_FILE_SIZE = 10 * 1024 * 1024; // 10 MB
    private static final Set<String> ALLOWED_EXTENSIONS = Set.of(
            "jpg", "jpeg", "png", "gif", "webp", "pdf", "docx", "xlsx", "csv"
    );
    private static final Set<String> ALLOWED_MIME_TYPES = Set.of(
            "image/jpeg", "image/png", "image/gif", "image/webp",
            "application/pdf",
            "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            "text/csv"
    );

    public void validate(MultipartFile file) {
        if (file == null || file.isEmpty()) {
            throw new FileValidationException("File is empty or null");
        }

        validateFileSize(file);
        validateFilename(file);
        validateMimeType(file);
    }

    private void validateFileSize(MultipartFile file) {
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new FileValidationException(
                String.format("File size %d bytes exceeds maximum allowed %d bytes",
                    file.getSize(), MAX_FILE_SIZE));
        }
    }

    private void validateFilename(MultipartFile file) {
        String originalFilename = file.getOriginalFilename();
        if (originalFilename == null || originalFilename.isBlank()) {
            throw new FileValidationException("Filename is missing");
        }

        // Sanitize and validate extension
        String sanitized = sanitizeFilename(originalFilename);
        String extension = getExtension(sanitized).toLowerCase();

        if (!ALLOWED_EXTENSIONS.contains(extension)) {
            throw new FileValidationException(
                "File type '" + extension + "' is not allowed");
        }
    }

    private void validateMimeType(MultipartFile file) {
        // 1. Check declared MIME type (client-provided — not fully trustworthy)
        String declaredMime = file.getContentType();
        if (declaredMime == null || !ALLOWED_MIME_TYPES.contains(declaredMime)) {
            throw new FileValidationException(
                "Declared content type '" + declaredMime + "' is not allowed");
        }

        // 2. Detect actual MIME from file magic bytes (Apache Tika recommended)
        try (InputStream is = file.getInputStream()) {
            Tika tika = new Tika();
            String detectedMime = tika.detect(is);
            if (!ALLOWED_MIME_TYPES.contains(detectedMime)) {
                throw new FileValidationException(
                    "Detected content type '" + detectedMime + "' is not allowed");
            }
        } catch (IOException e) {
            throw new FileValidationException("Could not read file content for validation");
        }
    }

    public String sanitizeFilename(String filename) {
        if (filename == null) return "unknown";
        // Remove path separators and dangerous characters
        return filename
                .replaceAll("[/\\\\:*?\"<>|]", "_")  // Illegal filename chars
                .replaceAll("\\.\\.", "_")              // Path traversal (..)
                .replaceAll("^\\.", "_")               // Hidden files
                .trim();
    }

    private String getExtension(String filename) {
        int dotIndex = filename.lastIndexOf('.');
        return dotIndex > 0 ? filename.substring(dotIndex + 1) : "";
    }
}
```

---

## 2. File Storage Strategies

### 2.1 Local Filesystem Storage

```java
@Service
@Slf4j
public class LocalFileStorageService implements FileStorageService {

    private final Path storageRoot;
    private final String baseUrl;

    public LocalFileStorageService(
            @Value("${app.storage.local.path}") String storagePath,
            @Value("${app.storage.base-url}") String baseUrl) {
        this.storageRoot = Paths.get(storagePath).toAbsolutePath().normalize();
        this.baseUrl = baseUrl;

        // Create root directory if it doesn't exist
        try {
            Files.createDirectories(this.storageRoot);
        } catch (IOException e) {
            throw new StorageException("Could not initialize storage directory", e);
        }

        log.info("Local storage initialized at: {}", this.storageRoot);
    }

    @Override
    public StoredFile store(MultipartFile file) {
        String originalName = file.getOriginalFilename();
        String extension = extractExtension(originalName);
        String storedName = generateStoredName(extension);

        // Organize by date to avoid too many files in one directory
        Path dateDir = resolveDate SubDirectory();
        Path targetPath = dateDir.resolve(storedName);

        // CRITICAL: Validate the resolved path is within storage root
        validatePathWithinRoot(targetPath);

        try {
            Files.createDirectories(dateDir);
            file.transferTo(targetPath);

            log.info("File stored: original={}, stored={}, size={}",
                    originalName, targetPath, file.getSize());

            return StoredFile.builder()
                    .id(UUID.randomUUID().toString())
                    .originalName(originalName)
                    .storedName(storedName)
                    .relativePath(storageRoot.relativize(targetPath).toString())
                    .size(file.getSize())
                    .contentType(file.getContentType())
                    .accessUrl(buildAccessUrl(storedName))
                    .build();

        } catch (IOException e) {
            throw new StorageException("Failed to store file: " + originalName, e);
        }
    }

    @Override
    public Resource loadAsResource(String relativePath) {
        Path filePath = storageRoot.resolve(relativePath).normalize();

        // CRITICAL: Path traversal prevention
        validatePathWithinRoot(filePath);

        Resource resource = new FileSystemResource(filePath);
        if (!resource.exists() || !resource.isReadable()) {
            throw new FileNotFoundException("File not found: " + relativePath);
        }
        return resource;
    }

    @Override
    public void delete(String relativePath) {
        Path filePath = storageRoot.resolve(relativePath).normalize();
        validatePathWithinRoot(filePath);
        try {
            Files.deleteIfExists(filePath);
        } catch (IOException e) {
            throw new StorageException("Failed to delete file: " + relativePath, e);
        }
    }

    private void validatePathWithinRoot(Path path) {
        if (!path.startsWith(storageRoot)) {
            throw new SecurityException(
                "Path traversal attempt detected! Path '" + path +
                "' is outside storage root '" + storageRoot + "'");
        }
    }

    private Path resolveDateSubDirectory() {
        LocalDate today = LocalDate.now();
        return storageRoot
                .resolve(String.valueOf(today.getYear()))
                .resolve(String.format("%02d", today.getMonthValue()))
                .resolve(String.format("%02d", today.getDayOfMonth()));
    }

    private String generateStoredName(String extension) {
        // UUID-based naming: no collisions, no guessability
        return UUID.randomUUID().toString() +
               (extension.isEmpty() ? "" : "." + extension);
    }

    private String extractExtension(String filename) {
        if (filename == null) return "";
        int dotIndex = filename.lastIndexOf('.');
        return dotIndex > 0 ? filename.substring(dotIndex + 1).toLowerCase() : "";
    }

    private String buildAccessUrl(String storedName) {
        return baseUrl + "/api/files/download/" + storedName;
    }
}
```

### 2.2 Path Traversal — Attack & Defense

```
Attack example:
  filename = "../../etc/passwd"
  Naive code: Paths.get(storageRoot + "/" + filename)
  Result: /var/uploads/../../etc/passwd → /etc/passwd  ← FILE LEAK!
```

**Defense pattern:**
```java
// ALWAYS normalize and validate
Path resolvedPath = storageRoot.resolve(userInput).normalize();

if (!resolvedPath.startsWith(storageRoot)) {
    throw new SecurityException("Path traversal detected!");
}
```

### 2.3 File Naming Strategies

| Strategy | Pros | Cons | Use Case |
|---|---|---|---|
| UUID | Collision-free, unguessable | Long, no semantics | Default choice |
| SHA-256 hash | Dedup built-in | Collision (tiny risk) | Dedup storage |
| Original name | Human-readable | Collision, injection risk | Never in isolation |
| Timestamp + UUID | Sortable | Slightly longer | Log/audit files |
| Content hash (MD5) | Dedup, integrity check | MD5 collisions | Legacy systems |

```java
// UUID (recommended default)
String name = UUID.randomUUID() + "." + extension;

// Content hash (SHA-256) for deduplication
MessageDigest digest = MessageDigest.getInstance("SHA-256");
byte[] hash = digest.digest(file.getBytes());
String hexHash = HexFormat.of().formatHex(hash);
String name = hexHash + "." + extension;

// Timestamp + UUID for sortable filenames
String name = Instant.now().toEpochMilli() + "-" + UUID.randomUUID() + "." + extension;
```

---

## 3. AWS S3 Integration

### 3.1 Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.25.0</version>
</dependency>
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3-transfer-manager</artifactId>
    <version>2.25.0</version>
</dependency>
<!-- For pre-signed URLs -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3-presigner</artifactId>
    <version>2.25.0</version>
</dependency>
```

### 3.2 S3Client Configuration

```java
@Configuration
public class S3Config {

    @Value("${aws.region:us-east-1}")
    private String region;

    @Value("${aws.s3.endpoint:#{null}}")   // For LocalStack
    private String endpoint;

    @Value("${aws.access-key-id:#{null}}")
    private String accessKeyId;

    @Value("${aws.secret-access-key:#{null}}")
    private String secretAccessKey;

    @Bean
    public S3Client s3Client() {
        S3ClientBuilder builder = S3Client.builder()
                .region(Region.of(region))
                .httpClientBuilder(ApacheHttpClient.builder()
                        .maxConnections(100)
                        .connectionTimeout(Duration.ofSeconds(10))
                        .socketTimeout(Duration.ofSeconds(30)));

        // Use explicit credentials (for local dev / LocalStack)
        if (accessKeyId != null && secretAccessKey != null) {
            builder.credentialsProvider(
                StaticCredentialsProvider.create(
                    AwsBasicCredentials.create(accessKeyId, secretAccessKey)));
        } else {
            // Production: use IAM role / instance profile
            builder.credentialsProvider(DefaultCredentialsProvider.create());
        }

        // LocalStack / custom endpoint
        if (endpoint != null) {
            builder.endpointOverride(URI.create(endpoint))
                   .forcePathStyle(true); // required for LocalStack
        }

        return builder.build();
    }

    @Bean
    public S3Presigner s3Presigner() {
        S3Presigner.Builder builder = S3Presigner.builder()
                .region(Region.of(region));

        if (accessKeyId != null && secretAccessKey != null) {
            builder.credentialsProvider(
                StaticCredentialsProvider.create(
                    AwsBasicCredentials.create(accessKeyId, secretAccessKey)));
        }
        if (endpoint != null) {
            builder.endpointOverride(URI.create(endpoint));
        }
        return builder.build();
    }
}
```

### 3.3 S3 File Storage Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class S3FileStorageService implements FileStorageService {

    private final S3Client s3Client;
    private final S3Presigner s3Presigner;

    @Value("${aws.s3.bucket}")
    private String bucketName;

    @Value("${aws.s3.presigned-url-expiry-minutes:60}")
    private int presignedUrlExpiryMinutes;

    @Override
    public StoredFile store(MultipartFile file) {
        String originalName = file.getOriginalFilename();
        String extension = extractExtension(originalName);
        String key = buildS3Key(extension);

        try {
            PutObjectRequest putRequest = PutObjectRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .contentType(file.getContentType())
                    .contentLength(file.getSize())
                    .metadata(Map.of(
                            "original-filename", sanitize(originalName),
                            "uploaded-at", Instant.now().toString()
                    ))
                    // Server-side encryption
                    .serverSideEncryption(ServerSideEncryption.AES256)
                    .build();

            s3Client.putObject(putRequest,
                    RequestBody.fromInputStream(file.getInputStream(), file.getSize()));

            log.info("Uploaded to S3: bucket={}, key={}, size={}", bucketName, key, file.getSize());

            return StoredFile.builder()
                    .id(UUID.randomUUID().toString())
                    .originalName(originalName)
                    .storedName(key)
                    .size(file.getSize())
                    .contentType(file.getContentType())
                    .accessUrl(generatePresignedUrl(key))
                    .build();

        } catch (IOException e) {
            throw new StorageException("Failed to upload file to S3", e);
        } catch (S3Exception e) {
            log.error("S3 error: code={}, message={}", e.awsErrorDetails().errorCode(), e.getMessage());
            throw new StorageException("S3 upload failed: " + e.awsErrorDetails().errorMessage(), e);
        }
    }

    public String generatePresignedUrl(String key) {
        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(presignedUrlExpiryMinutes))
                .getObjectRequest(GetObjectRequest.builder()
                        .bucket(bucketName)
                        .key(key)
                        .build())
                .build();

        PresignedGetObjectRequest presignedRequest = s3Presigner.presignGetObject(presignRequest);
        return presignedRequest.url().toString();
    }

    public String generatePresignedUploadUrl(String key, String contentType) {
        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(15))
                .putObjectRequest(PutObjectRequest.builder()
                        .bucket(bucketName)
                        .key(key)
                        .contentType(contentType)
                        .build())
                .build();

        PresignedPutObjectRequest presignedRequest = s3Presigner.presignPutObject(presignRequest);
        return presignedRequest.url().toString();
    }

    @Override
    public Resource loadAsResource(String key) {
        try {
            GetObjectRequest getRequest = GetObjectRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .build();

            ResponseInputStream<GetObjectResponse> s3Object = s3Client.getObject(getRequest);
            return new InputStreamResource(s3Object);
        } catch (NoSuchKeyException e) {
            throw new FileNotFoundException("File not found in S3: " + key);
        }
    }

    @Override
    public void delete(String key) {
        try {
            s3Client.deleteObject(DeleteObjectRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .build());
            log.info("Deleted from S3: bucket={}, key={}", bucketName, key);
        } catch (S3Exception e) {
            throw new StorageException("Failed to delete from S3: " + key, e);
        }
    }

    private String buildS3Key(String extension) {
        LocalDate today = LocalDate.now();
        return String.format("uploads/%d/%02d/%02d/%s%s",
                today.getYear(), today.getMonthValue(), today.getDayOfMonth(),
                UUID.randomUUID(),
                extension.isEmpty() ? "" : "." + extension);
    }

    private String extractExtension(String filename) {
        if (filename == null) return "";
        int dot = filename.lastIndexOf('.');
        return dot > 0 ? filename.substring(dot + 1).toLowerCase() : "";
    }

    private String sanitize(String input) {
        return input == null ? "unknown" : input.replaceAll("[^a-zA-Z0-9._\\-]", "_");
    }
}
```

### 3.4 LocalStack for Local Development

```yaml
# docker-compose.yml
version: '3.8'
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      SERVICES: s3
      AWS_DEFAULT_REGION: us-east-1
    volumes:
      - ./localstack-init:/docker-entrypoint-initaws.d

# localstack-init/create-bucket.sh
awslocal s3 mb s3://my-app-bucket
awslocal s3api put-bucket-acl --bucket my-app-bucket --acl public-read
```

```yaml
# application-local.yml
aws:
  region: us-east-1
  s3:
    endpoint: http://localhost:4566
    bucket: my-app-bucket
  access-key-id: test
  secret-access-key: test
```

### 3.5 Multipart Upload for Large Files (S3)

```java
public String uploadLargeFile(InputStream inputStream, String key,
                               String contentType, long fileSize) {
    // Use multipart upload for files > 5MB
    int partSize = 5 * 1024 * 1024; // 5 MB per part

    CreateMultipartUploadRequest createRequest = CreateMultipartUploadRequest.builder()
            .bucket(bucketName)
            .key(key)
            .contentType(contentType)
            .serverSideEncryption(ServerSideEncryption.AES256)
            .build();

    String uploadId = s3Client.createMultipartUpload(createRequest).uploadId();
    List<CompletedPart> completedParts = new ArrayList<>();
    int partNumber = 1;

    try {
        byte[] buffer = new byte[partSize];
        int bytesRead;

        while ((bytesRead = inputStream.read(buffer)) != -1) {
            byte[] partData = bytesRead < partSize ?
                    Arrays.copyOf(buffer, bytesRead) : buffer;

            UploadPartRequest uploadPartRequest = UploadPartRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .uploadId(uploadId)
                    .partNumber(partNumber)
                    .contentLength((long) bytesRead)
                    .build();

            UploadPartResponse partResponse = s3Client.uploadPart(
                    uploadPartRequest, RequestBody.fromBytes(partData));

            completedParts.add(CompletedPart.builder()
                    .partNumber(partNumber)
                    .eTag(partResponse.eTag())
                    .build());

            partNumber++;
            log.debug("Uploaded part {} for key={}", partNumber - 1, key);
        }

        // Complete the upload
        s3Client.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
                .bucket(bucketName)
                .key(key)
                .uploadId(uploadId)
                .multipartUpload(CompletedMultipartUpload.builder()
                        .parts(completedParts)
                        .build())
                .build());

        return key;

    } catch (Exception e) {
        // Abort on failure to avoid orphaned parts (you pay for them!)
        s3Client.abortMultipartUpload(AbortMultipartUploadRequest.builder()
                .bucket(bucketName)
                .key(key)
                .uploadId(uploadId)
                .build());
        throw new StorageException("Multipart upload failed", e);
    }
}
```

---

## 4. File Download & Streaming

### 4.1 Basic File Download

```java
@GetMapping("/download/{fileId}")
public ResponseEntity<Resource> downloadFile(@PathVariable String fileId) {
    FileMetadata metadata = fileMetadataService.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException("File not found: " + fileId));

    // Access control check
    checkAccessPermission(metadata);

    Resource resource = fileStorageService.loadAsResource(metadata.getStoredPath());

    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION,
                    ContentDisposition.attachment()
                            .filename(metadata.getOriginalName(), StandardCharsets.UTF_8)
                            .build()
                            .toString())
            .contentType(MediaType.parseMediaType(metadata.getContentType()))
            .contentLength(metadata.getSize())
            .body(resource);
}
```

### 4.2 StreamingResponseBody for Large Files

> [!IMPORTANT]
> `InputStreamResource` loads content into memory when serialized. For large files, use `StreamingResponseBody` which streams directly to the servlet output stream without buffering in the Spring dispatcher thread.

```java
@GetMapping("/stream/{fileId}")
public ResponseEntity<StreamingResponseBody> streamFile(@PathVariable String fileId) {
    FileMetadata metadata = fileMetadataService.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException("File not found: " + fileId));

    StreamingResponseBody responseBody = outputStream -> {
        try (InputStream inputStream = fileStorageService.openStream(metadata.getStoredPath())) {
            byte[] buffer = new byte[8192]; // 8KB buffer
            int bytesRead;
            while ((bytesRead = inputStream.read(buffer)) != -1) {
                outputStream.write(buffer, 0, bytesRead);
                outputStream.flush(); // Progressive streaming
            }
        }
    };

    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION,
                    "attachment; filename=\"" + metadata.getOriginalName() + "\"")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .contentLength(metadata.getSize())
            .body(responseBody);
}
```

> [!TIP]
> Configure Spring MVC's async support when using `StreamingResponseBody` to prevent thread pool exhaustion:
> ```yaml
> spring:
>   mvc:
>     async:
>       request-timeout: 300000  # 5 minutes for large file downloads
> ```

### 4.3 Range Requests for Video Streaming

Range requests allow clients to request specific byte ranges — essential for video players seeking to arbitrary positions.

```java
@GetMapping("/video/{fileId}")
public ResponseEntity<StreamingResponseBody> streamVideo(
        @PathVariable String fileId,
        @RequestHeader(value = "Range", required = false) String rangeHeader) {

    FileMetadata metadata = fileMetadataService.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException("Not found: " + fileId));

    long fileSize = metadata.getSize();

    if (rangeHeader == null) {
        // Full file response
        StreamingResponseBody body = out -> {
            try (InputStream is = fileStorageService.openStream(metadata.getStoredPath())) {
                is.transferTo(out);
            }
        };
        return ResponseEntity.ok()
                .header(HttpHeaders.ACCEPT_RANGES, "bytes")
                .contentType(MediaType.parseMediaType(metadata.getContentType()))
                .contentLength(fileSize)
                .body(body);
    }

    // Parse Range header: "bytes=start-end"
    long[] range = parseRange(rangeHeader, fileSize);
    long start = range[0];
    long end = range[1];
    long contentLength = end - start + 1;

    StreamingResponseBody body = out -> {
        try (InputStream is = fileStorageService.openStream(metadata.getStoredPath())) {
            is.skip(start);
            byte[] buffer = new byte[8192];
            long remaining = contentLength;
            int bytesRead;
            while (remaining > 0 &&
                   (bytesRead = is.read(buffer, 0, (int) Math.min(buffer.length, remaining))) != -1) {
                out.write(buffer, 0, bytesRead);
                remaining -= bytesRead;
            }
        }
    };

    return ResponseEntity.status(HttpStatus.PARTIAL_CONTENT)
            .header(HttpHeaders.CONTENT_RANGE,
                    "bytes " + start + "-" + end + "/" + fileSize)
            .header(HttpHeaders.ACCEPT_RANGES, "bytes")
            .contentType(MediaType.parseMediaType(metadata.getContentType()))
            .contentLength(contentLength)
            .body(body);
}

private long[] parseRange(String rangeHeader, long fileSize) {
    // Format: "bytes=1024-2047"
    String range = rangeHeader.replace("bytes=", "");
    String[] parts = range.split("-");
    long start = Long.parseLong(parts[0]);
    long end = parts.length > 1 && !parts[1].isEmpty()
            ? Long.parseLong(parts[1])
            : fileSize - 1;

    if (start >= fileSize || end >= fileSize || start > end) {
        throw new InvalidRangeException("Invalid range: " + rangeHeader);
    }
    return new long[]{start, end};
}
```

### 4.4 Content-Disposition Header

```java
// Force browser download (attachment)
.header(HttpHeaders.CONTENT_DISPOSITION,
    ContentDisposition.attachment()
        .filename("report.pdf", StandardCharsets.UTF_8)  // RFC 5987 encoding
        .build().toString())

// Display inline (e.g., image or PDF in browser)
.header(HttpHeaders.CONTENT_DISPOSITION,
    ContentDisposition.inline()
        .filename("photo.jpg", StandardCharsets.UTF_8)
        .build().toString())
```

---

## 5. Async File Processing

### 5.1 Processing Large Files Asynchronously

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class AsyncFileProcessingService {

    private final FileJobRepository fileJobRepository;
    private final FileStorageService storageService;

    @Async("fileProcessingExecutor")
    public CompletableFuture<Void> processCsvFile(String jobId, String storedPath) {
        FileJob job = fileJobRepository.findById(jobId)
                .orElseThrow(() -> new IllegalStateException("Job not found: " + jobId));

        try {
            job.setStatus(JobStatus.PROCESSING);
            job.setStartedAt(Instant.now());
            fileJobRepository.save(job);

            // Process in chunks to avoid OOM
            try (BufferedReader reader = new BufferedReader(
                    new InputStreamReader(storageService.openStream(storedPath)))) {

                String line;
                long processed = 0;
                long total = countLines(storedPath);

                reader.readLine(); // skip header
                while ((line = reader.readLine()) != null) {
                    processCsvLine(line);
                    processed++;

                    // Update progress every 100 rows
                    if (processed % 100 == 0) {
                        int progress = (int) ((processed * 100) / total);
                        job.setProgress(progress);
                        fileJobRepository.save(job);
                    }
                }
            }

            job.setStatus(JobStatus.COMPLETED);
            job.setProgress(100);
            job.setCompletedAt(Instant.now());
            fileJobRepository.save(job);

            log.info("File processing completed: jobId={}", jobId);
            return CompletableFuture.completedFuture(null);

        } catch (Exception e) {
            log.error("File processing failed: jobId={}", jobId, e);
            job.setStatus(JobStatus.FAILED);
            job.setErrorMessage(e.getMessage());
            fileJobRepository.save(job);
            return CompletableFuture.failedFuture(e);
        }
    }

    private void processCsvLine(String line) {
        // Parse and persist business data
    }

    private long countLines(String path) throws IOException {
        try (Stream<String> lines = Files.lines(Path.of(path))) {
            return lines.count();
        }
    }
}
```

### 5.2 Async Executor Configuration

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Bean(name = "fileProcessingExecutor")
    public Executor fileProcessingExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);           // 2 threads always alive
        executor.setMaxPoolSize(5);            // up to 5 for peak load
        executor.setQueueCapacity(10);         // queue 10 jobs before rejecting
        executor.setThreadNamePrefix("file-proc-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            log.error("Uncaught async exception in {}: {}", method.getName(), ex.getMessage(), ex);
    }
}
```

### 5.3 Progress Tracking Endpoint

```java
@RestController
@RequestMapping("/api/file-jobs")
@RequiredArgsConstructor
public class FileJobController {

    private final AsyncFileProcessingService processingService;
    private final FileJobRepository fileJobRepository;

    @PostMapping("/upload-csv")
    public ResponseEntity<FileJobResponse> uploadCsv(
            @RequestParam("file") MultipartFile file) {

        // Store the file
        StoredFile stored = fileStorageService.store(file);

        // Create job record
        FileJob job = FileJob.builder()
                .id(UUID.randomUUID().toString())
                .status(JobStatus.QUEUED)
                .storedPath(stored.getRelativePath())
                .originalFilename(file.getOriginalFilename())
                .createdAt(Instant.now())
                .progress(0)
                .build();
        fileJobRepository.save(job);

        // Trigger async processing
        processingService.processCsvFile(job.getId(), stored.getRelativePath());

        return ResponseEntity.accepted()
                .body(FileJobResponse.from(job));
    }

    @GetMapping("/{jobId}/status")
    public ResponseEntity<FileJobResponse> getStatus(@PathVariable String jobId) {
        FileJob job = fileJobRepository.findById(jobId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        return ResponseEntity.ok(FileJobResponse.from(job));
    }
}
```

---

## 6. File Metadata Management

### 6.1 File Entity Pattern

```java
@Entity
@Table(name = "file_metadata")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class FileMetadata {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String originalName;

    @Column(nullable = false)
    private String storedName;

    @Column(nullable = false)
    private String storedPath;          // relative path or S3 key

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private StorageType storageType;    // LOCAL, S3, GCS

    @Column(nullable = false)
    private String contentType;

    @Column(nullable = false)
    private Long size;                  // bytes

    private String checksum;            // SHA-256 for integrity

    @Column(nullable = false)
    private String uploadedBy;          // user ID

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "entity_id")
    private DomainEntity ownerEntity;   // e.g., a Product or User

    @Column(nullable = false, updatable = false)
    private Instant uploadedAt;

    private Instant deletedAt;          // Soft delete

    @Column(nullable = false)
    private boolean publiclyAccessible = false;
}
```

### 6.2 File Service — Linking to Domain Objects

```java
@Service
@Transactional
@RequiredArgsConstructor
public class FileMetadataService {

    private final FileMetadataRepository fileMetadataRepository;
    private final FileStorageService storageService;

    public FileMetadata saveMetadata(MultipartFile file, StoredFile stored, String uploadedBy) {
        return fileMetadataRepository.save(FileMetadata.builder()
                .originalName(file.getOriginalFilename())
                .storedName(stored.getStoredName())
                .storedPath(stored.getRelativePath())
                .storageType(StorageType.S3)
                .contentType(file.getContentType())
                .size(file.getSize())
                .checksum(computeChecksum(file))
                .uploadedBy(uploadedBy)
                .uploadedAt(Instant.now())
                .build());
    }

    public void softDelete(String fileId, String requestedBy) {
        FileMetadata metadata = findById(fileId);
        // Optionally check ownership
        metadata.setDeletedAt(Instant.now());
        fileMetadataRepository.save(metadata);
        // Optionally schedule physical deletion via a cleanup job
    }

    private String computeChecksum(MultipartFile file) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(file.getBytes());
            return HexFormat.of().formatHex(hash);
        } catch (Exception e) {
            return null;
        }
    }
}
```

---

## 7. Image Processing

### 7.1 Thumbnailator Library

```xml
<dependency>
    <groupId>net.coobird</groupId>
    <artifactId>thumbnailator</artifactId>
    <version>0.4.20</version>
</dependency>
```

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ImageProcessingService {

    private final FileStorageService storageService;

    public StoredFile generateThumbnail(MultipartFile originalFile, int width, int height) {
        try {
            // Read original image
            BufferedImage original = ImageIO.read(originalFile.getInputStream());
            if (original == null) {
                throw new ImageProcessingException("Cannot read image from file");
            }

            // Generate thumbnail in memory
            ByteArrayOutputStream thumbnailOut = new ByteArrayOutputStream();
            Thumbnails.of(originalFile.getInputStream())
                    .size(width, height)
                    .keepAspectRatio(true)
                    .outputFormat("jpg")
                    .outputQuality(0.85)
                    .toOutputStream(thumbnailOut);

            // Wrap thumbnail bytes as a MultipartFile
            MockMultipartFile thumbnailFile = new MockMultipartFile(
                    "thumbnail",
                    "thumb_" + originalFile.getOriginalFilename(),
                    "image/jpeg",
                    thumbnailOut.toByteArray()
            );

            return storageService.store(thumbnailFile);

        } catch (IOException e) {
            throw new ImageProcessingException("Thumbnail generation failed", e);
        }
    }

    public StoredFile convertToWebP(MultipartFile file) {
        // Apache Batik or ImageMagick via Process for WebP conversion
        // Thumbnailator doesn't support WebP natively — use ImageIO with WebP plugin
        try {
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            Thumbnails.of(file.getInputStream())
                    .scale(1.0)
                    .outputFormat("webp") // requires imageio-webp plugin
                    .toOutputStream(out);

            MockMultipartFile webpFile = new MockMultipartFile(
                    "file",
                    changeExtension(file.getOriginalFilename(), "webp"),
                    "image/webp",
                    out.toByteArray()
            );
            return storageService.store(webpFile);
        } catch (IOException e) {
            throw new ImageProcessingException("WebP conversion failed", e);
        }
    }

    private String changeExtension(String filename, String newExt) {
        int dot = filename.lastIndexOf('.');
        return (dot > 0 ? filename.substring(0, dot) : filename) + "." + newExt;
    }
}
```

---

## 8. Temporary Files

### 8.1 Creating Temp Files Safely

```java
@Service
@Slf4j
public class TempFileService {

    private final Path tempDir;

    public TempFileService(@Value("${app.temp.dir:#{null}}") String configuredDir) throws IOException {
        this.tempDir = configuredDir != null
                ? Files.createDirectories(Paths.get(configuredDir))
                : Files.createTempDirectory("app-uploads-");
        log.info("Temp directory: {}", this.tempDir);
    }

    /**
     * Create a temp file and return its path. Caller is responsible for deletion.
     */
    public Path createTempFile(String prefix, String suffix) throws IOException {
        return Files.createTempFile(tempDir, prefix + "-", "." + suffix);
    }

    /**
     * Write MultipartFile to a temp file for processing.
     */
    public Path writeToTempFile(MultipartFile file) throws IOException {
        String extension = extractExtension(file.getOriginalFilename());
        Path tempFile = createTempFile("upload", extension);
        file.transferTo(tempFile);
        log.debug("Wrote to temp file: {}", tempFile);
        return tempFile;
    }

    /**
     * Delete temp file safely (no exception if missing).
     */
    public void deleteQuietly(Path path) {
        if (path != null) {
            try {
                Files.deleteIfExists(path);
                log.debug("Deleted temp file: {}", path);
            } catch (IOException e) {
                log.warn("Failed to delete temp file: {}", path, e);
            }
        }
    }

    private String extractExtension(String filename) {
        if (filename == null) return "tmp";
        int dot = filename.lastIndexOf('.');
        return dot > 0 ? filename.substring(dot + 1) : "tmp";
    }
}
```

### 8.2 Try-With-Resources Pattern for Temp Files

```java
@Service
@RequiredArgsConstructor
public class FileConversionService {

    private final TempFileService tempFileService;

    public byte[] convertToPdf(MultipartFile docxFile) {
        Path tempDocx = null;
        Path tempPdf = null;
        try {
            tempDocx = tempFileService.writeToTempFile(docxFile);
            tempPdf = tempFileService.createTempFile("converted", "pdf");

            // Perform conversion (LibreOffice headless, Apache POI, etc.)
            performConversion(tempDocx, tempPdf);

            return Files.readAllBytes(tempPdf);

        } catch (IOException e) {
            throw new ConversionException("PDF conversion failed", e);
        } finally {
            tempFileService.deleteQuietly(tempDocx);
            tempFileService.deleteQuietly(tempPdf);
        }
    }
}
```

### 8.3 Scheduled Cleanup of Orphaned Temp Files

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class TempFileCleanupJob {

    @Value("${app.temp.cleanup.age-hours:24}")
    private int maxAgeHours;

    @Scheduled(cron = "0 0 2 * * *") // daily at 2 AM
    public void cleanOldTempFiles() {
        Path tempDir = Paths.get(System.getProperty("java.io.tmpdir"), "app-uploads");
        if (!Files.exists(tempDir)) return;

        Instant cutoff = Instant.now().minus(maxAgeHours, ChronoUnit.HOURS);

        try (Stream<Path> files = Files.walk(tempDir)) {
            files.filter(Files::isRegularFile)
                 .filter(p -> {
                     try {
                         return Files.getLastModifiedTime(p).toInstant().isBefore(cutoff);
                     } catch (IOException e) {
                         return false;
                     }
                 })
                 .forEach(p -> {
                     try {
                         Files.delete(p);
                         log.info("Cleaned up old temp file: {}", p);
                     } catch (IOException e) {
                         log.warn("Failed to delete temp file: {}", p);
                     }
                 });
        } catch (IOException e) {
            log.error("Error during temp file cleanup", e);
        }
    }
}
```

---

## 9. Security Considerations

### 9.1 File Type Validation — MIME vs Extension

```java
@Component
public class FileTypeValidator {

    private static final Map<String, Set<byte[]>> MAGIC_BYTES = Map.of(
        "image/jpeg", Set.of(new byte[]{(byte)0xFF, (byte)0xD8, (byte)0xFF}),
        "image/png",  Set.of(new byte[]{(byte)0x89, 0x50, 0x4E, 0x47}),
        "application/pdf", Set.of(new byte[]{0x25, 0x50, 0x44, 0x46})  // %PDF
    );

    public String detectMimeFromMagicBytes(InputStream inputStream) throws IOException {
        // Read first 8 bytes for magic number detection
        byte[] header = inputStream.readNBytes(8);

        for (Map.Entry<String, Set<byte[]>> entry : MAGIC_BYTES.entrySet()) {
            for (byte[] magic : entry.getValue()) {
                if (startsWith(header, magic)) {
                    return entry.getKey();
                }
            }
        }
        return "application/octet-stream";
    }

    private boolean startsWith(byte[] data, byte[] prefix) {
        if (data.length < prefix.length) return false;
        for (int i = 0; i < prefix.length; i++) {
            if (data[i] != prefix[i]) return false;
        }
        return true;
    }
}
```

> [!WARNING]
> **Never rely solely on `Content-Type` header or file extension** for security-critical decisions. A `.jpg` file can contain PHP or Java bytecode. Always inspect the actual file bytes (magic numbers) or use Apache Tika for robust MIME detection.

### 9.2 Complete Security-Hardened Upload Endpoint

```java
@PostMapping(value = "/secure-upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
@PreAuthorize("hasRole('USER')")
public ResponseEntity<FileUploadResponse> secureUpload(
        @RequestParam("file") MultipartFile file,
        Authentication authentication) {

    String uploadedBy = authentication.getName();

    // 1. Empty check
    if (file.isEmpty()) throw new FileValidationException("Empty file");

    // 2. Size check (defense in depth — Spring config also limits this)
    if (file.getSize() > 10 * 1024 * 1024)
        throw new FileValidationException("File too large");

    // 3. Filename sanitization — get safe extension only
    String originalName = file.getOriginalFilename();
    String sanitizedName = sanitizeFilename(originalName);
    String extension = extractExtension(sanitizedName);

    // 4. Extension allowlist
    if (!ALLOWED_EXTENSIONS.contains(extension.toLowerCase()))
        throw new FileValidationException("Extension not allowed: " + extension);

    // 5. MIME via Apache Tika (reads magic bytes from actual content)
    String detectedMime = tikaDetect(file);
    if (!ALLOWED_MIME_TYPES.contains(detectedMime))
        throw new FileValidationException("Content type not allowed: " + detectedMime);

    // 6. Virus scan (ClamAV or similar)
    if (!virusScanService.isSafe(file))
        throw new FileValidationException("File failed virus scan");

    // 7. Store OUTSIDE web root with UUID name (never serve from classpath)
    StoredFile stored = fileStorageService.store(file);

    // 8. Record metadata with uploader identity
    FileMetadata metadata = fileMetadataService.saveMetadata(file, stored, uploadedBy);

    return ResponseEntity.status(HttpStatus.CREATED)
            .body(FileUploadResponse.from(metadata));
}
```

### 9.3 Virus Scanning with ClamAV

```java
@Service
@Slf4j
public class ClamAVScanService implements VirusScanService {

    @Value("${clamav.host:localhost}")
    private String host;

    @Value("${clamav.port:3310}")
    private int port;

    @Override
    public boolean isSafe(MultipartFile file) {
        try (Socket socket = new Socket(host, port);
             OutputStream out = socket.getOutputStream();
             InputStream in = socket.getInputStream()) {

            // ClamAV INSTREAM command
            out.write("zINSTREAM\0".getBytes());

            // Send data in chunks
            byte[] fileBytes = file.getBytes();
            byte[] lengthBytes = ByteBuffer.allocate(4)
                    .putInt(fileBytes.length).array();
            out.write(lengthBytes);
            out.write(fileBytes);

            // End-of-stream marker
            out.write(new byte[]{0, 0, 0, 0});
            out.flush();

            // Read response
            byte[] responseBytes = in.readAllBytes();
            String response = new String(responseBytes).trim();

            log.info("ClamAV scan result for {}: {}", file.getOriginalFilename(), response);
            return response.contains("OK") && !response.contains("FOUND");

        } catch (IOException e) {
            log.error("ClamAV scan failed for {}", file.getOriginalFilename(), e);
            // Fail-closed: treat scan failure as unsafe
            throw new VirusScanException("Virus scan service unavailable", e);
        }
    }
}
```

### 9.4 Access Control for Downloads

```java
@GetMapping("/download/{fileId}")
@PreAuthorize("isAuthenticated()")
public ResponseEntity<Resource> downloadFile(
        @PathVariable String fileId,
        Authentication authentication) {

    FileMetadata metadata = fileMetadataService.findById(fileId)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

    // Authorization: owner OR admin OR public file
    boolean isOwner = metadata.getUploadedBy().equals(authentication.getName());
    boolean isAdmin = authentication.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
    boolean isPublic = metadata.isPubliclyAccessible();

    if (!isOwner && !isAdmin && !isPublic) {
        throw new ResponseStatusException(HttpStatus.FORBIDDEN,
                "Access denied to file: " + fileId);
    }

    Resource resource = fileStorageService.loadAsResource(metadata.getStoredPath());

    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION,
                    ContentDisposition.attachment()
                            .filename(metadata.getOriginalName(), StandardCharsets.UTF_8)
                            .build().toString())
            .contentType(MediaType.parseMediaType(metadata.getContentType()))
            .contentLength(metadata.getSize())
            // Prevent intermediate caches from storing private files
            .cacheControl(CacheControl.noStore())
            .body(resource);
}
```

---

## 10. Production Patterns

### 10.1 CDN Integration

```
Client ──▶ CDN (CloudFront/Fastly) ──▶ S3 Origin (private)
                │
                └──▶ (cache hit) → direct response
```

```java
@Service
public class CdnUrlService {

    @Value("${cdn.base-url}")
    private String cdnBaseUrl;

    @Value("${cdn.enabled:false}")
    private boolean cdnEnabled;

    /**
     * Transforms an S3 key into a CDN URL.
     * CDN is configured to serve from S3 as origin.
     */
    public String buildUrl(String s3Key) {
        if (cdnEnabled) {
            return cdnBaseUrl + "/" + s3Key;
        }
        // Fallback: direct S3 URL or application endpoint
        return "/api/files/download/" + s3Key;
    }

    /**
     * For private files: sign CDN URLs with CloudFront signed URLs
     * instead of S3 pre-signed URLs (CDN can cache signed responses)
     */
    public String buildSignedCdnUrl(String s3Key, Duration expiry) {
        // CloudFront URL signing logic
        // Requires private key and key pair ID from CloudFront
        Instant expiration = Instant.now().plus(expiry);
        return CloudFrontUrlSigner.getSignedURLWithCannedPolicy(
                cdnBaseUrl + "/" + s3Key,
                keyPairId,
                privateKey,
                Date.from(expiration));
    }
}
```

### 10.2 File URL vs Database Blob Storage

| Approach | Pros | Cons | Use Case |
|---|---|---|---|
| External URL (S3/CDN) | Scales infinitely, cheap, CDN-ready | More complex, requires external service | **Recommended for most apps** |
| DB Blob (`bytea`/`BLOB`) | Transactions, simple backup | Bloats DB, slow queries, no CDN | Small files, strict consistency |
| Filesystem path | Simple, fast local reads | Doesn't scale across instances | Single-server apps |

> [!IMPORTANT]
> **Never store files in your database BLOB unless they are very small (< 100KB) and require strict transactional consistency with the owning record.** For everything else, use object storage (S3/GCS/Azure Blob) with a metadata record in the database.

### 10.3 Chunked Upload for Resumable Uploads

```java
@RestController
@RequestMapping("/api/chunked-upload")
@RequiredArgsConstructor
@Slf4j
public class ChunkedUploadController {

    private final ChunkedUploadService uploadService;

    /**
     * Initialize a chunked upload session.
     */
    @PostMapping("/init")
    public ResponseEntity<ChunkedUploadSession> initUpload(
            @RequestBody InitUploadRequest request) {

        ChunkedUploadSession session = uploadService.initSession(
                request.getFilename(),
                request.getTotalSize(),
                request.getTotalChunks(),
                request.getContentType());

        return ResponseEntity.ok(session);
    }

    /**
     * Upload a single chunk.
     * Client sends chunks sequentially or can parallelize and send any order.
     */
    @PostMapping("/{sessionId}/chunk/{chunkIndex}")
    public ResponseEntity<ChunkAckResponse> uploadChunk(
            @PathVariable String sessionId,
            @PathVariable int chunkIndex,
            @RequestParam("chunk") MultipartFile chunk) {

        uploadService.storeChunk(sessionId, chunkIndex, chunk);
        boolean complete = uploadService.isComplete(sessionId);

        return ResponseEntity.ok(ChunkAckResponse.builder()
                .sessionId(sessionId)
                .chunkIndex(chunkIndex)
                .complete(complete)
                .build());
    }

    /**
     * Assemble all chunks into the final file.
     */
    @PostMapping("/{sessionId}/complete")
    public ResponseEntity<FileUploadResponse> completeUpload(
            @PathVariable String sessionId) {

        StoredFile finalFile = uploadService.assembleChunks(sessionId);
        return ResponseEntity.ok(FileUploadResponse.from(finalFile));
    }
}
```

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class ChunkedUploadService {

    private final Path chunksDir;
    private final ChunkedUploadSessionRepository sessionRepository;
    private final FileStorageService storageService;

    public ChunkedUploadSession initSession(String filename, long totalSize,
                                             int totalChunks, String contentType) {
        String sessionId = UUID.randomUUID().toString();
        Path sessionDir = chunksDir.resolve(sessionId);
        try {
            Files.createDirectories(sessionDir);
        } catch (IOException e) {
            throw new StorageException("Cannot create chunk directory", e);
        }

        ChunkedUploadSession session = ChunkedUploadSession.builder()
                .id(sessionId)
                .filename(filename)
                .contentType(contentType)
                .totalSize(totalSize)
                .totalChunks(totalChunks)
                .uploadedChunks(0)
                .createdAt(Instant.now())
                .build();

        return sessionRepository.save(session);
    }

    public void storeChunk(String sessionId, int chunkIndex, MultipartFile chunk) {
        ChunkedUploadSession session = sessionRepository.findById(sessionId)
                .orElseThrow(() -> new IllegalStateException("Session not found"));

        Path chunkPath = chunksDir.resolve(sessionId).resolve(chunkIndex + ".part");
        try {
            chunk.transferTo(chunkPath);
            session.setUploadedChunks(session.getUploadedChunks() + 1);
            sessionRepository.save(session);
        } catch (IOException e) {
            throw new StorageException("Failed to store chunk " + chunkIndex, e);
        }
    }

    public boolean isComplete(String sessionId) {
        ChunkedUploadSession session = sessionRepository.findById(sessionId)
                .orElseThrow(() -> new IllegalStateException("Session not found"));
        return session.getUploadedChunks() >= session.getTotalChunks();
    }

    public StoredFile assembleChunks(String sessionId) {
        ChunkedUploadSession session = sessionRepository.findById(sessionId)
                .orElseThrow(() -> new IllegalStateException("Session not found"));

        Path sessionDir = chunksDir.resolve(sessionId);
        Path finalFile = sessionDir.resolve("complete_" + session.getFilename());

        try (OutputStream out = Files.newOutputStream(finalFile)) {
            for (int i = 0; i < session.getTotalChunks(); i++) {
                Path chunkPath = sessionDir.resolve(i + ".part");
                if (!Files.exists(chunkPath)) {
                    throw new StorageException("Missing chunk " + i + " for session " + sessionId);
                }
                Files.copy(chunkPath, out);
            }
        } catch (IOException e) {
            throw new StorageException("Failed to assemble chunks", e);
        }

        // Upload assembled file to final storage
        try {
            MockMultipartFile multipartFile = new MockMultipartFile(
                    "file",
                    session.getFilename(),
                    session.getContentType(),
                    Files.newInputStream(finalFile));

            StoredFile stored = storageService.store(multipartFile);

            // Cleanup chunk session
            FileSystemUtils.deleteRecursively(sessionDir);
            sessionRepository.deleteById(sessionId);

            return stored;
        } catch (IOException e) {
            throw new StorageException("Failed to finalize assembled file", e);
        }
    }
}
```

---

## 11. Configuration Reference

### 11.1 Complete Configuration

```yaml
# application.yml
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 50MB            # Per-file max
      max-request-size: 100MB        # Total multipart request
      file-size-threshold: 4KB       # Buffer to disk above this
      location: /tmp/spring-uploads  # Temp file location
      resolve-lazily: false

# Custom application properties
app:
  storage:
    type: s3                         # local | s3 | gcs
    local:
      path: /var/app-data/uploads
    base-url: https://api.myapp.com
  temp:
    dir: /tmp/app-processing
    cleanup:
      age-hours: 24

aws:
  region: us-east-1
  s3:
    bucket: myapp-uploads-prod
    presigned-url-expiry-minutes: 60
    endpoint: # blank for real AWS; set for LocalStack

clamav:
  host: localhost
  port: 3310
  enabled: true

cdn:
  enabled: true
  base-url: https://cdn.myapp.com
```

### 11.2 Exception Handling for File Errors

```java
@RestControllerAdvice
@Slf4j
public class FileExceptionHandler {

    /**
     * Thrown when the file exceeds spring.servlet.multipart.max-file-size
     * The actual exception is MaxUploadSizeExceededException wrapping
     * Apache Commons FileUploadBase.FileSizeLimitExceededException
     */
    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ResponseEntity<ErrorResponse> handleMaxSizeExceeded(
            MaxUploadSizeExceededException ex) {
        log.warn("File upload size exceeded: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE)
                .body(ErrorResponse.of("FILE_TOO_LARGE",
                        "File size exceeds the maximum allowed limit"));
    }

    @ExceptionHandler(FileValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidationError(
            FileValidationException ex) {
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
                .body(ErrorResponse.of("FILE_VALIDATION_ERROR", ex.getMessage()));
    }

    @ExceptionHandler(StorageException.class)
    public ResponseEntity<ErrorResponse> handleStorageError(StorageException ex) {
        log.error("Storage error", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ErrorResponse.of("STORAGE_ERROR", "File storage operation failed"));
    }

    @ExceptionHandler(FileNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(FileNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(ErrorResponse.of("FILE_NOT_FOUND", ex.getMessage()));
    }
}
```

---

## ⚠️ Interview Traps & Gotchas

### Trap 1: Max file size throws which exception?

> [!WARNING]
> When `spring.servlet.multipart.max-file-size` is exceeded, Spring throws **`MaxUploadSizeExceededException`** (which wraps `FileSizeLimitExceededException` from Commons FileUpload or `IllegalStateException` from the servlet API). This is thrown **before** your controller method is even called — so you must handle it in a `@ControllerAdvice`, not in the method itself.

```java
// ❌ WRONG — this handler will never catch the size exception inside the controller
@PostMapping("/upload")
public ResponseEntity<?> upload(@RequestParam MultipartFile file) {
    try { ... }
    catch (MaxUploadSizeExceededException e) { ... } // Never reached!
}

// ✅ CORRECT — handle globally
@ExceptionHandler(MaxUploadSizeExceededException.class)
public ResponseEntity<?> handleSizeExceeded(MaxUploadSizeExceededException ex) { ... }
```

### Trap 2: `getOriginalFilename()` is unsafe

`MultipartFile.getOriginalFilename()` returns the filename **as submitted by the client's browser**. An attacker can set it to:
- `"../../etc/shadow"` → Path traversal
- `"exploit.php"` → Code execution if served from wrong dir
- `"'OR 1=1--"` → SQL injection if stored directly

**Always sanitize** before using the filename for anything.

### Trap 3: Storing files in `classpath:/` or `resources/`

```java
// ❌ ABSOLUTELY WRONG ANTI-PATTERN
String path = "src/main/resources/uploads/" + file.getOriginalFilename();
file.transferTo(new File(path));
```

**Why wrong:**
1. `resources/` is compiled into the JAR — you can't write to it at runtime in packaged form
2. Even in exploded form, uploads go inside the application directory — a complete mess
3. Redeployment wipes the files

**Always use an external directory** configured via properties.

### Trap 4: Blocking the thread during file I/O

```java
// ❌ Blocks a Tomcat request thread for the entire upload duration
@PostMapping("/upload")
public ResponseEntity<?> upload(@RequestParam MultipartFile file) {
    byte[] bytes = file.getBytes();       // Loads ENTIRE file into heap
    processFile(bytes);                   // Long blocking operation
    return ResponseEntity.ok().build();
}
```

For large files: use `StreamingResponseBody`, `@Async`, or reactive WebFlux.

### Trap 5: Path traversal in file download

```java
// ❌ VULNERABLE — attacker sends filename = "../../etc/passwd"
@GetMapping("/download")
public ResponseEntity<Resource> download(@RequestParam String filename) {
    Path file = Paths.get("/var/uploads/").resolve(filename); // TRAVERSAL!
    return ResponseEntity.ok(new FileSystemResource(file));
}

// ✅ SAFE — always normalize and validate against root
Path file = uploadDir.resolve(filename).normalize();
if (!file.startsWith(uploadDir)) throw new SecurityException("Path traversal!");
```

### Trap 6: Multi-instance deployment and local file storage

If you have 2+ application instances behind a load balancer and use local filesystem storage:
- User uploads to Instance A → file stored locally on A
- Next request goes to Instance B → file not found!

**Solutions:**
1. **Shared network filesystem** (NFS, EFS) — all instances mount the same volume
2. **Object storage (S3/GCS)** — the correct solution for cloud deployments
3. **Sticky sessions** — band-aid, not recommended

### Trap 7: `transferTo()` can only be called once

`MultipartFile.transferTo()` consumes the underlying input stream. Calling it twice throws an `IllegalStateException`. If you need to validate AND store:

```java
// ❌ May fail if stream was consumed during validation
fileValidator.validate(file);   // reads InputStream
file.transferTo(destination);  // InputStream already consumed!

// ✅ Safe approach: validate from bytes (if file is small enough)
byte[] bytes = file.getBytes();
validateBytes(bytes);
Files.write(destination, bytes);

// ✅ Or: copy to temp file first, then validate and move
Path temp = Files.createTempFile("validate-", ".tmp");
file.transferTo(temp);
validatePath(temp);
Files.move(temp, finalDestination);
```

### Trap 8: Memory exhaustion with `file.getBytes()`

```java
// ❌ Loads entire 500MB video into heap → OutOfMemoryError
byte[] bytes = file.getBytes();

// ✅ Stream it instead
try (InputStream is = file.getInputStream()) {
    file.transferTo(destinationPath);   // or stream to S3 directly
}
```

---

## 🔍 Deep Dive — Internals

### How Spring Handles Multipart Requests

```
HTTP POST (multipart/form-data)
        │
        ▼
DispatcherServlet.checkMultipart()
        │
        ├── Calls MultipartResolver.isMultipart()
        │       [StandardServletMultipartResolver — Servlet 3.0+]
        │
        ├── Calls MultipartResolver.resolveMultipart(request)
        │       → Wraps request in StandardMultipartHttpServletRequest
        │       → Triggers lazy (or eager) parsing of parts
        │
        ▼
Parts parsed by Tomcat's MultipartStream
        │
        ├── Small parts (< file-size-threshold) → kept in memory (ByteArrayInputStream)
        └── Large parts (> file-size-threshold) → spooled to temp file on disk
```

**Key classes:**
- `StandardServletMultipartResolver` — default since Spring 3.1 (uses Servlet 3.0 API)
- `CommonsMultipartResolver` — deprecated, used Apache Commons FileUpload
- `StandardMultipartFile` — concrete implementation of `MultipartFile`
- `MultipartAutoConfiguration` — Spring Boot auto-config based on `spring.servlet.multipart.*`

### S3 Transfer Manager Internals

AWS SDK v2's `S3TransferManager` uses:
1. **Multipart upload** automatically for files > 8MB (configurable)
2. **Parallel part upload** using CRT (Common Runtime) async HTTP client
3. **Pause/Resume** capability via persistent state

```java
// Using S3TransferManager for high-throughput uploads
@Bean
public S3TransferManager transferManager(S3AsyncClient s3AsyncClient) {
    return S3TransferManager.builder()
            .s3Client(s3AsyncClient)
            .build();
}

// S3AsyncClient with CRT for maximum throughput
@Bean
public S3AsyncClient s3AsyncClient() {
    return S3AsyncClient.crtBuilder()
            .region(Region.US_EAST_1)
            .targetThroughputInGbps(10.0)
            .minimumPartSizeInBytes(8 * 1024 * 1024L)
            .build();
}
```

### StreamingResponseBody Thread Model

```
                    ┌─ Tomcat Thread ─────────────────────────────────────┐
Request arrives ──▶ │ Controller method returns StreamingResponseBody     │
                    │ (method returns quickly, no blocking)               │
                    └──────────────────────────────────────────────────── ┘
                         │
                         ▼ StreamingResponseBody.writeTo() runs on...
                    ┌─ Async Thread (from MVC async executor) ────────────┐
                    │ outputStream.write() chunks data to client           │
                    │ Tomcat thread is FREE for other requests             │
                    └─────────────────────────────────────────────────────┘
```

This is why `StreamingResponseBody` is important: it **frees the Tomcat connector thread** while streaming data, allowing the server to handle other requests during a long download.

---

## 📝 Summary & Key Takeaways

| Topic | Key Point |
|---|---|
| `MultipartFile` | `getOriginalFilename()` and `getContentType()` are client-controlled — never trust for security |
| Configuration | Set `max-file-size`, `max-request-size`, AND Tomcat's `maxPostSize` for large uploads |
| Storage | Use S3/object storage in multi-instance production. Never local filesystem in cloud deployments |
| Path traversal | Always `.normalize()` and validate `.startsWith(root)` before resolving user-supplied paths |
| File naming | UUID is the default safe choice; SHA-256 hash enables deduplication |
| Large files | Use `StreamingResponseBody` for downloads, `@Async` for processing — never `getBytes()` |
| Security | Validate: size → extension → MIME (magic bytes via Tika) → virus scan (ClamAV) |
| S3 | Use multipart upload for files > 5MB; always abort on failure to avoid orphaned parts |
| Temp files | Always clean up in `finally` blocks or use scheduled jobs for orphaned files |
| Exception | `MaxUploadSizeExceededException` must be caught in `@ControllerAdvice`, not the controller |
| Multi-instance | Local storage breaks horizontal scaling — use shared storage (S3, EFS) |

---

## ❓ Practice Questions

1. **What exception is thrown when a file exceeds `spring.servlet.multipart.max-file-size`? Where must you catch it?**

2. **Explain how a path traversal attack works in file downloads. Write a safe download method that prevents it.**

3. **A user uploads a file named `../../etc/passwd`. What does your `sanitizeFilename()` method output?**

4. **Why is `MultipartFile.getOriginalFilename()` considered unsafe? What are the two risks?**

5. **Your app has 3 instances behind a load balancer and uses local filesystem storage. What problem occurs and how do you fix it?**

6. **When does `transferTo()` throw `IllegalStateException`? How would you handle a scenario where you need to validate the file content before storing it?**

7. **Compare `InputStreamResource` and `StreamingResponseBody` for large file downloads. Why is one preferred over the other?**

8. **Explain the difference between `@RequestParam` and `@RequestPart` for multipart handling. When would you use each?**

9. **Your S3 multipart upload fails halfway through. What happens if you don't abort the upload? What is the financial implication?**

10. **Describe the chunked upload pattern. What information does the client need from the server in step 1 (init)?**

11. **Why should you never store uploaded files in `src/main/resources/` or the classpath?**

12. **A user uploads a file with `.jpg` extension but it actually contains PHP code. What layers of validation would catch this, and which ones wouldn't?**

13. **What is `file-size-threshold` in Spring's multipart config? What happens to files below vs. above this threshold?**

14. **How would you implement a download that supports HTTP Range requests for video streaming? What response status code do you return for a partial response?**

15. **Describe how you would integrate ClamAV for virus scanning in a Spring Boot file upload flow. What should happen if the ClamAV service is unavailable?**
