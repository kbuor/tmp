# AIFA Dev UAT-Ready Infrastructure on TPCOMS
# Tài liệu hiện trạng hạ tầng và hướng dẫn test end-to-end

**Ngày lập:** 19/05/2026  
**Môi trường:** `aifa-dev`  
**Nền tảng triển khai:** TPCOMS On-Premise Cloud, VMware Cloud Director, Kubernetes, Harbor, PostgreSQL, Redis, RabbitMQ, MinIO  
**Trạng thái hiện tại:** UAT-ready cho tầng backend/API/worker/data runtime. Frontend UI chưa có artifact trong package AIFA đã gửi.

---

## 1. Tóm tắt trạng thái hiện tại

Môi trường AIFA Dev đã được dựng trên hạ tầng TPCOMS On-Premise và đã hoàn tất validation runtime chính.

Các thành phần backend/worker/data service chính đã hoạt động ổn định:

| Thành phần | Trạng thái | Ghi chú |
|---|---:|---|
| `aifa-admin-web-be` | PASS | 2 replicas, health `200 OK` qua Ingress/LB |
| `aifa-client-backend` | PASS | 2 replicas, health `200 OK` qua Ingress/LB |
| `aifa-mcp-backend` | PASS | MCP SSE service hoạt động nội bộ qua Kubernetes |
| `aifa-process-file-upload-worker` | PASS | Consumer RabbitMQ hoạt động, upload file đã validate |
| `aifa-run-process-worker` | PASS | Consumer RabbitMQ hoạt động, run-process đã validate end-to-end |
| PostgreSQL | PASS | Lưu documents, NKC data, report process result |
| Redis | PASS | Runtime cache, process status, client backend cache |
| RabbitMQ | PASS | Queue chính sạch, DLQ sạch |
| MinIO | PASS | Lưu object test input và uploaded files |
| Harbor | PASS | Lưu image nội bộ, Kubernetes pull image thành công |
| Ingress/LB | PASS | Admin API và Client API health `200 OK` |
| Frontend UI | CHƯA CÓ ARTIFACT | AIFA cần gửi frontend image/source hoặc trỏ frontend hiện hữu về API TPCOMS |

Kết quả validation quan trọng:

- Upload file đã PASS với 4 loại file:
  - NKC Excel
  - CDTK Excel
  - PDF BCTC
  - XML GTGT
- NKC Excel đã ghi thành công **5482 rows** vào PostgreSQL.
- Run-process đã PASS end-to-end qua:
  - RabbitMQ
  - Redis
  - Vertex/LLM
  - MCP SSE
  - MCP tools
  - PostgreSQL
- RabbitMQ queue chính và DLQ đều sạch.
- Các backend public qua Ingress/LB đều trả health `200 OK`.

---

## 2. Phạm vi tài liệu

Tài liệu này mô tả toàn bộ những gì AIFA hiện đang có trên hạ tầng TPCOMS, bao gồm:

- Hạ tầng VM
- Mạng nội bộ, DNS, Load Balancer
- Kubernetes cluster
- Harbor image registry
- Data services: PostgreSQL, Redis, RabbitMQ, MinIO
- Kubernetes namespace `aifa-dev`
- Deployment, Service, Ingress
- ConfigMap, Secret, image tag
- Worker queue, DLQ
- Kết quả validation
- Endpoint để AIFA test
- Checklist để test end-to-end
- Các điểm còn thiếu để thành production/public application hoàn chỉnh

Tài liệu **không chứa giá trị secret/password/key**. Các key nhạy cảm chỉ được mô tả theo tên biến hoặc vai trò.

---

## 3. Kiến trúc tổng quan

Luồng tổng quan của môi trường hiện tại:

```text
AIFA user / frontend
        |
        | HTTP
        v
aifa-lb-01 / HAProxy / Ingress NGINX
        |
        +--> aifa-admin-web-be:8000
        |
        +--> aifa-client-backend:5000
                    |
                    +--> RabbitMQ
                    |       |
                    |       +--> aifa-process-file-upload-worker
                    |       +--> aifa-run-process-worker
                    |
                    +--> PostgreSQL
                    +--> Redis
                    +--> MinIO
                    +--> MCP backend
                            |
                            +--> MCP tools
                            +--> LLM/Vertex/OpenAI integrations
```

Môi trường hiện tại đã có đầy đủ phần backend/runtime. Để người dùng test qua UI thật, cần một trong hai hướng:

- AIFA dùng frontend hiện hữu và trỏ API base URL sang endpoint TPCOMS.
- AIFA gửi frontend Docker image/source build để TPCOMS deploy thêm frontend UI trên Kubernetes.

---

## 4. Hạ tầng VM

Tất cả VM thuộc domain nội bộ:

```text
aifa.tpcloud
```

Timezone chuẩn:

```text
Asia/Ho_Chi_Minh
```

OS chuẩn:

```text
Ubuntu 24.04.4 LTS
```

Danh sách VM:

| VM | IP nội bộ | Vai trò |
|---|---:|---|
| `aifa-dns-01` | `10.80.10.2` | DNS server nội bộ cho domain `aifa.tpcloud` |
| `aifa-bastion-01` | `10.80.10.10` | Bastion/ops host |
| `aifa-lb-01` | `10.80.10.20` | Load Balancer / HAProxy, entrypoint HTTP/HTTPS |
| `aifa-registry-01` | `10.80.10.30` | Harbor Registry |
| `aifa-k8s-cp-01` | `10.80.20.11` | Kubernetes control-plane |
| `aifa-k8s-worker-01` | `10.80.20.21` | Kubernetes worker |
| `aifa-k8s-worker-02` | `10.80.20.22` | Kubernetes worker |
| `aifa-data-01` | `10.80.30.11` | Data node: PostgreSQL, Redis, RabbitMQ, MinIO |

Subnet đang sử dụng:

| Subnet | Vai trò |
|---|---|
| `10.80.10.0/24` | Management / shared services |
| `10.80.20.0/24` | Kubernetes nodes |
| `10.80.30.0/24` | Data services |

---

## 5. DNS nội bộ

DNS server:

```text
aifa-dns-01.aifa.tpcloud
10.80.10.2
```

Domain nội bộ:

```text
aifa.tpcloud
```

