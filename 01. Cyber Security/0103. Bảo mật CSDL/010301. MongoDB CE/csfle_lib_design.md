# **TÀI LIỆU THIẾT KẾ: THƯ VIỆN MÃ HÓA DỮ LIỆU ĐẦU VÀO (csfle-lib)**

---

## **I. MỤC TIÊU**

Xây dựng một thư viện Java nội bộ (`csfle-lib`) đóng gói toàn bộ logic mã hóa/giải mã cấp trường (CSFLE) thành một lớp trừu tượng duy nhất, giúp các dịch vụ nghiệp vụ:

- **Ghi dữ liệu mới**: Tự động mã hóa các trường nhạy cảm trước khi lưu vào MongoDB.
- **Truy vấn dữ liệu**: Tự động mã hóa giá trị tìm kiếm trước khi gửi câu lệnh find/query.
- **Đọc dữ liệu**: Tự động giải mã các trường nhị phân BinData trả về thành dữ liệu rõ.

Lập trình viên nghiệp vụ **không cần biết** chi tiết kỹ thuật mã hóa bên dưới.

---

## **II. KIẾN TRÚC TỔNG QUAN**

```mermaid
graph TD
    subgraph "Ứng dụng Nghiệp vụ"
        A[Service Layer] --> B[csfle-lib]
    end

    subgraph "csfle-lib"
        B --> C[CsfleMongoClientFactory]
        B --> D[EncryptionSchemaRegistry]
        B --> E[KeyManager]
    end

    subgraph "Hạ tầng"
        C --> F[MongoDB Driver + AutoEncryption]
        E --> G[HashiCorp Vault]
        E --> H["MongoDB (encryption.__keyVault)"]
    end
```

### **Phân lớp bên trong thư viện:**

| Lớp | Trách nhiệm |
| :--- | :--- |
| **CsfleMongoClientFactory** | Khởi tạo và cấu hình `MongoClient` với `AutoEncryptionSettings`. Đây là điểm vào duy nhất để lấy client có khả năng mã hóa tự động. |
| **EncryptionSchemaRegistry** | Quản lý danh sách các collection và trường cần mã hóa, kèm thuật toán áp dụng (Deterministic hoặc Random). Cho phép đăng ký schema bằng code hoặc file cấu hình. |
| **KeyManager** | Quản lý vòng đời khóa: lấy Master Key từ Vault, tạo/tra cứu Data Encryption Key (DEK) trong `__keyVault`, cache khóa trên RAM. |

---

## **III. DANH SÁCH THÀNH PHẦN VÀ API CHÍNH**

### **1. CsfleMongoClientFactory**

| Phương thức | Mô tả |
| :--- | :--- |
| `MongoClient createEncryptedClient()` | Trả về `MongoClient` đã được cấu hình `AutoEncryptionSettings` với schemaMap và kmsProviders. Lập trình viên dùng client này như client MongoDB bình thường. |
| `MongoClient createPlainClient()` | Trả về `MongoClient` không mã hóa, phục vụ các tác vụ quản trị hoặc đọc dữ liệu thô (debug, migration). |

### **2. EncryptionSchemaRegistry**

| Phương thức | Mô tả |
| :--- | :--- |
| `void registerField(String collection, String field, Algorithm algo)` | Đăng ký một trường cần mã hóa. `algo` là `DETERMINISTIC` (hỗ trợ tìm kiếm chính xác) hoặc `RANDOM` (bảo mật tối đa, không tìm kiếm được). |
| `Map<String, BsonDocument> buildSchemaMap()` | Sinh ra `schemaMap` dạng BSON từ danh sách trường đã đăng ký, sẵn sàng truyền vào `AutoEncryptionSettings`. |

### **3. KeyManager**

| Phương thức | Mô tả |
| :--- | :--- |
| `byte[] getMasterKey()` | Kết nối HashiCorp Vault (qua REST API hoặc SDK) để lấy Master Key 96 bytes. Kết quả được cache trên RAM trong suốt vòng đời ứng dụng. |
| `UUID getOrCreateDEK(String keyAltName)` | Tra cứu DEK theo bí danh trong `encryption.__keyVault`. Nếu chưa tồn tại thì tự động sinh mới, bọc bằng Master Key và lưu vào `__keyVault`. |

### **4. Enum Algorithm**

| Giá trị | Ý nghĩa | Khi nào dùng |
| :--- | :--- | :--- |
| `DETERMINISTIC` | Cùng giá trị đầu vào luôn ra cùng kết quả mã hóa. | Các trường cần **tìm kiếm chính xác** (số căn cước, mã nhân viên). |
| `RANDOM` | Mỗi lần mã hóa ra kết quả khác nhau. | Các trường **không cần tìm kiếm** nhưng cần bảo mật tối đa (số điện thoại, địa chỉ, ghi chú). |

---

## **IV. LUỒNG XỬ LÝ**

### **1. Luồng khởi tạo ứng dụng (Application Startup)**

```mermaid
sequenceDiagram
    participant App as Ứng dụng Nghiệp vụ
    participant Lib as csfle-lib
    participant Vault as HashiCorp Vault
    participant DB as MongoDB (__keyVault)

    App ->> Lib: 1. Gọi CsfleMongoClientFactory.createEncryptedClient()
    Lib ->> Vault: 2. KeyManager.getMasterKey() — Lấy Master Key 96 bytes
    Vault -->> Lib: 3. Trả về Master Key
    Lib ->> DB: 4. KeyManager.getOrCreateDEK("employee-key") — Tra cứu/Sinh DEK
    DB -->> Lib: 5. Trả về UUID của DEK
    Lib ->> Lib: 6. EncryptionSchemaRegistry.buildSchemaMap() — Sinh BSON Schema
    Lib -->> App: 7. Trả về MongoClient có khả năng mã hóa tự động
```

