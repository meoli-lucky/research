# **TÀI LIỆU KIẾN TRÚC: KỊCH BẢN DI CƯ & KÍCH HOẠT BẢO MẬT DỮ LIỆU ĐA TẦNG**

Tài liệu này trình bày chi tiết hai phương án triển khai bảo mật: **Phương án 1 (Data-at-Rest + CSFLE)** và **Phương án 2 (Chỉ CSFLE)** cho cụm Percona MongoDB CE đang chứa **60GB** dữ liệu rõ (Plaintext) tại môi trường Production. 

Quy trình được thiết kế theo nguyên tắc **Downtime Ứng dụng có kiểm soát (App Downtime)** để đảm bảo an toàn tuyệt đối, loại bỏ hoàn toàn rủi ro biến động dữ liệu (write mutation), đồng thời tối ưu hóa tốc độ nhờ cơ chế đồng bộ gốc của Replica Set thay vì sao lưu vật lý thủ công.

---

## **PHẦN I. SO SÁNH TỔNG QUAN HAI PHƯƠNG ÁN**

| Tiêu chí | Phương án 1 (Data-at-Rest + CSFLE - Khuyên dùng) | Phương án 2 (Chỉ CSFLE) |
| :--- | :--- | :--- |
| **Mức độ bảo mật** | **Tối đa (Gold Standard)**. Chống rò rỉ dữ liệu khi bị đánh cắp ổ cứng vật lý (at-rest) và chống lộ lọt từ Hacker/DBA cấp ứng dụng (CSFLE). | **Trung bình - Cao**. Chỉ chống lộ lọt từ tầng ứng dụng trở lên. Các tệp dữ liệu `.wt` trên đĩa cứng vẫn ở dạng thô không mã hóa. |
| **Độ phức tạp Hạ tầng** | Cần triển khai cụm **HashiCorp Vault HA** và cấu hình liên kết đĩa vật lý cho từng node MongoDB. | Rất thấp. Không can thiệp vào cấu hình DB Server, chỉ quản lý Master Key trên Vault bằng ứng dụng. |
| **Tổng Downtime Hệ thống**| **1.5 - 2.5 tiếng** (Dừng ứng dụng hoàn toàn từ lúc bắt đầu backup, đồng bộ đĩa tĩnh tuần tự 3 node, chạy tool mã hóa 60GB dữ liệu cũ, reindex ES và deploy). | **1.0 - 1.5 tiếng** (Dừng ứng dụng từ lúc bắt đầu backup, chạy tool mã hóa 60GB dữ liệu cũ, reindex ES và deploy). |
| **Trạng thái cụm DB lúc di cư** | Cụm DB chạy liên tục nhưng đóng băng 100% dữ liệu đầu vào (0 write/read từ app). | Cụm DB chạy liên tục nhưng đóng băng 100% dữ liệu đầu vào (0 write/read từ app). |

---

## **PHẦN II. PHƯƠNG ÁN 1: MÃ HÓA ĐA TẦNG (DATA-AT-REST + CSFLE) QUA STATIC ROLLING SYNC**

### **Mô hình hoạt động:**
* **Ứng dụng tạm dừng hoạt động** để đóng băng dữ liệu (không phát sinh ghi mới).
* **Cụm MongoDB vẫn hoạt động** để thực hiện đồng bộ đĩa tĩnh cuốn chiếu giữa các Node mà không cần dump/restore bằng file.

```mermaid
graph TD
    A[Dừng Ứng dụng - Đóng băng DB] --> B[Mã hóa Node 3 Secondary qua Initial Sync]
    B --> C[Mã hóa Node 2 Secondary qua Initial Sync]
    C --> D[Chuyển Primary sang Node 3 hoặc 2]
    D --> E[Mã hóa Node 1 Primary cũ qua Initial Sync]
    E --> F[Cụm DB đã mã hóa At-Rest 100%]
    F --> G[Chạy Tool mã hóa CSFLE cho dữ liệu cũ 60GB]
    G --> H[Reindex Elasticsearch & Deploy App mới bật CSFLE]
    H --> I[Mở lại Ứng dụng - Hoàn thành]
```