Các record quan trọng:

| Hostname | Target | Ghi chú |
|---|---:|---|
| `harbor.aifa.tpcloud` | `10.80.10.30` | Harbor Registry |
| `registry.aifa.tpcloud` | `10.80.10.30` | Alias registry |
| `k8s-api.aifa.tpcloud` | `10.80.20.11` | Kubernetes API endpoint hiện tại |
| `postgres.aifa.tpcloud` | `10.80.30.11` | Data node/PostgreSQL |
| `aifa-admin-dev.aifa.tpcloud` | `10.80.10.20` | Admin Backend API qua LB/Ingress |
| `aifa-client-dev.aifa.tpcloud` | `10.80.10.20` | Client Backend API qua LB/Ingress |

Ghi chú:

- Nếu AIFA test từ mạng nội bộ/VPN, các hostname cần resolve về `10.80.10.20`.
- Nếu AIFA cần test từ Internet, cần cấu hình public DNS và NAT/firewall từ public IP về `aifa-lb-01`.

---

## 6. Load Balancer và Ingress

VM load balancer:

```text
aifa-lb-01
10.80.10.20
```

Vai trò:

- Nhận traffic HTTP/HTTPS từ người dùng hoặc frontend.
- Forward traffic vào Ingress NGINX trên Kubernetes worker nodes.
- Đóng vai trò entrypoint cho các domain dev/UAT.

Ingress Controller:

```text
Ingress NGINX
Namespace: ingress-nginx
Mode: NodePort
HTTP NodePort: 30080
HTTPS NodePort: 30443
```

HAProxy trên `aifa-lb-01` đang forward:

```text
aifa-lb-01:80  -> aifa-k8s-worker-01:30080 / aifa-k8s-worker-02:30080
aifa-lb-01:443 -> aifa-k8s-worker-01:30443 / aifa-k8s-worker-02:30443
```

Ingress hiện có trong namespace `aifa-dev`:

| Ingress | Host | Service backend | Port |
|---|---|---|---:|
| `aifa-admin-web-be` | `aifa-admin-dev.aifa.tpcloud` | `aifa-admin-web-be` | `8000` |
| `aifa-client-backend` | `aifa-client-dev.aifa.tpcloud` | `aifa-client-backend` | `5000` |

Health check qua LB:

```bash
curl -i -H "Host: aifa-admin-dev.aifa.tpcloud" http://10.80.10.20/health

curl -i -H "Host: aifa-client-dev.aifa.tpcloud" http://10.80.10.20/health
```

Kết quả hiện tại:

```text
HTTP/1.1 200 OK
```

---

## 7. Kubernetes Cluster

Phiên bản Kubernetes:

```text
Kubernetes v1.36.1
```

Cài đặt bằng:

```text
kubeadm
```

Container runtime:

```text
containerd 2.2.1
```

CNI:

```text
Calico
Pod CIDR: 192.168.0.0/16
```

Nodes:

| Node | IP | Role | Trạng thái |
|---|---:|---|---|
| `aifa-k8s-cp-01` | `10.80.20.11` | control-plane | Ready |
| `aifa-k8s-worker-01` | `10.80.20.21` | worker | Ready |
| `aifa-k8s-worker-02` | `10.80.20.22` | worker | Ready |

Các add-on đã cài:

| Add-on | Trạng thái |
|---|---|
| Calico | Running |
| CoreDNS | Running |
| Metrics Server | Running |
| Helm | Available |
| Ingress NGINX | Running |
| Harbor CA trust | Đã trust trên Kubernetes nodes |

Namespace ứng dụng:

```text
aifa-dev
```

---

## 8. Harbor Registry

Registry nội bộ:

```text
harbor.aifa.tpcloud
10.80.10.30
```

Harbor version:

```text
v2.13.0
```

TLS:

- Harbor đang chạy HTTPS bằng internal CA/certificate.
- Kubernetes nodes đã trust Harbor CA.
- `crictl pull` từ Kubernetes nodes đã validate thành công.

Project Harbor chính:

```text
aifa-dev
```

ImagePullSecret trong Kubernetes:

```text
harbor-regcred
```

Image tag đang dùng cho UAT:

```text
v20260519-aifa-rebuild-v2
```

Các image AIFA rebuild v2 đã deploy:

| Component | Image |
|---|---|
| Admin backend | `harbor.aifa.tpcloud/aifa-dev/aifa-admin-web-be:v20260519-aifa-rebuild-v2` |
| Client backend | `harbor.aifa.tpcloud/aifa-dev/aifa-client-backend:v20260519-aifa-rebuild-v2` |
| MCP backend | `harbor.aifa.tpcloud/aifa-dev/aifa-mcp-backend:v20260519-aifa-rebuild-v2` |
| Upload worker | `harbor.aifa.tpcloud/aifa-dev/aifa-process-file-upload:v20260519-aifa-rebuild-v2` |
| Run-process worker | `harbor.aifa.tpcloud/aifa-dev/aifa-run-process:v20260519-aifa-rebuild-v2` |

Các image trong package rebuild v2 AIFA gửi đã được xử lý hết:

| Image local từ tar | Mapping deploy |
|---|---|
| `aifa-admin-be:amd64` | `aifa-admin-web-be` |
| `aifa-dev-backend:amd64` | `aifa-client-backend` |
| `aifa-dev-mcp-backend:amd64` | `aifa-mcp-backend` |
| `aifa-dev-run-process-lambda:amd64` | `aifa-run-process` |
| `aifa-dev-upload-pipeline-lambda:amd64` | `aifa-process-file-upload` |

Không có frontend image/source trong package rebuild v2 hiện tại.

---

## 9. Data Services trên `aifa-data-01`

VM:

```text
aifa-data-01
10.80.30.11
```

Các dịch vụ data/runtime chính:

| Dịch vụ | Port | Vai trò |
|---|---:|---|
| PostgreSQL | `5432` | Database chính |
| Redis | `6379` | Cache, process status, client backend cache |
| RabbitMQ | `5672` | Message broker runtime |
| RabbitMQ Management | `15672` | Quản trị RabbitMQ |
| MinIO | `9000` | S3-compatible object storage |

Trong Kubernetes, các dịch vụ này được đại diện bằng Service nội bộ trong namespace `aifa-dev`:

| Kubernetes Service | ClusterIP | Port |
|---|---:|---|
| `aifa-postgresql` | `10.98.78.39` | `5432/TCP` |
| `aifa-redis` | `10.98.72.138` | `6379/TCP` |
| `aifa-rabbitmq` | `10.97.117.229` | `5672/TCP`, `15672/TCP` |
| `aifa-minio` | `10.101.136.206` | `9000/TCP` |

Ghi chú:

- Các Service trên có selector `<none>`, nghĩa là đang được dùng để đại diện cho data services bên ngoài Pod workload.
- Các ứng dụng trong Kubernetes gọi data services qua DNS nội bộ Kubernetes như `aifa-postgresql`, `aifa-redis`, `aifa-rabbitmq`, `aifa-minio`.

---

## 10. Kubernetes Deployments

### 10.1 `aifa-admin-web-be`

Mục đích:

- Admin Backend API.
- Health endpoint.
- OpenAPI endpoint.
- Phục vụ chức năng admin/backend của AIFA.

Thông tin triển khai:

| Thuộc tính | Giá trị |
|---|---|
| Deployment | `aifa-admin-web-be` |
| Namespace | `aifa-dev` |
| Replicas | `2` |
| Container port | `8000` |
| Image | `harbor.aifa.tpcloud/aifa-dev/aifa-admin-web-be:v20260519-aifa-rebuild-v2` |
| Service | `aifa-admin-web-be` |
| Service port | `8000/TCP` |
| Ingress host | `aifa-admin-dev.aifa.tpcloud` |
| Readiness probe | `/health` port `8000` |
| Liveness probe | `/health` port `8000` |
| PDB | `aifa-admin-web-be-pdb`, `minAvailable: 1` |

Health test:

```bash
curl -i -H "Host: aifa-admin-dev.aifa.tpcloud" http://10.80.10.20/health
```

Kết quả kỳ vọng:

```text
HTTP/1.1 200 OK
```

### 10.2 `aifa-client-backend`

Mục đích:

- Client Backend API.
- Backend cho frontend client nếu AIFA trỏ UI vào môi trường TPCOMS.
- Cung cấp API cho upload/process/chat/business flow tùy theo logic ứng dụng.

Thông tin triển khai:

| Thuộc tính | Giá trị |
|---|---|
| Deployment | `aifa-client-backend` |
| Namespace | `aifa-dev` |
| Replicas | `2` |
| Container port | `5000` |
| Image | `harbor.aifa.tpcloud/aifa-dev/aifa-client-backend:v20260519-aifa-rebuild-v2` |
| Service | `aifa-client-backend` |
| Service port | `5000/TCP` |
| Ingress host | `aifa-client-dev.aifa.tpcloud` |
| Readiness probe | `/health` port `5000` |
| Liveness probe | `/health` port `5000` |
| PDB | `aifa-client-backend-pdb`, `minAvailable: 1` |

Health test:

```bash
curl -i -H "Host: aifa-client-dev.aifa.tpcloud" http://10.80.10.20/health
```

Kết quả kỳ vọng:

```text
HTTP/1.1 200 OK
{"status": "healthy"}
```

Ghi chú kỹ thuật quan trọng:

- App yêu cầu Redis config key là `REDIS_CACHE`.
- `REDIS_CACHE` phải là Redis URL hợp lệ.
- Redis password có ký tự đặc biệt nên password đã được URL-encode trong `REDIS_CACHE`/`REDIS_URL`.

### 10.3 `aifa-mcp-backend`

Mục đích:

- MCP backend service.
- Cung cấp MCP tools qua SSE.
- Được `aifa-run-process-worker` và `aifa-client-backend` gọi thông qua MCP SSE URL.

Thông tin triển khai:

| Thuộc tính | Giá trị |
|---|---|
| Deployment | `aifa-mcp-backend` |
| Namespace | `aifa-dev` |
| Replicas | `1` |
| Container port | `8765` |
| Image | `harbor.aifa.tpcloud/aifa-dev/aifa-mcp-backend:v20260519-aifa-rebuild-v2` |
| Service | `aifa-mcp-backend` |
| Service port | `8765/TCP` |
| Internal MCP URL | `http://aifa-mcp-backend:8765/sse` |

MCP connectivity đã validate:

```text
DNS resolve OK
TCP connect OK
SSE endpoint HTTP 200
```

### 10.4 `aifa-process-file-upload-worker`

Mục đích:

- Consumer xử lý upload file.
- Nhận message từ RabbitMQ queue.
- Đọc file từ MinIO/S3 path.
- Xử lý nội dung file.
- Lưu metadata/file vào PostgreSQL/MinIO.
- Ghi dữ liệu NKC/structured XML nếu phù hợp.
- Có thể upload vector store tùy loại file.

Thông tin triển khai:

| Thuộc tính | Giá trị |
|---|---|
| Deployment | `aifa-process-file-upload-worker` |
| Namespace | `aifa-dev` |
| Replicas | `1` |
| Image | `harbor.aifa.tpcloud/aifa-dev/aifa-process-file-upload:v20260519-aifa-rebuild-v2` |
| RabbitMQ queue | `aifa-process-file-upload-queue-dev` |
| DLQ | `aifa-process-file-upload-queue-dev-dlq` |

Log khởi động kỳ vọng:

```text
Consuming RabbitMQ queue: aifa-process-file-upload-queue-dev
```

### 10.5 `aifa-run-process-worker`

Mục đích:

- Consumer xử lý run-process/report process.
- Nhận message từ RabbitMQ queue.
- Dùng Redis cache để quản lý trạng thái/progress.
- Gọi Vertex/LLM.
- Gọi MCP backend qua SSE.
- Lưu kết quả vào PostgreSQL.

Thông tin triển khai:

| Thuộc tính | Giá trị |
|---|---|
| Deployment | `aifa-run-process-worker` |
| Namespace | `aifa-dev` |
| Replicas | `1` |
| Image | `harbor.aifa.tpcloud/aifa-dev/aifa-run-process:v20260519-aifa-rebuild-v2` |
| RabbitMQ queue | `aifa-run-process-queue-dev` |
| DLQ | `aifa-run-process-queue-dev-dlq` |
| MCP SSE URL | `http://aifa-mcp-backend:8765/sse` |

