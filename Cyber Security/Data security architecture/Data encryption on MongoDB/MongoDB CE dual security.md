# **TÀI LIỆU KIẾN TRÚC TỔNG QUÁT: BẢO MẬT DỮ LIỆU ĐA TẦNG (DATA SECURITY ARCHITECTURE)**

*Tài liệu này áp dụng với công nghệ Hashicop Vault và Percona MongoDB CE*

## **I\. TỔNG QUAN KIẾN TRÚC (ARCHITECTURE OVERVIEW)**

Hệ thống áp dụng mô hình bảo mật **"Zero-Trust Data"** bằng cách kết hợp hai giải pháp mã hóa độc lập để bảo vệ dữ liệu ở mọi trạng thái:

1. **Mã hóa cấp trường phía khách (CSFLE):** Thực hiện tại tầng **Dịch vụ Nghiệp vụ (Business Service)**. Dữ liệu nhạy cảm được mã hóa trước khi truyền qua mạng và ghi vào DB. Chống rò rỉ dữ liệu ngay cả khi Quản trị viên DB (DBA) hoặc Hacker chiếm được quyền root của hệ quản trị cơ sở dữ liệu.  
2. **Mã hóa dữ liệu tĩnh (Data-at-Rest Encryption):** Thực hiện tại tầng Lưu trữ (Storage Engine) của Percona MongoDB. Toàn bộ file dữ liệu vật lý lưu trên ổ cứng sẽ được mã hóa, chống rò rỉ dữ liệu khi bị đánh cắp ổ đĩa vật lý hoặc sao chép trộm file database.

Tất cả các khóa mẹ (Master Keys) điều khiển hai tính năng trên được quản lý tập trung và nghiêm ngặt tại cụm **HashiCorp Vault**.

![alt text](medias/secureflow-1.png)

```plantuml

@startuml
skinparam backgroundColor white
skinparam roundcorner 10
skinparam shadowing true

' Định nghĩa các dải màu phân vùng hệ thống
!define VAULT_COLOR #F96
!define APP_COLOR #69C
!define DB_COLOR #439A86
!define DISK_COLOR #666

rectangle "HashiCorp Vault / OpenBao\n(Central KMS)" as Vault_Zone #F96 {
    artifact "App Master Key\n(Mở bọc khóa con DEK)" as AppMasterKey
    artifact "Disk Key\n(Mã hóa hạ tầng đĩa)" as DBDiskKey
}

rectangle "Tầng Ứng Dụng (On-premise)\n[Xử lý Dữ liệu thô - Plaintext]" as App_Zone #69C {
    node "Dịch vụ Nghiệp vụ\n(Business Service)" as BusinessService
}

rectangle "Tầng Dữ Liệu (Percona MongoDB CE)\n[Kho chứa câm - Chỉ nhận BinData]" as DB_Zone #439A86 {
    database "Cụm Replica Set 3 Node" as MongoCluster
    collections "Collection: encryption.__keyVault\n(Lưu trữ các Khóa con DEK đã bọc)" as KeyVaultColl
}

rectangle "Hệ Thống Lưu Trữ Vật Lý" as Disk_Zone {
    storage "Các Ổ Cứng Vật Lý\n[Mã hóa khối tĩnh]" as HardDisks #666
}

' Thể hiện các luồng tương tác tổng quát
BusinessService -up-> AppMasterKey : "1. Gọi lấy Master Key mở bọc DEK"
BusinessService <--> KeyVaultColl : "2. Đọc/Ghi khóa con DEK đã bọc"

BusinessService --> MongoCluster : "3. Gửi câu lệnh Insert / Query\n[Trường nhạy cảm dạng BinData 6]"
MongoCluster -up-> DBDiskKey : "4. Khởi động: Gọi lấy khóa đĩa"
MongoCluster ==> HardDisks : "5. Mã hóa toàn khối dữ liệu\ntrước khi ghi xuống đĩa"

@enduml

```

## **II\. THIẾT KẾ CHI TIẾT**