### **BƯỚC 1: CHUẨN BỊ HẠ TẦNG & PHẦN MỀM**

#### **1. Cấu hình HashiCorp Vault (KMS)**
Sử dụng công cụ quản lý khóa tĩnh qua **Vault KV Secrets Engine - Version 2** (Không yêu cầu bản Enterprise KMIP).
* Khởi tạo KV v2 tại đường dẫn: `secret/`
* Tạo bí mật (Secret) chứa Master Key cho MongoDB:
  ```bash
  vault kv put secret/mongodb/master-key key=$(openssl rand -base64 32)
  ```
* Tạo chính sách (Policy) cấp quyền đọc khóa cho MongoDB:
  ```hcl
  # mongodb-policy.hcl
  path "secret/data/mongodb/master-key" {
      capabilities = ["read"]
  }
  ```
* Tạo Token vĩnh viễn hoặc tự động gia hạn cho MongoDB và lưu vào file `/etc/mongodb/vault.token` trên từng node MongoDB Server.

#### **2. Viết Tool Di Cư Dữ Liệu (CSFLE Migration Tool)**
* Lập trình một công cụ riêng bằng Java (hoặc Python) chạy trên mạng LAN của DB.
* Tool hoạt động theo cơ chế **Batch & Rate-Limit** để chuyển đổi dữ liệu Plaintext cũ sang `BinData Subtype 6`.
* **Cú pháp cập nhật cốt lõi (Java):**
  ```java
  // Sử dụng ClientEncryption thủ công để mã hóa và cập nhật dữ liệu cũ
  byte[] plaintext = doc.getString("personalID").getBytes(StandardCharsets.UTF_8);
  BsonBinary encryptedVal = clientEncryption.encrypt(
      new BsonString(new String(plaintext)),
      new EncryptOptions("AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic").keyId(dekUuid)
  );
  collection.updateOne(eq("_id", doc.getObjectId("_id")), set("personalID", encryptedVal));
  ```

---

### **BƯỚC 2: TIẾN HÀNH THI CÔNG (MÃ HÓA STATIC ROLLING SYNC)**

> [!IMPORTANT]
> **THỜI GIAN BẢO TRÌ BẮT ĐẦU:** Dừng toàn bộ các ứng dụng kết nối tới cụm MongoDB. Đảm bảo số lượng Connection hoạt động về gần bằng 0 (chỉ còn các connection quản trị).

#### **1. Thực hiện mã hóa trên Node 3 (Secondary)**
1. Đăng nhập vào Node 3 và dừng dịch vụ MongoDB:
   ```bash
   systemctl stop mongod
   ```
2. Lưu Token của Vault vào `/etc/mongodb/vault.token` (chỉ cho phép user `mongod` đọc: `chmod 400`).
3. Chỉnh sửa tệp `/etc/mongod.conf`, cấu hình kích hoạt mã hóa tĩnh tích hợp Vault KV v2:
   ```yaml
   security:
     enableEncryption: true
     encryptionCipherMode: AES256-CBC
     vault:
       serverName: vault.company.internal
       port: 8200
       tokenFile: /etc/mongodb/vault.token
       secret: secret/data/mongodb/master-key
       # serverCAFile: /etc/mongodb/vault-ca.pem # (Bật nếu dùng Vault HTTPS)
   ```
4. **Xóa sạch thư mục dữ liệu hiện tại của Node 3** (MongoDB yêu cầu thư mục trống để khởi tạo mã hóa tĩnh):
   ```bash
   rm -rf /var/lib/mongodb/*
   ```
5. Khởi động lại dịch vụ MongoDB trên Node 3:
   ```bash
   systemctl start mongod
   ```