Log khởi động kỳ vọng:

```text
Consuming RabbitMQ queue: aifa-run-process-queue-dev
```

---

## 11. Kubernetes Services

Danh sách Service chính:

| Service | Type | Port | Selector |
|---|---|---:|---|
| `aifa-admin-web-be` | ClusterIP | `8000/TCP` | `app=aifa-admin-web-be` |
| `aifa-client-backend` | ClusterIP | `5000/TCP` | `app=aifa-client-backend` |
| `aifa-mcp-backend` | ClusterIP | `8765/TCP` | `app=aifa-mcp-backend` |
| `aifa-minio` | ClusterIP | `9000/TCP` | `<none>` |
| `aifa-postgresql` | ClusterIP | `5432/TCP` | `<none>` |
| `aifa-rabbitmq` | ClusterIP | `5672/TCP`, `15672/TCP` | `<none>` |
| `aifa-redis` | ClusterIP | `6379/TCP` | `<none>` |

---

## 12. Kubernetes Ingress

Danh sách Ingress:

| Ingress | Host | Service | Port | Trạng thái |
|---|---|---|---:|---|
| `aifa-admin-web-be` | `aifa-admin-dev.aifa.tpcloud` | `aifa-admin-web-be` | `8000` | PASS |
| `aifa-client-backend` | `aifa-client-dev.aifa.tpcloud` | `aifa-client-backend` | `5000` | PASS |

Hiện tại Ingress đang phục vụ HTTP port `80`.

Để production hoặc public Internet chính thức, cần bổ sung:

- TLS certificate
- Ingress TLS secret
- Public DNS
- Firewall/NAT rule nếu expose ra Internet

---

## 13. ConfigMap `aifa-runtime-config`

ConfigMap chính:

```text
aifa-runtime-config
```

Vai trò:

- Chứa cấu hình runtime không nhạy cảm.
- Được mount vào các deployment thông qua `envFrom`.

Các nhóm cấu hình chính:

| Nhóm | Key tiêu biểu | Vai trò |
|---|---|---|
| Application | `APP_ENV`, `TZ` | Environment và timezone |
| Database | `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_NAME`, `DB_HOST`, `DB_PORT`, `DB_NAME` | Kết nối PostgreSQL |
| Redis | `REDIS_HOST`, `REDIS_PORT` | Redis service |
| RabbitMQ | `RABBITMQ_HOST`, `RABBITMQ_PORT`, `RABBITMQ_USER`, `RABBITMQ_VHOST` | RabbitMQ connection |
| S3/MinIO | `S3_ENDPOINT`, `S3_BUCKET`, `S3_FORCE_PATH_STYLE`, `S3_REGION` | S3-compatible object storage |
| MCP | `MCP_HOST`, `MCP_PORT`, `MCP_SERVER_URL`, `MCP_SSE_URL` | MCP backend/SSE |
| CORS | `CORS_ALLOWED_ORIGINS` | Origin được phép gọi API |

Giá trị MCP nội bộ đang dùng:

```text
MCP_SSE_URL=http://aifa-mcp-backend:8765/sse
MCP_SERVER_URL=http://aifa-mcp-backend:8765/sse
```

Các origin dev/UAT đã thêm:

```text
http://aifa-admin-dev.aifa.tpcloud
http://aifa-client-dev.aifa.tpcloud
http://app-dev.aifa.tpcloud
http://10.80.10.20
```

Khi AIFA có frontend domain thật, cần bổ sung domain đó vào `CORS_ALLOWED_ORIGINS`.

---

## 14. Secret `aifa-runtime-secret`

Secret chính:

```text
aifa-runtime-secret
```

Vai trò:

- Chứa credentials và runtime secret.
- Được mount vào các deployment thông qua `envFrom`.
- Không được gửi raw YAML ra ngoài vì có secret base64.

Các key quan trọng đang có:

| Key | Vai trò |
|---|---|
| `DATABASE_USER` | Database username |
| `DATABASE_PASSWORD` | Database password |
| `DATABASE_URL` | Database connection URL |
| `SQLALCHEMY_DATABASE_URI` | SQLAlchemy DB URI |
| `RABBITMQ_PASSWORD` | RabbitMQ application password |
| `REDIS_PASSWORD` | Redis password |
| `REDIS_URL` | Redis URL đã URL-encode password |
| `REDIS_CACHE` | Redis cache URL cho Flask client backend |
| `AWS_ACCESS_KEY_ID` | Access key dùng cho S3-compatible/MinIO hoặc AWS compatibility |
| `AWS_SECRET_ACCESS_KEY` | Secret key dùng cho S3-compatible/MinIO hoặc AWS compatibility |
| `VERTEXAI_3RD_SA_INFO_DICT` | Google/Vertex credential dạng service account info |
| `LLM_OPENAI_KEY` | OpenAI/LLM key nếu app dùng |
| `OPENAI_API_KEY` | OpenAI API key nếu app dùng |
| `LANGFUSE_PUBLIC_KEY` | Langfuse public key |
| `LANGFUSE_SECRET_KEY` | Langfuse secret key |
| `LANGFUSE_BASE_URL` | Langfuse endpoint |

Lưu ý bảo mật:

- Một số credential đã từng xuất hiện trong terminal/log trong quá trình debug.
- Sau khi AIFA xác nhận UAT thành công, nên rotate các credential nhạy cảm.

---

## 15. RabbitMQ

Vhost:

```text
aifa_dev
```

Các queue hiện có:

| Queue | Vai trò | Consumer |
|---|---|---:|
| `aifa-process-file-upload-queue-dev` | Queue upload file | `1` |
| `aifa-process-file-upload-queue-dev-dlq` | DLQ upload file | `0` |
| `aifa-run-process-queue-dev` | Queue run process | `1` |
| `aifa-run-process-queue-dev-dlq` | DLQ run process | `0` |
| `dev-audit-process-task-queue` | Audit/process queue phụ | `0` |
| `dev-audit-process-task-dlq` | DLQ audit/process phụ | `0` |

Trạng thái clean hiện tại:

```text
messages_ready = 0
messages_unacknowledged = 0
messages = 0
```