### 1. **Mã hóa cấp trường phía khách (CSFLE):**

#### 1.1. **Khởi tạo và sinh khóa con (DEK Generation)**
Trước khi ứng dụng có thể mã hóa dữ liệu, nó tự sinh ra con ngẫu nhiên (Plaintext DEK), gửi lên Vault để bọc bảo mật bằng Master Key, sau đó lưu vào bộ lưu trữ tập trung của MongoDB dưới dạng đã bọc (Encrypted DEK).
![alt text](medias/csfle1.1.png)

```plantuml
@startuml
skinparam backgroundColor white
skinparam roundcorner 10
skinparam shadowing true
skinparam SequenceMessageAlign center

actor "Đội ngũ Vận hành / Dev" as Admin
participant "Dịch vụ Nghiệp vụ\n(Business Service / RAM)" as App #69C
participant "HashiCorp Vault / OpenBao\n(Central KMS)" as Vault #F96
database "Percona MongoDB\n(Collection: encryption.__keyVault)" as DB #439A86

== QUY TRÌNH SINH KHÓA CON (DEK) VÀ BỌC BẢO MẬT ==

Admin -> App : 1. Kích hoạt lệnh thiết lập hệ thống bảo mật (Setup)
activate App

App -> App : 2. Sinh chuỗi ngẫu nhiên chất lượng cao trên RAM\n(Plaintext DEK - Khóa gốc chạm vào dữ liệu)

App -> Vault : 3. Gửi Plaintext DEK nhờ Vault bọc lại\n(Dùng Transit Secrets Engine)
activate Vault
note over Vault
  **Tại Vault:**
  Dùng App Master Key để mã hóa 
  chuỗi DEK gốc thành khối Ciphertext.
end note
Vault --> App : 4. Trả về chuỗi DEK đã bọc bảo mật (Encrypted DEK)
deactivate Vault

App -> App : 5. Đóng gói Encrypted DEK kèm tên bí danh\n(Ví dụ đặt tên bí danh: "employee-secrets-key")

App -> DB : 6. Lưu tài liệu khóa đã bọc vào kho chứa tập trung
activate DB
note over DB
  Lưu vào collection đặc biệt:
  **"encryption.__keyVault"**
end note
DB --> App : 7. Xác nhận ghi khóa thành công (Trả về UUID của khóa)
deactivate DB

App --> Admin : 8. Trả về thông tin cấu hình thành công\n(Quy ước tên bí danh "employee-secrets-key" cho đội Dev)
deactivate App
@startuml
```

#### 1.2. **Mã hóa và ghi mới dữ liệu (CSFLE Insert Data)**

Khi thực hiện ghi bản ghi mới, Dịch vụ Nghiệp vụ tìm khóa con bằng tên bí danh, mã hóa trường thông tin nhạy cảm thành chuỗi nhị phân BinData Subtype 6 trên RAM rồi mới đẩy câu lệnh qua mạng tới MongoDB.

![alt text](medias/csfle1.2.png)

```plantuml
@startuml
skinparam backgroundColor white
skinparam roundcorner 10
skinparam shadowing true
skinparam SequenceMessageAlign center

participant "Logic Nghiệp vụ\n(Mã nguồn App)" as Logic
participant "MongoDB Driver\n(RAM của App)" as Driver #69C
database "Percona MongoDB\n(Collection __keyVault)" as DBKey
database "Percona MongoDB\n(Collection Nghiệp vụ)" as DBData

== QUY TRÌNH MÃ HÓA TRƯỚC, GỬI BẢN GHI SAU ==

Logic -> Driver : 1. Gọi lệnh Insert bản ghi kèm Plaintext\n(Chỉ định dùng khóa: "employee-secrets-key")
activate Driver

Driver -> DBKey : 2. Tra cứu Khóa con trong kho theo tên bí danh
activate DBKey
DBKey --> Driver : 3. Trả về Khóa con DEK (Trạng thái đang bị bọc)
deactivate DBKey

note over Driver
  **Xử lý trên RAM của App:**
  - Driver gọi sang Vault để mở bọc lấy DEK thô.
  - Dùng DEK thô băm số Căn cước thô thành khối nhị phân.
end note
Driver -> Driver : 4. Biến đổi dữ liệu thô thành **BinData Subtype 6**\n(Khối nhị phân mang theo thẻ tên UUID của khóa)

Driver -> DBData : 5. Lệnh INSERT chính thức (Trường nhạy cảm gửi đi là BinData 6)
activate DBData
note over DBData
  **Tại MongoDB Server:**
  Nhận bản ghi và lưu trữ dạng nhị phân.
  Server không có khóa, không đọc được Plaintext.
end note
DBData --> Driver : 6. Xác nhận ghi bản ghi thành công
deactivate DBData

Driver --> Logic : 7. Phản hồi hoàn tất tác vụ Nghiệp vụ
deactivate Driver
@enduml
```