6. **Kiểm tra trạng thái đồng bộ:** Truy cập `mongosh` trên Node 1 (Primary) và chạy lệnh:
   ```javascript
   rs.status()
   ```
   *Node 3 sẽ ở trạng thái `STARTUP2` và bắt đầu chạy tự động **Initial Sync** để kéo 60GB dữ liệu từ Node 1. Toàn bộ dữ liệu ghi xuống đĩa Node 3 sẽ được mã hóa tĩnh 100%.*
   *Đợi cho đến khi trạng thái Node 3 chuyển sang `SECONDARY` (xanh).*

#### **2. Thực hiện mã hóa trên Node 2 (Secondary)**
1. Làm hoàn toàn tương tự 6 bước trên đối với Node 2.
2. Đợi Node 2 hoàn thành **Initial Sync** và chuyển sang trạng thái `SECONDARY` (xanh).

#### **3. Thực hiện chuyển giao Primary và Mã hóa Node 1 (Primary cũ)**
1. Đăng nhập vào `mongosh` trên Node 1 (Primary hiện tại), chạy lệnh nhường quyền:
   ```javascript
   rs.stepDown(60) // Buộc Node 1 trở thành Secondary trong vòng 60 giây
   ```
2. Cụm Replica Set sẽ tự động bầu chọn. Node 2 hoặc Node 3 (đã được mã hóa đĩa tĩnh) sẽ lên làm **Primary mới**.
3. Đăng nhập vào Node 1 (lúc này đã là Secondary) và thực hiện tương tự:
   * Stop dịch vụ `mongod`.
   * Cấu hình `/etc/mongod.conf` tích hợp Vault.
   * Xóa sạch `/var/lib/mongodb/*`.
   * Start dịch vụ `mongod` và chờ Node 1 đồng bộ xong thành `SECONDARY`.

*Đến đây, toàn bộ 3 Node trên đĩa cứng vật lý đã được mã hóa tĩnh hoàn toàn bằng Master Key của Vault.*

---

### **BƯỚC 3: DI CƯ DỮ LIỆU CSFLE (CLIENT-SIDE FIELD LEVEL ENCRYPTION)**

Vì dữ liệu 60GB vừa được đồng bộ hoàn toàn vẫn là dạng Plaintext (dữ liệu rõ), chúng ta cần chuyển đổi các trường nhạy cảm sang mã hóa:

1. **Khởi chạy Migration Tool Tối ưu (Java Multi-threaded Bulk Update):**
   * **Vấn đề nghẽn:** Với **100 triệu bản ghi**, nếu cập nhật tuần tự từng bản ghi (mất 100ms - 200ms/bản ghi do mạng RTT, lock index), việc di cư sẽ mất **115 ngày**. Kể cả tốc độ 1ms/bản ghi vẫn mất **27 tiếng**.
   * **Giải pháp tối ưu:** 
     * **Sử dụng `bulkWrite`:** Gộp từ **5,000 đến 10,000 bản ghi** vào một lô duy nhất để giảm 5,000 lần số lượt Network RTT.
     * **Mã hóa song song trên RAM (Parallel Processing):** Dùng Java `ForkJoinPool` tận dụng đa nhân CPU trên Server để mã hóa đồng thời trên RAM trước khi đóng gói gửi lệnh ghi xuống DB.
     * **Drop & Rebuild Index:** Tạm thời drop các index liên quan đến trường được mã hóa (`personalID`) trước khi chạy di cư và rebuild lại sau khi hoàn tất. Tốc độ ghi tăng gấp **3 - 5 lần**.
     * **Write Concern nới lỏng:** Sử dụng `{ w: 1, j: false }` trong quá trình bảo trì để DB phản hồi nhanh nhất.