Command kiểm tra:

```bash
rabbitmqctl -p aifa_dev list_queues name messages_ready messages_unacknowledged messages consumers
```

---

## 16. MinIO

Endpoint nội bộ Kubernetes:

```text
http://aifa-minio:9000
```

Bucket dev chính:

```text
aifa-dev
```

Test objects đã upload vào:

```text
s3://aifa-dev/aifa-test-input/1-So-nhat-ky-chung.xlsx
s3://aifa-dev/aifa-test-input/2-Bang-can-doi-tai-khoan-mau-quan-tri.xlsx
s3://aifa-dev/aifa-test-input/3-360-BCTC-2024.pdf
s3://aifa-dev/aifa-test-input/360-GTGT-24Q1.xml
```

Files sau khi process được lưu vào path dạng:

```text
s3://aifa-dev/uploads/<user_id>/<project_id>/<filename>
```

Ví dụ đã validate:

```text
uploads/898a951c-80a1-70df-60aa-36bd794d467a/773/1-So-nhat-ky-chung.xlsx
```

---

## 17. PostgreSQL

Database chính:

```text
aifa_dev
```

Các bảng/schema liên quan đã validate:

| Object | Vai trò |
|---|---|
| `documents` | Metadata document upload |
| `private_project_773.nkc` | Dữ liệu NKC đã parse từ Excel |
| `structured_documents` | Structured data cho XML/tax forms |
| `report_process` | Trạng thái report/run process |
| `report_process_item` | Kết quả từng process item |
| `process_tasks` | Task tracking cho run-process |

Kết quả upload file đã validate:

| Document ID | File | Type | Project |
|---:|---|---|---:|
| `6412` | `1-So-nhat-ky-chung.xlsx` | `excel` | `773` |
| `6413` | `2-Bang-can-doi-tai-khoan-mau-quan-tri.xlsx` | `excel` | `773` |
| `6414` | `3-360-BCTC-2024.pdf` | `pdf` | `773` |
| `6415` | `360-GTGT-24Q1.xml` | `xml` | `773` |

NKC validation:

```text
private_project_773.nkc
document_id = 6412
rows = 5482
```

Run-process validation:

| Object | Value |
|---|---|
| `report_process.id` | `728` |
| `report_process.status` | `completed` |
| `report_process_item.id` | `1845` |
| `report_process_item.status` | `success` |
| `report_process_item.result_status` | `nothing` |
| `saved_prompt_title` | `TPCOMS on-prem run-process validation` |

Ghi chú:

- `result_status=nothing` không phải lỗi runtime.
- Đây là kết quả business của prompt test khá chung chung.
- Runtime đã PASS vì process completed, item success, queue ACK và DLQ sạch.

---

## 18. Redis

Redis được dùng cho:

- Runtime cache.
- Process status cache.
- Client backend cache.
- WebSocket/progress status nếu app đi qua UI thật.

Service nội bộ:

```text
aifa-redis:6379
```

Các cấu hình Redis quan trọng:

| Key | Vai trò |
|---|---|
| `REDIS_HOST` | Host Redis |
| `REDIS_PORT` | Port Redis |
| `REDIS_PASSWORD` | Password Redis |
| `REDIS_URL` | Redis URL |
| `REDIS_CACHE` | Redis URL mà Flask client backend thật sự đọc |

Lưu ý quan trọng:

- Client backend không đọc `REDIS_URL` trực tiếp cho phần Redis cache.
- Code đọc `REDIS_CACHE`.
- Password Redis có ký tự đặc biệt nên URL phải được URL-encode.

---

## 19. MCP / Agent / LLM

MCP backend:

```text
aifa-mcp-backend
http://aifa-mcp-backend:8765/sse
```

Đã validate:

- DNS resolve service thành công.
- TCP connect port `8765` thành công.
- SSE endpoint trả `HTTP 200`.
- Run-process worker và client backend đã load MCP tools thành công.
- Log ghi nhận MCP initialization với 4 tools.

LLM/Vertex:

- Runtime đã nhận `VERTEXAI_3RD_SA_INFO_DICT`.
- Run-process đã vượt qua bước Vertex credential/init.
- Agent đã đi qua MCP service và ghi kết quả DB.

Lưu ý cần AIFA xác nhận thêm:

- Trong quá trình test, từng thấy log liên quan Vanna/ChromaDB có dấu hiệu phụ thuộc S3/AWS cũ như `InvalidAccessKeyId`.
- Vấn đề này không làm runtime fail nhưng có thể ảnh hưởng chất lượng query/knowledge base.
- AIFA cần xác nhận cấu hình Vanna/ChromaDB/knowledge base khi chuyển từ AWS sang on-prem/MinIO.

---

## 20. Validation upload_file

### 20.1 File NKC Excel

Input:

```text
s3://aifa-dev/aifa-test-input/1-So-nhat-ky-chung.xlsx
```

Kết quả:

```text
document_id = 6412
summary = NKC: 1 tables, 5482 rows uploaded
private_project_773.nkc rows = 5482
```

### 20.2 File CDTK Excel

Input:

```text
s3://aifa-dev/aifa-test-input/2-Bang-can-doi-tai-khoan-mau-quan-tri.xlsx
```

Kết quả:

```text
document_id = 6413
status = pending
vector_ids present
summary generated
```

### 20.3 File PDF BCTC

Input:

```text
s3://aifa-dev/aifa-test-input/3-360-BCTC-2024.pdf
```

Kết quả:

```text
document_id = 6414
status = pending
vector_ids present
summary generated
```

### 20.4 File XML GTGT

Input:

```text
s3://aifa-dev/aifa-test-input/360-GTGT-24Q1.xml
```

Kết quả:

```text
document_id = 6415
status = pending
structured data saved
```

### 20.5 RabbitMQ sau upload validation

Kỳ vọng:

```text
aifa-process-file-upload-queue-dev      0 ready / 0 unack / 0 total / 1 consumer
aifa-process-file-upload-queue-dev-dlq  0 ready / 0 unack / 0 total / 0 consumer
```

---

## 21. Validation run-process

Run-process validation dùng:

```text
user_id = 898a951c-80a1-70df-60aa-36bd794d467a
project_id = 792
client_id = 801
process_id = 73
report_process_id = 728
```

