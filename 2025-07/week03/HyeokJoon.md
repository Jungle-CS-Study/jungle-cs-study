# File 처리 스터디 가이드

## 목차
1. [기본 파일 I/O](#1-기본-파일-io)
2. [Spring MultipartFile](#2-spring-multipartfile)
3. [스트리밍 처리](#3-스트리밍-처리)
4. [이미지 파일 처리](#4-이미지-파일-처리)
5. [파일 업로드/다운로드](#5-파일-업로드다운로드)
6. [메모리 최적화](#6-메모리-최적화)
7. [보안 고려사항](#7-보안-고려사항)
8. [실습 예제](#8-실습-예제)

---

## 1. 기본 파일 I/O

### 왜 NIO를 사용하는가?

**기존 문제점 (Java IO):**
- **블로킹 I/O**: 파일 읽기/쓰기 시 스레드가 완전히 멈춤
- **성능 한계**: 대량의 파일 처리 시 스레드 풀 고갈
- **메모리 비효율**: 전체 파일을 메모리에 로드해야 함

**NIO의 해결책:**
- **논블로킹**: 다른 작업과 병렬 처리 가능
- **채널 기반**: 양방향 통신으로 효율성 증대
- **버퍼 활용**: 메모리 사용량 최적화

**실제 사용 시나리오:**
```
상황: 1만개의 로그 파일을 동시에 처리해야 하는 경우
- 기존 IO: 1만개 스레드 필요 → 메모리 부족
- NIO: 소수의 스레드로 모든 파일 처리 가능
```

### 1.1 Java NIO vs IO

**히스토리:**
- Java 1.0 (1996): 기본 IO 도입
- Java 1.4 (2002): NIO 도입 - "New I/O"
- Java 7 (2011): NIO.2 도입 - "Non-blocking I/O"
```java
// 전통적인 IO
FileInputStream fis = new FileInputStream("file.txt");
byte[] buffer = new byte[1024];
int bytesRead = fis.read(buffer);

// NIO (Non-blocking IO)
Path path = Paths.get("file.txt");
byte[] data = Files.readAllBytes(path);
```
---

## 2. Spring MultipartFile

### 왜 MultipartFile을 사용하는가?

**웹에서 파일 업로드의 역사:**
1. **초기 웹 (1990년대)**: 텍스트만 전송 가능
2. **HTML 4.0 (1997)**: `<input type="file">` 도입
3. **RFC 2388 (1998)**: multipart/form-data 표준화
4. **Spring 등장**: 복잡한 파싱 로직을 추상화

**기존 문제점:**
```java
// Servlet API로 직접 처리 (Spring 이전)
String contentType = request.getContentType();
if (contentType.startsWith("multipart/form-data")) {
    // 복잡한 바이너리 파싱 로직 필요
    // boundary 찾기, 헤더 파싱, 인코딩 처리...
}
```

**MultipartFile의 장점:**
- **추상화**: 복잡한 파싱 로직 숨김
- **메타데이터**: 파일명, 크기, 타입 자동 추출
- **스트림 접근**: 메모리 효율적 처리
- **검증 지원**: 크기, 타입 제한 쉽게 적용

**실제 사용 시나리오:**
```
상황: 사용자가 프로필 이미지를 업로드하는 경우
- 파일 검증 (크기, 형식)
- 임시 저장
- 이미지 리사이징
- 최종 저장
→ MultipartFile 하나로 모든 과정 처리
```

### 2.1 기본 사용법
```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) {
    String filename = file.getOriginalFilename();
    long size = file.getSize();
    String contentType = file.getContentType();
    
    return ResponseEntity.ok("업로드 완료");
}
```

### 2.2 파일 검증
```java
private void validateFile(MultipartFile file) {
    if (file.isEmpty()) {
        throw new IllegalArgumentException("파일이 비어있습니다");
    }
    
    String filename = file.getOriginalFilename();
    if (filename == null || filename.trim().isEmpty()) {
        throw new IllegalArgumentException("파일명이 없습니다");
    }
    
    String extension = getFileExtension(filename);
    if (!Arrays.asList("jpg", "jpeg", "png").contains(extension.toLowerCase())) {
        throw new IllegalArgumentException("지원하지 않는 파일 형식입니다");
    }
}

private String getFileExtension(String filename) {
    return filename.substring(filename.lastIndexOf(".") + 1);
}
```

### 2.3 파일 저장
```java
@Value("${file.upload.dir}")
private String uploadDir;

public String saveFile(MultipartFile file) throws IOException {
    String filename = UUID.randomUUID() + "_" + file.getOriginalFilename();
    Path filePath = Paths.get(uploadDir, filename);
    
    Files.copy(file.getInputStream(), filePath, StandardCopyOption.REPLACE_EXISTING);
    return filename;
}
```

---

## 3. 스트리밍 처리

### 왜 스트리밍 처리가 필요한가?

**메모리 한계의 현실:**
```
시나리오: 4GB 동영상 파일 업로드
- 전체 로드: 4GB RAM 필요 → OutOfMemoryError
- 스트리밍: 8KB 버퍼로 처리 가능
```

**스트리밍이 해결하는 문제:**
- **메모리 부족**: 큰 파일도 작은 메모리로 처리
- **응답 지연**: 전체 처리 완료를 기다리지 않음
- **동시성**: 여러 사용자가 동시에 대용량 파일 처리 가능

**실제 사용 시나리오:**
```
상황: 동영상 스트리밍 서비스
- 사용자가 재생 버튼 클릭
- 전체 파일 다운로드 대기 X
- 첫 몇 초분만 받아서 즉시 재생 시작
- 재생하면서 나머지 부분 계속 다운로드
```

### 3.1 대용량 파일 처리 전략
```java
private static final long STREAMING_THRESHOLD = 5 * 1024 * 1024; // 5MB

public void processFile(MultipartFile file, HttpServletResponse response) throws IOException {
    if (file.getSize() <= STREAMING_THRESHOLD) {
        // 메모리 처리
        processInMemory(file, response);
    } else {
        // 스트리밍 처리
        processWithStreaming(file, response);
    }
}
```

### 3.2 스트리밍 읽기/쓰기
```java
public void streamFile(InputStream input, OutputStream output) throws IOException {
    byte[] buffer = new byte[8192]; // 8KB 버퍼
    int bytesRead;
    
    while ((bytesRead = input.read(buffer)) != -1) {
        output.write(buffer, 0, bytesRead);
        output.flush(); // 즉시 전송
    }
}
```

### 3.3 진행률 추적
```java
public class ProgressTrackingInputStream extends FilterInputStream {
    private final long totalBytes;
    private long bytesRead = 0;
    private final Consumer<Double> progressCallback;
    
    public ProgressTrackingInputStream(InputStream in, long totalBytes, 
                                    Consumer<Double> progressCallback) {
        super(in);
        this.totalBytes = totalBytes;
        this.progressCallback = progressCallback;
    }
    
    @Override
    public int read(byte[] b, int off, int len) throws IOException {
        int result = super.read(b, off, len);
        if (result > 0) {
            bytesRead += result;
            double progress = (double) bytesRead / totalBytes * 100;
            progressCallback.accept(progress);
        }
        return result;
    }
}
```

---

## 4. 이미지 파일 처리

### 왜 서버에서 이미지 처리를 하는가?

**서버 처리가 필요한 이유:**
- **보안**: 클라이언트는 조작 가능, 서버는 신뢰할 수 있음
- **일관성**: 모든 디바이스에서 동일한 결과
- **성능**: 서버의 강력한 CPU 활용
- **저장 최적화**: 다양한 크기의 썸네일 미리 생성

**ImageIO의 등장 배경:**
```java
// ImageIO 이전 (AWT 시대)
Image img = Toolkit.getDefaultToolkit().createImage("image.jpg");
// 복잡한 MediaTracker 사용 필요
// 제한적인 포맷 지원

// ImageIO 이후 (Java 1.4+)
BufferedImage img = ImageIO.read(new File("image.jpg"));
// 간단하고 다양한 포맷 지원
```

**실제 사용 시나리오:**
```
상황: 인스타그램 같은 SNS 서비스
- 원본 이미지 (4K, 10MB)
- 피드용 중간 크기 (1080p, 500KB)
- 썸네일 (150x150, 20KB)
- 프로필용 원형 이미지 (100x100)
→ 업로드 시 서버에서 모든 버전 생성
```

### 4.1 이미지 읽기/쓰기
```java
// 이미지 읽기
BufferedImage image = ImageIO.read(file.getInputStream());
int width = image.getWidth();
int height = image.getHeight();

// 이미지 쓰기
ByteArrayOutputStream baos = new ByteArrayOutputStream();
ImageIO.write(processedImage, "jpg", baos);
byte[] imageBytes = baos.toByteArray();
```

### 4.2 이미지 리사이징
```java
public BufferedImage resizeImage(BufferedImage original, int targetWidth, int targetHeight) {
    BufferedImage resized = new BufferedImage(targetWidth, targetHeight, original.getType());
    Graphics2D g2d = resized.createGraphics();
    
    g2d.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BILINEAR);
    g2d.drawImage(original, 0, 0, targetWidth, targetHeight, null);
    g2d.dispose();
    
    return resized;
}
```

### 4.3 이미지 포맷 변환
```java
public byte[] convertImageFormat(BufferedImage image, String targetFormat) throws IOException {
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    
    if ("jpg".equalsIgnoreCase(targetFormat) && image.getColorModel().hasAlpha()) {
        // PNG to JPG: 투명도 제거
        BufferedImage jpgImage = new BufferedImage(image.getWidth(), image.getHeight(), BufferedImage.TYPE_INT_RGB);
        Graphics2D g2d = jpgImage.createGraphics();
        g2d.setColor(Color.WHITE);
        g2d.fillRect(0, 0, image.getWidth(), image.getHeight());
        g2d.drawImage(image, 0, 0, null);
        g2d.dispose();
        image = jpgImage;
    }
    
    ImageIO.write(image, targetFormat, baos);
    return baos.toByteArray();
}
```
---

## 5. 메모리 최적화

### 왜 메모리 최적화가 중요한가?

**메모리 부족의 현실적 문제:**
```
상황: 파일 업로드 서버
- 동시 사용자 100명
- 각자 10MB 파일 업로드
- 필요 메모리: 100 × 10MB = 1GB
- 실제 서버 메모리: 512MB
→ OutOfMemoryError 발생
```

**실제 사용 시나리오:**
```
상황: 이미지 처리 서비스
기존 방식:
- 10MB 이미지 → 메모리에 전체 로드
- 처리 중 추가 메모리 사용
- 총 30-40MB 메모리 필요

최적화 후:
- 스트리밍 처리
- 임시 파일 활용
- 8KB 버퍼만 사용
→ 1000배 메모리 절약
```

### 6.1 메모리 사용량 모니터링
```java
public class MemoryMonitor {
    public static void logMemoryUsage(String operation) {
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.printf("[%s] 사용 메모리: %d MB / 전체: %d MB%n", 
                         operation, usedMemory / 1024 / 1024, totalMemory / 1024 / 1024);
    }
}
```

### 6.2 임시 파일 활용
```java
public File createTempFile(MultipartFile file) throws IOException {
    String prefix = "upload_";
    String suffix = getFileExtension(file.getOriginalFilename());
    
    File tempFile = File.createTempFile(prefix, "." + suffix);
    tempFile.deleteOnExit(); // JVM 종료 시 자동 삭제
    
    try (FileOutputStream fos = new FileOutputStream(tempFile)) {
        fos.write(file.getBytes());
    }
    
    return tempFile;
}
```

### 6.3 리소스 관리
```java
// try-with-resources 사용
public void processFile(String filename) throws IOException {
    try (FileInputStream fis = new FileInputStream(filename);
         BufferedInputStream bis = new BufferedInputStream(fis);
         FileOutputStream fos = new FileOutputStream("output.txt");
         BufferedOutputStream bos = new BufferedOutputStream(fos)) {
        
        // 파일 처리 로직
        byte[] buffer = new byte[8192];
        int bytesRead;
        while ((bytesRead = bis.read(buffer)) != -1) {
            bos.write(buffer, 0, bytesRead);
        }
    }
}
```

---

## 7. 보안 고려사항

### 왜 파일 업로드에 보안이 중요한가?

**파일 업로드 보안 사고 사례:**
1. **2019년**: 한 쇼핑몰, 이미지 업로드로 서버 해킹
2. **2020년**: 파일명 조작으로 시스템 파일 덮어쓰기
3. **2021년**: 악성 스크립트 업로드로 XSS 공격

**파일 업로드의 보안 위험:**
```java
// 위험한 코드 예시
String filename = file.getOriginalFilename(); // "../../../etc/passwd"
File dest = new File("uploads/" + filename);
file.transferTo(dest); // 시스템 파일 덮어쓰기 가능!
```

**보안 위협의 종류:**
- **경로 순회 공격**: `../` 를 이용한 시스템 파일 접근
- **파일 타입 위조**: 악성 스크립트를 이미지로 위장
- **서비스 거부 공격**: 대용량 파일로 디스크/메모리 고갈
- **코드 실행**: 업로드된 스크립트 파일 실행

**보안 대책의 발전:**
1. **1세대**: 파일 확장자만 체크
2. **2세대**: MIME 타입 검증 추가
3. **3세대**: 매직 넘버(파일 헤더) 검증
4. **현재**: 다층 보안 + 샌드박스 실행

**실제 사용 시나리오:**
```
상황: 온라인 쇼핑몰 상품 이미지 업로드
위험 요소:
- 판매자가 악성 파일 업로드 시도
- 구매자가 해당 페이지 방문 시 스크립트 실행
- 개인정보 탈취, 계정 탈취 가능

보안 대책:
- 파일 타입 엄격 검증
- 업로드 디렉토리 실행 권한 제거
- CDN을 통한 정적 파일 서빙
→ 안전한 이미지 업로드 구현
```

### 7.1 파일명 검증
```java
public String sanitizeFilename(String filename) {
    if (filename == null) return null;
    
    // 위험한 문자 제거
    filename = filename.replaceAll("[^a-zA-Z0-9._-]", "_");
    
    // 경로 순회 공격 방지
    filename = filename.replace("..", "_");
    filename = filename.replace("/", "_");
    filename = filename.replace("\\", "_");
    
    // 길이 제한
    if (filename.length() > 255) {
        String extension = getFileExtension(filename);
        filename = filename.substring(0, 255 - extension.length() - 1) + "." + extension;
    }
    
    return filename;
}
```

### 7.2 파일 타입 검증
```java
public boolean isValidImageFile(MultipartFile file) throws IOException {
    // MIME 타입 검증
    String contentType = file.getContentType();
    if (!Arrays.asList("image/jpeg", "image/png", "image/gif").contains(contentType)) {
        return false;
    }
    
    // 매직 넘버 검증
    byte[] header = new byte[8];
    file.getInputStream().read(header);
    
    // JPEG: FF D8 FF
    if (header[0] == (byte) 0xFF && header[1] == (byte) 0xD8 && header[2] == (byte) 0xFF) {
        return true;
    }
    
    // PNG: 89 50 4E 47 0D 0A 1A 0A
    if (header[0] == (byte) 0x89 && header[1] == 0x50 && header[2] == 0x4E && header[3] == 0x47) {
        return true;
    }
    
    return false;
}
```

### 7.3 파일 크기 제한
```java
@Component
public class FileUploadConfig {
    @Value("${app.file.max-size:10485760}") // 10MB
    private long maxFileSize;
    
    public void validateFileSize(MultipartFile file) {
        if (file.getSize() > maxFileSize) {
            throw new IllegalArgumentException(
                String.format("파일 크기가 제한을 초과했습니다. 최대: %d bytes", maxFileSize)
            );
        }
    }
}
```

---

## 8. 실습 예제

### 왜 이런 기능들이 실무에서 필요한가?

**실무 프로젝트에서의 파일 처리:**
1. **전자상거래**: 상품 이미지, 리뷰 사진
2. **SNS**: 프로필 사진, 게시물 이미지
3. **문서 관리**: PDF, 오피스 파일
4. **미디어 서비스**: 동영상, 음악 파일
5. **백업 시스템**: 압축, 암호화

**각 예제의 실무 적용:**

### 8.1 이미지 썸네일 생성기

**실무 필요성:**
- **페이지 로딩 속도**: 원본 대신 썸네일로 빠른 로딩
- **대역폭 절약**: 모바일 사용자 데이터 절약
- **사용자 경험**: 빠른 미리보기 제공

**실제 사용 사례:**
- 유튜브: 동영상 썸네일
- 아마존: 상품 이미지 다양한 크기
- 인스타그램: 피드, 스토리, 프로필 각각 다른 크기
```java
@Service
public class ThumbnailService {
    
    public byte[] createThumbnail(MultipartFile file, int width, int height) throws IOException {
        BufferedImage original = ImageIO.read(file.getInputStream());
        BufferedImage thumbnail = resizeImage(original, width, height);
        
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        String format = getImageFormat(file.getOriginalFilename());
        ImageIO.write(thumbnail, format, baos);
        
        return baos.toByteArray();
    }
    
    private BufferedImage resizeImage(BufferedImage original, int width, int height) {
        Image scaled = original.getScaledInstance(width, height, Image.SCALE_SMOOTH);
        BufferedImage thumbnail = new BufferedImage(width, height, original.getType());
        
        Graphics2D g2d = thumbnail.createGraphics();
        g2d.drawImage(scaled, 0, 0, null);
        g2d.dispose();
        
        return thumbnail;
    }
}
```

### 8.2 파일 압축 서비스

**실무 필요성:**
- **저장 공간 절약**: 클라우드 스토리지 비용 절감
- **전송 속도**: 압축으로 업로드/다운로드 시간 단축
- **백업 효율성**: 대량 데이터 백업 시 필수

**실제 사용 사례:**
- Google Drive: 자동 압축으로 용량 절약
- 이메일 첨부: 대용량 파일 압축 전송
- 로그 시스템: 오래된 로그 자동 압축
```java
@Service
public class FileCompressionService {
    
    public byte[] compressFile(MultipartFile file) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        
        try (GZIPOutputStream gzipOut = new GZIPOutputStream(baos);
             InputStream inputStream = file.getInputStream()) {
            
            byte[] buffer = new byte[1024];
            int len;
            while ((len = inputStream.read(buffer)) != -1) {
                gzipOut.write(buffer, 0, len);
            }
        }
        
        return baos.toByteArray();
    }
    
    public byte[] decompressFile(byte[] compressedData) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        
        try (GZIPInputStream gzipIn = new GZIPInputStream(new ByteArrayInputStream(compressedData))) {
            byte[] buffer = new byte[1024];
            int len;
            while ((len = gzipIn.read(buffer)) != -1) {
                baos.write(buffer, 0, len);
            }
        }
        
        return baos.toByteArray();
    }
}
```

### 8.3 파일 메타데이터 추출기

**실무 필요성:**
- **검색 기능**: 파일 내용 기반 검색
- **중복 제거**: 해시값으로 동일 파일 감지
- **감사 추적**: 언제, 누가, 무엇을 업로드했는지 추적
- **자동 분류**: 파일 타입별 자동 폴더 분류

**실제 사용 사례:**
- Dropbox: 중복 파일 자동 감지
- Google Photos: 사진 메타데이터로 자동 분류
- 기업 문서 관리: 파일 버전 관리, 변경 이력 추적
```java
@Service
public class FileMetadataService {
    
    public FileMetadata extractMetadata(MultipartFile file) throws IOException {
        FileMetadata metadata = new FileMetadata();
        metadata.setOriginalName(file.getOriginalFilename());
        metadata.setSize(file.getSize());
        metadata.setContentType(file.getContentType());
        metadata.setUploadTime(LocalDateTime.now());
        
        if (isImageFile(file)) {
            BufferedImage image = ImageIO.read(file.getInputStream());
            metadata.setWidth(image.getWidth());
            metadata.setHeight(image.getHeight());
        }
        
        // 파일 해시 계산
        String hash = calculateMD5(file.getBytes());
        metadata.setHash(hash);
        
        return metadata;
    }
    
    private String calculateMD5(byte[] data) throws IOException {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] hash = md.digest(data);
            return DatatypeConverter.printHexBinary(hash).toLowerCase();
        } catch (NoSuchAlgorithmException e) {
            throw new IOException("MD5 알고리즘을 찾을 수 없습니다", e);
        }
    }
}
```