##### **Mã nguồn Ví dụ Java Migration Tool tối ưu (Multi-threaded & Bulk Write):**
```java
package com.company.security;

import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.WriteConcern;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import com.mongodb.client.model.UpdateOneModel;
import com.mongodb.client.model.WriteModel;
import com.mongodb.client.model.BulkWriteOptions;
import com.mongodb.client.vault.ClientEncryption;
import com.mongodb.client.vault.ClientEncryptions;
import com.mongodb.ClientEncryptionSettings;
import org.bson.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

public class BulkCsfleMigrationTool {

    private static final String CONNECTION_STRING = "mongodb://admin:secret@localhost:27017/?replicaSet=rs0";
    private static final String DB_NAME = "companyDb";
    private static final String COLLECTION_NAME = "employees";
    private static final int BATCH_SIZE = 5000; // Số lượng bản ghi xử lý trong 1 lô (Batch)
    private static final int THREAD_POOL_SIZE = Runtime.getRuntime().availableProcessors(); // Số luồng song song theo số nhân CPU

    public static void main(String[] args) throws Exception {
        byte[] masterKey = fetchMasterKeyFromVault();
        Map<String, Map<String, Object>> kmsProviders = new HashMap<>();
        Map<String, Object> localProvider = new HashMap<>();
        localProvider.put("key", masterKey);
        kmsProviders.put("local", localProvider);

        UUID dekUuid = getExistingDataEncryptionKey();
        
        MongoClientSettings clientSettings = MongoClientSettings.builder()
                .applyConnectionString(new ConnectionString(CONNECTION_STRING))
                // Tối ưu hóa Write Concern để tăng tốc độ ghi tĩnh lúc bảo trì
                .writeConcern(WriteConcern.W1.withJournal(false)) 
                .build();

        try (MongoClient mongoClient = MongoClients.create(clientSettings)) {
            MongoDatabase database = mongoClient.getDatabase(DB_NAME);
            MongoCollection<Document> collection = database.getCollection(COLLECTION_NAME);

            ClientEncryptionSettings clientEncryptionSettings = ClientEncryptionSettings.builder()
                    .keyVaultMongoClientSettings(clientSettings)
                    .keyVaultNamespace("encryption.__keyVault")
                    .kmsProviders(kmsProviders)
                    .build();

            try (ClientEncryption clientEncryption = ClientEncryptions.create(clientEncryptionSettings)) {
                
                System.out.println("Bắt đầu di cư dữ liệu cũ bằng cơ chế Parallel Bulk Write...");
                long startTime = System.currentTimeMillis();

                // 1. Lấy danh sách các ID cần mã hóa (đọc tuần tự cursor tránh tràn RAM)
                // Lọc các tài liệu có trường 'personalID' ở dạng String (chưa bị mã hóa thành BinData)
                BsonDocument filter = new BsonDocument("personalID", new BsonDocument("$type", new BsonString("string")));
                
                List<Document> batch = new ArrayList<>();
                long totalProcessed = 0;
                
                // ExecutorService quản lý đa luồng mã hóa trên RAM
                ExecutorService executor = Executors.newFixedThreadPool(THREAD_POOL_SIZE);

                for (Document doc : collection.find(filter).noCursorTimeout(true).batchSize(BATCH_SIZE)) {
                    batch.add(doc);
                    if (batch.size() >= BATCH_SIZE) {
                        processBatchParallel(collection, clientEncryption, dekUuid, batch, executor);
                        totalProcessed += batch.size();
                        System.out.println("Đã mã hóa và ghi thành công: " + totalProcessed + " bản ghi.");
                        batch.clear();
                    }
                }
                
                // Xử lý nốt batch cuối cùng nếu còn dư
                if (!batch.isEmpty()) {
                    processBatchParallel(collection, clientEncryption, dekUuid, batch, executor);
                    totalProcessed += batch.size();
                    System.out.println("Đã mã hóa và ghi thành công nốt: " + totalProcessed + " bản ghi.");
                }

                executor.shutdown();
                executor.awaitTermination(1, TimeUnit.HOURS);
                
                long duration = System.currentTimeMillis() - startTime;
                System.out.println("HOÀN TẤT DI CƯ! Tổng số bản ghi đã xử lý: " + totalProcessed);
                System.out.println("Thời gian thực hiện: " + (duration / 1000 / 60) + " phút.");
            }
        }
    }

    /**
     * Xử lý song song mã hóa trên RAM cho 1 lô (Batch) và gửi bulkWrite xuống DB
     */
    private static void processBatchParallel(
            MongoCollection<Document> collection,
            ClientEncryption clientEncryption,
            UUID dekUuid,
            List<Document> batch,
            ExecutorService executor
    ) throws Exception {
        
        // Chia việc mã hóa RAM cho Thread Pool xử lý đồng thời
        List<CompletableFuture<WriteModel<Document>>> futures = batch.stream()
                .map(doc -> CompletableFuture.supplyAsync(() -> {
                    Object rawId = doc.get("personalID");
                    if (rawId instanceof String) {
                        BsonBinary encryptedId = clientEncryption.encrypt(
                                new BsonString((String) rawId),
                                new com.mongodb.client.model.vault.EncryptOptions("AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic")
                                        .keyId(new BsonBinary(dekUuid))
                        );
                        
                        // Đóng gói lệnh cập nhật đè
                        BsonDocument filter = new BsonDocument("_id", new BsonObjectId(doc.getObjectId("_id")));
                        BsonDocument update = new BsonDocument("$set", new BsonDocument("personalID", encryptedId));
                        return new UpdateOneModel<Document>(filter, update);
                    }
                    return null;
                }, executor))
                .collect(Collectors.toList());

        // Chờ tất cả các luồng mã hóa RAM của batch hiện tại chạy xong
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();

        // Thu thập kết quả lệnh Update và lọc bỏ phần null
        List<WriteModel<Document>> updates = new ArrayList<>();
        for (CompletableFuture<WriteModel<Document>> future : futures) {
            WriteModel<Document> model = future.get();
            if (model != null) {
                updates.add(model);
            }
        }

        // Thực hiện Bulk Write xuống MongoDB (Unordered giúp server xử lý song song các luồng ghi)
        if (!updates.isEmpty()) {
            collection.bulkWrite(updates, new BulkWriteOptions().ordered(false));
        }
    }

    private static byte[] fetchMasterKeyFromVault() {
        // Mock lấy khóa từ Vault
        return new byte[96];
    }

    private static UUID getExistingDataEncryptionKey() {
        // Tra cứu UUID từ keyVault
        return UUID.randomUUID();
    }
}
```
   * *Nhờ thiết kế gộp lô 5,000 bản ghi và chạy mã hóa đa luồng trên RAM của máy chủ Application Server, tốc độ di cư thực tế trên mạng LAN 10Gbps kết hợp SSD Enterprise có thể dễ dàng đạt từ **15,000 đến 25,000 bản ghi/giây**, rút ngắn toàn bộ thời gian di cư 100 triệu bản ghi xuống chỉ còn khoảng **1.1 đến 1.8 tiếng**.*