### **2. Luồng ghi dữ liệu mới (Insert)**

```mermaid
sequenceDiagram
    participant Dev as Code Nghiệp vụ
    participant Client as MongoClient (Auto-Encrypt)
    participant DB as MongoDB Server

    Dev ->> Client: 1. collection.insertOne({ personalID: "0123456789", phone: "090..." })
    Note over Client: Driver tự động mã hóa personalID (Deterministic)<br/>và phone (Random) thành BinData Subtype 6
    Client ->> DB: 2. INSERT { personalID: BinData(...), phone: BinData(...) }
    DB -->> Client: 3. Xác nhận ghi thành công
    Client -->> Dev: 4. Phản hồi hoàn tất
```

### **3. Luồng truy vấn (Query/Find)**

```mermaid
sequenceDiagram
    participant Dev as Code Nghiệp vụ
    participant Client as MongoClient (Auto-Encrypt)
    participant DB as MongoDB Server

    Dev ->> Client: 1. collection.find({ personalID: "0123456789" })
    Note over Client: Driver tự động mã hóa giá trị tìm kiếm<br/>"0123456789" thành BinData Subtype 6 (Deterministic)
    Client ->> DB: 2. FIND { personalID: BinData(...) }
    DB -->> Client: 3. Trả về document (các trường nhạy cảm vẫn là BinData)
    Note over Client: Driver tự động giải mã BinData<br/>về Plaintext trên RAM
    Client -->> Dev: 4. Trả về { personalID: "0123456789", phone: "090..." }
```

---

## **V. CẤU HÌNH**

Thư viện đọc cấu hình từ file `csfle.yml` (hoặc biến môi trường):

```yaml
csfle:
  # Kết nối MongoDB
  mongodb:
    uri: "mongodb://admin:secret@host1:27017,host2:27017,host3:27017/?replicaSet=rs0"
    database: "companyDb"

  # Kết nối HashiCorp Vault
  vault:
    address: "https://vault.company.internal:8200"
    token-file: "/etc/app/vault.token"
    master-key-path: "secret/data/csfle/master-key"

  # Key Vault
  key-vault:
    namespace: "encryption.__keyVault"
    default-key-alt-name: "employee-secrets-key"

  # Danh sách trường mã hóa theo collection
  schemas:
    - collection: "employees"
      fields:
        - name: "personalID"
          algorithm: DETERMINISTIC
        - name: "phoneNumber"
          algorithm: RANDOM
        - name: "homeAddress"
          algorithm: RANDOM
    - collection: "customers"
      fields:
        - name: "taxCode"
          algorithm: DETERMINISTIC
        - name: "bankAccount"
          algorithm: RANDOM
```

---

## **VI. CÁCH TÍCH HỢP VÀO ỨNG DỤNG NGHIỆP VỤ**

### **1. Thêm dependency (Maven)**

```xml
<dependency>
    <groupId>com.company.security</groupId>
    <artifactId>csfle-lib</artifactId>
    <version>1.0.0</version>
</dependency>
```

### **2. Sử dụng trong code nghiệp vụ**

```java
// Khởi tạo client mã hóa (thường làm 1 lần khi ứng dụng khởi động)
MongoClient client = CsfleMongoClientFactory.createEncryptedClient();
MongoCollection<Document> employees = client.getDatabase("companyDb").getCollection("employees");

// Ghi dữ liệu — Driver tự mã hóa personalID và phoneNumber
employees.insertOne(new Document()
    .append("name", "Nguyễn Văn A")
    .append("personalID", "0123456789")   // Tự động mã hóa Deterministic
    .append("phoneNumber", "0901234567")  // Tự động mã hóa Random
);

// Truy vấn — Driver tự mã hóa giá trị tìm kiếm và giải mã kết quả
Document result = employees.find(eq("personalID", "0123456789")).first();
System.out.println(result.getString("personalID"));  // In ra: 0123456789 (đã giải mã)
```

> [!IMPORTANT]
> Lập trình viên nghiệp vụ **chỉ cần gọi `CsfleMongoClientFactory.createEncryptedClient()`** một lần duy nhất. Sau đó sử dụng `MongoClient` trả về như bình thường. Toàn bộ logic mã hóa, giải mã, quản lý khóa đều được thư viện xử lý trong suốt (transparent).

---

## **VII. CẤU TRÚC THƯ MỤC DỰ ÁN**

```
csfle-lib/
├── pom.xml
├── src/main/java/com/company/security/csfle/
│   ├── CsfleMongoClientFactory.java      // Điểm vào chính
│   ├── EncryptionSchemaRegistry.java      // Quản lý schema mã hóa
│   ├── KeyManager.java                    // Quản lý Master Key & DEK
│   ├── Algorithm.java                     // Enum DETERMINISTIC / RANDOM
│   └── config/
│       └── CsfleConfig.java               // Đọc & parse file csfle.yml
├── src/main/resources/
│   └── csfle.yml                          // File cấu hình mẫu
└── src/test/java/com/company/security/csfle/
    ├── CsfleInsertTest.java               // Test mã hóa khi ghi
    ├── CsfleQueryTest.java                // Test mã hóa khi truy vấn
    └── KeyManagerTest.java                // Test quản lý khóa
```