Kết quả:

```text
report_process.id = 728
report_process.status = completed

report_process_item.id = 1845
report_process_item.status = success
report_process_item.result_status = nothing
```

RabbitMQ sau run-process validation:

```text
aifa-run-process-queue-dev      0 ready / 0 unack / 0 total / 1 consumer
aifa-run-process-queue-dev-dlq  0 ready / 0 unack / 0 total / 0 consumer
```

---

## 22. Endpoint hiện tại cho AIFA test

### 22.1 Admin Backend API

```text
http://aifa-admin-dev.aifa.tpcloud
```

Health:

```bash
curl -i http://aifa-admin-dev.aifa.tpcloud/health
```

Hoặc qua LB với Host header:

```bash
curl -i -H "Host: aifa-admin-dev.aifa.tpcloud" http://10.80.10.20/health
```

Kỳ vọng:

```text
HTTP/1.1 200 OK
```

### 22.2 Client Backend API

```text
http://aifa-client-dev.aifa.tpcloud
```

Health:

```bash
curl -i http://aifa-client-dev.aifa.tpcloud/health
```

Hoặc qua LB với Host header:

```bash
curl -i -H "Host: aifa-client-dev.aifa.tpcloud" http://10.80.10.20/health
```

Kỳ vọng:

```text
HTTP/1.1 200 OK
{"status": "healthy"}
```

### 22.3 MCP backend

MCP backend là service nội bộ, không public trực tiếp:

```text
http://aifa-mcp-backend:8765/sse
```

Ứng dụng nội bộ gọi MCP qua Kubernetes DNS.

---

## 23. Frontend UI và public application

Hiện tại chưa có frontend UI artifact trong package AIFA gửi.

Không tìm thấy image hoặc tar dạng:

```text
frontend
ui
client-web
admin-fe
web-fe
aifa-frontend
```

Vì vậy hiện trạng là:

```text
Backend/API/Worker/Data runtime: PASS
Public API qua Ingress/LB: PASS
End-user frontend UI: Chưa có artifact để deploy
```

Để AIFA test end-to-end qua UI, cần một trong hai phương án:

### Phương án A — AIFA dùng frontend hiện hữu

AIFA cấu hình frontend hiện hữu trỏ về:

```text
ADMIN_API_URL=http://aifa-admin-dev.aifa.tpcloud
CLIENT_API_URL=http://aifa-client-dev.aifa.tpcloud
API_BASE_URL=http://aifa-client-dev.aifa.tpcloud
```

Sau đó AIFA test login/upload/run-process từ UI hiện hữu.

### Phương án B — AIFA gửi frontend artifact

AIFA cần gửi:

- Frontend Docker image hoặc tar image.
- Hoặc source code frontend + hướng dẫn build.
- File `.env` mẫu.
- Các biến:
  - `API_BASE_URL`
  - `ADMIN_API_URL`
  - `CLIENT_API_URL`
  - `COGNITO_*`
  - `AUTH_*`
  - WebSocket endpoint nếu có
- Domain mong muốn cho UI:
  - Ví dụ `app-dev.aifa.tpcloud`
  - Hoặc `admin-ui-dev.aifa.tpcloud`
  - Hoặc domain public khác

Khi có frontend artifact, TPCOMS sẽ deploy thêm:

```text
aifa-client-frontend
aifa-admin-frontend nếu có
```

và public qua Ingress.

---

## 24. UAT / End-to-End Test Checklist cho AIFA

### 24.1 Chuẩn bị

AIFA cần xác nhận:

- Frontend đang dùng endpoint API nào.
- Frontend có gọi đúng:
  - `http://aifa-client-dev.aifa.tpcloud`
  - `http://aifa-admin-dev.aifa.tpcloud`
- DNS từ máy test resolve được các domain trên.
- Nếu test từ ngoài TPCOMS, cần xác nhận VPN/public NAT.

### 24.2 Health test

Kiểm tra:

```bash
curl -i http://aifa-admin-dev.aifa.tpcloud/health
curl -i http://aifa-client-dev.aifa.tpcloud/health
```

Kỳ vọng:

```text
HTTP/1.1 200 OK
```

### 24.3 Login/Auth

AIFA kiểm tra:

- Đăng nhập bằng user dev.
- Token/session được tạo.
- Permission theo user/project/client đúng.
- Nếu dùng Cognito/OAuth, callback/redirect URL phải bao gồm domain UAT mới.

### 24.4 Upload file

Test từ UI hoặc API:

- Upload NKC Excel.
- Upload CDTK Excel.
- Upload PDF BCTC.
- Upload XML GTGT.

Kỳ vọng:

- File vào MinIO.
- Document record được tạo trong PostgreSQL.
- Worker nhận message từ RabbitMQ.
- Queue chính không tồn message kẹt.
- DLQ sạch.
- Với NKC, dữ liệu được ghi vào schema project tương ứng.
- Với XML, structured data được ghi vào bảng phù hợp.

### 24.5 Run process

Test từ UI hoặc API:

- Chạy một report/process hợp lệ.
- Kiểm tra progress/status.
- Kiểm tra `report_process` chuyển completed.
- Kiểm tra `report_process_item` được tạo.
- Kiểm tra RabbitMQ run-process queue sạch.
- Kiểm tra DLQ sạch.
- Kiểm tra MCP backend có log `ListToolsRequest`/`CallToolRequest`.

### 24.6 Kiểm tra MinIO

AIFA/TPCOMS kiểm tra:

- Object input tồn tại.
- Object upload/output được ghi đúng bucket/path.
- S3 path-style access hoạt động.
- Bucket `aifa-dev` dùng đúng.

### 24.7 Kiểm tra RabbitMQ

TPCOMS kiểm tra:

```bash
rabbitmqctl -p aifa_dev list_queues name messages_ready messages_unacknowledged messages consumers
```

Kỳ vọng:

```text
aifa-process-file-upload-queue-dev      0 0 0 1
aifa-process-file-upload-queue-dev-dlq  0 0 0 0
aifa-run-process-queue-dev              0 0 0 1
aifa-run-process-queue-dev-dlq          0 0 0 0
```

### 24.8 Kiểm tra PostgreSQL

TPCOMS/AIFA kiểm tra:

- `documents`
- `private_project_<project_id>.nkc`
- `structured_documents`
- `report_process`
- `report_process_item`
- `process_tasks`

### 24.9 Kiểm tra Redis

Kiểm tra nếu có lỗi progress/WebSocket/cache:

- Redis reachable.
- `REDIS_CACHE` đúng format.
- Redis password đã URL-encode trong Redis URL.
- Process status cache có dữ liệu khi chạy process thật.

### 24.10 Kiểm tra CORS

Từ frontend domain, kiểm tra browser không bị CORS error.

Nếu frontend domain khác các domain hiện có, cần bổ sung vào:

```text
CORS_ALLOWED_ORIGINS
```

---

## 25. Health-check scripts

### 25.1 Script trên `aifa-k8s-cp-01`

Path:

```text
/root/aifa-dev-health-check.sh
```

Chức năng:

- Kiểm tra deployments.
- Kiểm tra pods.
- Kiểm tra services.
- Kiểm tra ingress.
- Kiểm tra admin/client backend health qua LB.

Chạy:

```bash
/root/aifa-dev-health-check.sh
```

### 25.2 Script đề xuất trên `aifa-data-01`

Path:

```text
/root/aifa-dev-data-check.sh
```

Chức năng:

- Kiểm tra RabbitMQ queues.
- Kiểm tra 4 document test upload.
- Kiểm tra NKC rows.
- Kiểm tra run-process validation.

Chạy:

```bash
/root/aifa-dev-data-check.sh
```

---

## 26. Evidence / backup paths

### 26.1 Control-plane evidence

Final runtime pass:

```text
/root/aifa-backup/final-aifa-dev-runtime-pass-20260519-225234
```

UAT-ready evidence:

```text
/root/aifa-backup/aifa-dev-uat-ready-20260519-230527
```

Client backend pass evidence:

```text
/root/aifa-backup/post-client-backend-v2-pass-20260519-224944
```

Admin backend pass evidence:

```text
/root/aifa-backup/post-admin-web-be-v2-pass-20260519-223147
```

### 26.2 Data node evidence

Final data/runtime pass:

```text
/root/aifa-runtime-check/final-aifa-dev-runtime-pass-20260519-225241
```

### 26.3 Lưu ý bảo mật evidence

Các folder evidence có thể chứa:

- ConfigMap YAML.
- Secret YAML base64.
- Logs.
- Runtime evidence.

Không gửi nguyên folder ra ngoài. Nếu cần gửi AIFA, nên tạo bản mask/redact secret.

---

## 27. Các vấn đề đã xử lý trong quá trình migration

| Vấn đề | Cách xử lý |
|---|---|
| Image architecture/rebuild | AIFA rebuild image amd64, TPCOMS load/tag/push Harbor |
| Harbor trust | Kubernetes nodes trust Harbor CA |
| Thiếu `VERTEXAI_3RD_SA_INFO_DICT` | Patch runtime secret và restart worker |
| Thiếu MCP service | Deploy `aifa-mcp-backend`, patch `MCP_SSE_URL` |
| Client backend thiếu `REDIS_CACHE` | Patch `REDIS_CACHE` vào secret |
| Redis password có ký tự đặc biệt | URL-encode password trong Redis URL |
| CORS local/default | Patch `CORS_ALLOWED_ORIGINS` |
| Backend chưa có probes | Thêm readiness/liveness probes |
| Backend chưa có PDB | Thêm PDB cho 2 backend chính |

---

## 28. Các điểm còn cần AIFA xác nhận

### 28.1 Frontend UI

Hiện chưa có frontend artifact.

AIFA cần xác nhận:

- Dùng frontend hiện hữu hay gửi frontend image/source.
- Frontend domain.
- API base URL.
- Auth callback/redirect URL.
- WebSocket endpoint nếu có.

### 28.2 Auth/Cognito/OAuth

Nếu login phụ thuộc Cognito/OAuth:

- Cần bổ sung callback URL mới.
- Cần kiểm tra allowed origins.
- Cần kiểm tra token audience/issuer.
- Cần xác nhận environment dev/staging/prod.

### 28.3 Vanna/ChromaDB/Knowledge Base

Trong quá trình test từng thấy dấu hiệu cấu hình còn phụ thuộc AWS/S3 cũ.

Cần AIFA xác nhận:

- Vanna/ChromaDB có cần migrate data không.
- Knowledge base hiện nằm ở đâu.
- Có cần đồng bộ từ AWS S3 sang MinIO không.
- Access key hiện tại dùng cho AWS hay MinIO.
- Tool query dữ liệu nghiệp vụ có cần training/re-index lại không.

### 28.4 Public Internet / TLS

Hiện API đang sẵn sàng qua LB nội bộ.

Để public Internet chính thức cần:

- Public DNS.
- NAT/firewall.
- TLS certificate.
- Ingress TLS.
- Security review.

### 28.5 Secret rotation

Sau UAT cần rotate các credential nhạy cảm đã từng dùng/debug.

---

## 29. Production hardening đề xuất

Trạng thái hiện tại là UAT-ready. Để production-ready cần thêm:

### 29.1 Bảo mật

- Rotate toàn bộ secret.
- Dùng robot account Harbor thay vì admin.
- Không lưu secret raw trong repo/folder bàn giao.
- Tắt log in credential.
- Hạn chế quyền SSH.
- NetworkPolicy cho Kubernetes nếu cần.
- TLS end-to-end cho public endpoint.

### 29.2 Backup

- PostgreSQL backup định kỳ.
- MinIO bucket backup/replication.
- RabbitMQ definitions export.
- Harbor backup.
- Kubernetes manifest backup.
- Snapshot VM milestone.

### 29.3 Monitoring

- Pod health.
- Deployment availability.
- RabbitMQ queue depth.
- DLQ alert.
- PostgreSQL availability.
- Redis availability.
- MinIO availability.
- Ingress 5xx alert.
- Worker log error alert.

### 29.4 CI/CD

- Build image.
- Push Harbor.
- Deploy Kubernetes.
- Rollback procedure.
- Environment promotion dev -> staging -> prod.

### 29.5 Scalability