2. **Reindex Elasticsearch:**
   * Sau khi tool chạy xong, tiến hành kích hoạt tính năng đồng bộ và reindex lại Elasticsearch đối với các thực thể liên quan đến trường vừa được mã hóa để đảm bảo công cụ tìm kiếm hoạt động chính xác.
3. **Deploy Ứng dụng Mới:**
   * Triển khai code mới của ứng dụng (đã kích hoạt cấu hình tự động mã hóa/giải mã CSFLE).
4. **Kích hoạt lại Ứng dụng (Mở lại hệ thống):**
   * Khởi động lại toàn bộ máy chủ ứng dụng cho người dùng truy cập.
   * **DOWNTIME KẾT THÚC.**

---

## **PHẦN III. PHƯƠNG ÁN 2: CHỈ KÍCH HOẠT MÃ HÓA CẤP TRƯỜNG (CSFLE)**

### **Mô hình hoạt động:**
* Bỏ qua hoàn toàn việc cấu hình mã hóa tĩnh tại tầng lưu trữ vật lý của MongoDB Server.
* Không restart, không wipe dữ liệu các Node DB.
* Toàn bộ quy trình chỉ xử lý ở phía Ứng dụng (Client) và di cư dữ liệu.

```mermaid
graph TD
    A[Dừng Ứng dụng - Đóng băng DB] --> B[Chạy Tool mã hóa CSFLE cho dữ liệu cũ 60GB]
    B --> C[Reindex Elasticsearch]
    C --> D[Deploy App mới bật cấu hình tự động mã hóa CSFLE]
    D --> E[Mở lại Ứng dụng - Hoàn thành]
```