#### 1.3. **Mã hóa giá trị tìm kiếm (CSFLE Query / Find Data)**

Do dữ liệu dưới database là chuỗi nhị phân, ứng dụng không thể tìm kiếm theo dạng chữ thô. Nó buộc phải mã hóa từ khóa tìm kiếm thành dạng nhị phân tương ứng (sử dụng thuật toán Deterministic) trước khi gửi truy vấn sang MongoDB Server để thực hiện so khớp byte-đối-byte.

![alt text](medias/csfle1.3.png)

```plantuml
@startuml
skinparam backgroundColor white
skinparam roundcorner 10
skinparam shadowing true
skinparam SequenceMessageAlign center

participant "Logic Nghiệp vụ\n(Mã nguồn App)" as Logic
participant "MongoDB Driver\n(RAM của App)" as Driver #69C
database "Percona MongoDB\n(Collection Nghiệp vụ)" as DBData

== QUY TRÌNH MÃ HÓA TỪ KHÓA TRƯỚC KHI TRUY VẤN ==

Logic -> Driver : 1. Tìm nhân viên có Căn cước thô = "0123456789"\n(Yêu cầu tìm kiếm khớp chính xác - Deterministic)
activate Driver

note over Driver
  **Xử lý mã hóa từ khóa tại App:**
  Driver sử dụng lại Khóa con DEK thô trên RAM 
  để mã hóa chữ thô "0123456789" ra khối nhị phân trùng khớp.
end note
Driver -> Driver : 2. Chuyển đổi từ khóa thô sang nhị phân **BinData 6**

Driver -> DBData : 3. Thực hiện lệnh FIND với tham số: { personalID: BinData 6 }
activate DBData
note over DBData
  **Tại MongoDB Server:**
  Server thực hiện quét chỉ mục (Index) hoặc 
  so khớp byte-đối-byte giữa chuỗi nhị phân nhận vào 
  với chuỗi nhị phân đang nằm dưới DB.
end note
DBData --> Driver : 4. Trả về tài liệu (Document) tìm thấy phù hợp
deactivate DBData

Driver --> Logic : 5. Chuyển tiếp bản ghi sang bước xử lý tiếp theo
deactivate Driver
@enduml
```

#### 1.4. **Giải mã và trả về dữ liệu (CSFLE Read Data)**

Khi nhận khối nhị phân đổ về từ MongoDB, Driver chạy trên RAM của ứng dụng tự động bóc tách tiêu đề dữ liệu để lấy mã UUID (Key ID), tìm đúng bản ghi khóa con trong __keyVault, gửi sang Vault mở bọc và dịch ngược dữ liệu về chữ thô ban đầu.

![alt text](medias/csfle1.4.png)