- HPA cho backend nếu workload đủ.
- Tăng replicas cho worker nếu queue lớn.
- Separate data services HA nếu production yêu cầu HA.
- PostgreSQL HA/backup/restore plan.
- Redis HA nếu cần.
- RabbitMQ HA nếu cần.
- MinIO distributed mode nếu cần.

---

## 30. Đề xuất quy trình AIFA test end-to-end

### Bước 1 — Xác nhận API backend

AIFA hoặc TPCOMS chạy:

```bash
curl -i http://aifa-admin-dev.aifa.tpcloud/health
curl -i http://aifa-client-dev.aifa.tpcloud/health
```

### Bước 2 — Kết nối frontend

AIFA chọn một trong hai:

- Trỏ frontend hiện hữu sang API TPCOMS.
- Gửi frontend artifact để TPCOMS deploy.

### Bước 3 — Login

- Kiểm tra đăng nhập.
- Kiểm tra token.
- Kiểm tra quyền project/client.

### Bước 4 — Upload file

Test 4 loại:

- NKC Excel
- CDTK Excel
- PDF
- XML

### Bước 5 — Kiểm tra xử lý phía backend

TPCOMS theo dõi:

```bash
kubectl -n aifa-dev logs deploy/aifa-process-file-upload-worker -f
rabbitmqctl -p aifa_dev list_queues name messages_ready messages_unacknowledged messages consumers
```

### Bước 6 — Run process

AIFA chạy process/report thật từ UI.

TPCOMS theo dõi:

```bash
kubectl -n aifa-dev logs deploy/aifa-run-process-worker -f
kubectl -n aifa-dev logs deploy/aifa-mcp-backend -f
rabbitmqctl -p aifa_dev list_queues name messages_ready messages_unacknowledged messages consumers
```

### Bước 7 — Ghi nhận kết quả

AIFA xác nhận:

- Kết quả UI.
- Kết quả report.
- Dữ liệu đúng nghiệp vụ.
- Không có queue kẹt.
- Không có DLQ.
- Không có lỗi browser console/CORS.
- Không có lỗi auth/token.

---

## 31. Trạng thái bàn giao UAT

Tại thời điểm lập tài liệu, môi trường đạt trạng thái:

```text
AIFA Dev Backend/API/Worker/Data Runtime: UAT-ready
```

Điều kiện đã đạt:

- Kubernetes workload chính Running.
- Restart count bằng 0.
- Backend health `200 OK`.
- Worker consumers hoạt động.
- RabbitMQ queue/DLQ sạch.
- Upload file validation PASS.
- Run-process validation PASS.
- MCP backend hoạt động.
- Data services hoạt động.
- Probes và PDB đã bổ sung.
- CORS đã được chuẩn hóa cho domain dev/UAT hiện tại.

Điều kiện còn thiếu để user test qua UI hoàn chỉnh:

```text
Frontend UI artifact hoặc frontend hiện hữu trỏ API sang TPCOMS.
```

---

## 32. Phụ lục command vận hành nhanh

### 32.1 Kiểm tra toàn bộ Kubernetes app

```bash
kubectl -n aifa-dev get deploy,svc,pod,ingress,pdb -o wide
```

### 32.2 Kiểm tra pod restart

```bash
kubectl -n aifa-dev get pod \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,RESTARTS:.status.containerStatuses[*].restartCount,STATUS:.status.phase,NODE:.spec.nodeName'
```

### 32.3 Kiểm tra image đang chạy

```bash
kubectl -n aifa-dev get deploy \
  -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{range .spec.template.spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
```

### 32.4 Kiểm tra health backend

```bash
curl -i -H "Host: aifa-admin-dev.aifa.tpcloud" http://10.80.10.20/health

curl -i -H "Host: aifa-client-dev.aifa.tpcloud" http://10.80.10.20/health
```

### 32.5 Kiểm tra RabbitMQ

```bash
rabbitmqctl -p aifa_dev list_queues name messages_ready messages_unacknowledged messages consumers
```

### 32.6 Xem log admin backend

```bash
kubectl -n aifa-dev logs deploy/aifa-admin-web-be --tail=200
```

### 32.7 Xem log client backend

```bash
kubectl -n aifa-dev logs deploy/aifa-client-backend --tail=200
```

### 32.8 Xem log upload worker

```bash
kubectl -n aifa-dev logs deploy/aifa-process-file-upload-worker --tail=200
```

### 32.9 Xem log run-process worker

```bash
kubectl -n aifa-dev logs deploy/aifa-run-process-worker --tail=200
```

### 32.10 Xem log MCP backend

```bash
kubectl -n aifa-dev logs deploy/aifa-mcp-backend --tail=200
```

### 32.11 Restart backend sau khi đổi ConfigMap/Secret

```bash
kubectl -n aifa-dev rollout restart deploy/aifa-admin-web-be
kubectl -n aifa-dev rollout restart deploy/aifa-client-backend
kubectl -n aifa-dev rollout status deploy/aifa-admin-web-be --timeout=180s
kubectl -n aifa-dev rollout status deploy/aifa-client-backend --timeout=180s
```

### 32.12 Rollback deployment nếu rollout lỗi

```bash
kubectl -n aifa-dev rollout undo deploy/aifa-admin-web-be

kubectl -n aifa-dev rollout undo deploy/aifa-client-backend
```

---

## 33. Kết luận

Môi trường AIFA Dev trên TPCOMS hiện đã hoàn thành phần nền tảng backend/runtime để AIFA bắt đầu test end-to-end.

TPCOMS đã hoàn tất:

- Hạ tầng VM.
- Kubernetes runtime.
- Harbor registry.
- Data services.
- Backend API.
- Worker upload file.
- Worker run-process.
- MCP backend.
- Ingress/LB API exposure.
- Runtime config cần thiết.
- Validation upload/process.
- Cleanup môi trường.
- Probe/PDB/CORS.
- Evidence UAT-ready.

AIFA cần tiếp tục:

- Cung cấp frontend artifact hoặc cấu hình frontend hiện hữu trỏ API về TPCOMS.
- Test UI/nghiệp vụ thật.
- Xác nhận Auth/Cognito/OAuth.
- Xác nhận Vanna/ChromaDB/knowledge base.
- Xác nhận yêu cầu public domain/TLS nếu cần expose ngoài Internet.