### **Các bước thực hiện chi tiết:**

#### **1. Bước chuẩn bị:**
* Khởi tạo **HashiCorp Vault** phục vụ lưu trữ Master Key (không cần thiết lập tích hợp sâu vào hạ tầng MongoDB Server).
* Viết Tool Di Cư Dữ Liệu (CSFLE Migration Tool) tương tự như ở Bước 1 của PA1.

#### **2. Bước thi công (Downtime hệ thống 1.0 - 1.5 tiếng):**
1. **Dừng toàn bộ ứng dụng cũ** (đóng băng dữ liệu đầu vào).
2. **Chạy Migration Tool:** Quét và chuyển đổi toàn bộ dữ liệu thô nhạy cảm cũ (`personalID`, `phoneNumber`) thành `BinData Subtype 6` trên RAM và cập nhật lại DB.
3. **Reindex dữ liệu trên Elasticsearch** để cập nhật chỉ mục tìm kiếm mới.
4. **Deploy phiên bản ứng dụng mới** (đã bật cấu hình tự động CSFLE trong MongoClient).
5. **Khởi động lại ứng dụng** cho người dùng truy cập.
   * **DOWNTIME KẾT THÚC.**

---

## **PHẦN IV. KỊCH BẢN PHÒNG NGỪA RỦI RO & KHÔI PHỤC (ROLLBACK PLAN)**

Trong trường hợp xảy ra sự cố đột xuất trong quá trình thi công bảo trì:

### **1. Kịch bản Rollback cho PA1 (Mã hóa đĩa tĩnh bị lỗi kết nối Vault)**
* **Tình huống:** Node đang đồng bộ không kết nối được Vault hoặc quá trình Initial Sync bị treo kéo dài.
* **Cách xử lý:** 
  1. Dừng dịch vụ `mongod` trên Node đang cấu hình lỗi.
  2. Khôi phục lại cấu hình gốc của tệp `/etc/mongod.conf` (Xóa bỏ phần `security.vault` và `enableEncryption`).
  3. Xóa thư mục dữ liệu bị lỗi `/var/lib/mongodb/*`.
  4. Khởi động lại Node, nó sẽ tự động đồng bộ lại dưới dạng không mã hóa từ Node Primary cũ. Hệ thống trở về trạng thái an toàn ban đầu.

### **2. Kịch bản Rollback cho CSFLE (Tool di cư bị lỗi giữa chừng)**
* **Tình huống:** Tool di cư dữ liệu cũ bị crash hoặc gặp lỗi logic nghiệp vụ làm sai lệch dữ liệu.
* **Cách xử lý:** 
  1. Thực hiện khôi phục (Restore) lại collection nghiệp vụ bị lỗi từ bản snapshot/backup dự phòng được tạo ngay trước thời điểm dừng App.
  2. Deploy lại phiên bản ứng dụng cũ (chưa bật CSFLE) để hệ thống hoạt động trở lại bình thường.
  3. Tiến hành phân tích log của tool để sửa lỗi trước khi lập kế hoạch bảo trì lần sau.