```plantuml
@startuml
skinparam backgroundColor white
skinparam roundcorner 10
skinparam shadowing true
skinparam SequenceMessageAlign center

participant "Logic Nghiệp vụ\n(Mã nguồn App)" as Logic
participant "MongoDB Driver\n(RAM của App)" as Driver #69C
database "Percona MongoDB\n(Collection __keyVault)" as DBKey
participant "HashiCorp Vault / OpenBao\n(Central KMS)" as Vault

== QUY TRÌNH NHẬN DỮ LIỆU NHỊ PHÂN VÀ TỰ ĐỘNG GIẢI MÃ ==

activate Driver
note over Driver
  **Tiếp tục từ Bước 3 (Sau khi nhận dữ liệu từ DB):**
  Driver quét trường personalID, phát hiện nhãn **BinData 6**.
  Đọc phần tiêu đề lấy ra mã số khóa: **UUID = "k-99-xyz"**.
end note

Driver -> DBKey : 1. Truy vết: Tìm bản ghi khóa có _id = "k-99-xyz"
activate DBKey
DBKey --> Driver : 2. Trả về Khóa con DEK tương ứng (Trạng thái đang bị bọc)
deactivate DBKey

Driver -> Vault : 3. Gửi Khóa con bị bọc lên nhờ mở khóa phong bì
activate Vault
Vault --> Driver : 4. Trả lại Khóa con DEK dạng thô (Plaintext DEK) về RAM của App
deactivate Vault

Driver -> Driver : 5. Dùng Khóa con DEK thô giải mã khối dữ liệu nhị phân\ntrở lại thành chữ thô nguyên bản: **"0123456789"**

Driver --> Logic : 6. Trả kết quả cuối cùng chứa dữ liệu thô rõ ràng cho code nghiệp vụ xử lý
deactivate Driver
@enduml
```

### 2. Data-at-Rest Encryption (Mã hóa dữ liệu tĩnh) 

Tầng lưu trữ WiredTiger của Percona MongoDB kết nối trực tiếp đến Vault để lấy Disk Key khi khởi động cụm. Quá trình đồng bộ hóa giữa các node Secondary vẫn giữ nguyên trạng thái nhị phân, và hành động mã hóa đĩa diễn ra ngay trước khi các block dữ liệu được đổ xuống ổ cứng vật lý.

![alt text](medias/data-at-rest.png)

```plantuml
@startuml
skinparam backgroundColor white
skinparam roundcorner 10
skinparam shadowing true
skinparam orientation left to right

' Định nghĩa màu sắc các thành phần
!define VAULT_COLOR #F96
!define DB_COLOR #439A86
!define DISK_COLOR #666

' 1. Kho khóa trung tâm
rectangle "HashiCorp Vault / OpenBao" as Vault #F96 {
    artifact "Disk Key\n(path: secret/data/infra/disk-key)" as DiskKey
}

' 2. Tầng xử lý cơ sở dữ liệu (RAM/CPU Server)
rectangle "Percona MongoDB Node\n(WiredTiger Storage Engine)" as MongoServer #439A86 {
    component "RAM Cache\n[Dữ liệu dạng BinData 6]" as Cache
    component "Bộ mã hóa khối\n[Block Encryption]" as Encryptor
}

' 3. Tầng lưu trữ vật lý (Disk)
rectangle "Hệ thống lưu trữ vật lý" as DiskZone {
    storage "Ổ CỨNG VẬT LÝ\n[File hệ thống .wt đã mã hóa]" as PhysicalDisk #666
}

' ==========================================
' Cấu trúc luồng dữ liệu di chuyển (Flow)
' ==========================================

' Luồng lấy khóa khi khởi động hệ thống
DiskKey -down-> Encryptor : " [1] Cấp khóa đĩa\nkhi DB khởi động"

' Luồng xử lý ghi dữ liệu xuống đĩa
Cache -right-> Encryptor : " [2] Đẩy block dữ liệu\nxuống tầng lưu trữ"

note top of Encryptor
  **Hành động của WiredTiger:**
  Dùng Disk Key mã hóa toàn bộ 
  Block dữ liệu trên RAM trước khi ghi.
end note

Encryptor -right-> PhysicalDisk : " [3] Ghi block dữ liệu\nđã mã hóa tĩnh (At-Rest)"

@enduml
```
