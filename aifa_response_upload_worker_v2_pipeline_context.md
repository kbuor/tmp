## Kết quả tổng quan

Các phần migration/runtime đã PASS:

- Worker Kubernetes-native đã chạy ổn trên namespace `aifa-dev`.
- Worker consume RabbitMQ queue OK.
- RabbitMQ queue/DLQ hoạt động đúng.
- Payload schema mới dạng `{"task":"process_file_upload","kwargs":{...}}` đã được worker nhận.
- Các runtime env cần thiết đã được bổ sung:
  - `USE_NEW_PIPELINE=true`
  - `LLM_OPENAI_KEY`
  - `OPENAI_API_KEY`
  - `LANGFUSE_PUBLIC_KEY`
  - `LANGFUSE_SECRET_KEY`
  - `LANGFUSE_BASE_URL`
  - `VERTEXAI_3RD_SA_INFO_DICT`
- MinIO/S3 patch đã hoạt động OK:
  - `bucket_name=aifa-dev`
  - `endpoint_url=http://aifa-minio:9000`
  - Extract key OK
  - Download file test từ MinIO OK: `445662 bytes`

Hiện tại flow đã đi được đến business pipeline thật, cụ thể là:

```text
RabbitMQ -> worker_entrypoint.py -> lambda_function.py -> LambdaContainer -> MinIO download -> ExcelUploadPipeline
```

Tuy nhiên flow đang dừng ở lỗi source code trong Excel pipeline:

```text
Excel pipeline failed: 'PipelineContext' object has no attribute 'classified_excel_type'
```

---

# Payload test đã publish

```json
{
  "task": "process_file_upload",
  "kwargs": {
    "file_path": "s3://aifa-dev/aifa-test-input/1-So-nhat-ky-chung.xlsx",
    "user_id": "phase11-test-user",
    "project_id": 823,
    "client_id": 782,
    "file_type": "nkc",
    "file_name": "1-So-nhat-ky-chung.xlsx",
    "connection_id": "phase11-test-connection",
    "message_id": "phase11-test-nkc-002"
  }
}
```

---

# Evidence 1 - MinIO/S3 verification PASS

Command kiểm tra trực tiếp trong pod `aifa-process-file-upload-worker`:

```bash
kubectl -n aifa-dev exec -i "$UPLOAD_POD" -- python - <<'PY'
import io
from app.infrastructure.serverless import LambdaContainer

url = "s3://aifa-dev/aifa-test-input/1-So-nhat-ky-chung.xlsx"

container = LambdaContainer()
storage = container.providers.storage

print("storage_class=", type(storage).__name__)
print("bucket_name=", getattr(storage, "_bucket_name", None))
print("endpoint_url=", getattr(storage, "_endpoint_url", None))

key = storage.extract_key_from_url(url)
print("extracted_key=", key)

buffer = io.BytesIO()
storage.download(key, buffer)
print("downloaded_bytes=", len(buffer.getvalue()))
PY
```

Output:

```text
storage_class= S3StorageAdapter
bucket_name= aifa-dev
endpoint_url= http://aifa-minio:9000
extracted_key= aifa-test-input/1-So-nhat-ky-chung.xlsx
downloaded_bytes= 445662
```

Kết luận:

- Worker đọc được MinIO endpoint nội bộ.
- Worker xác định đúng bucket `aifa-dev`.
- Worker parse đúng key từ URL dạng `s3://...`.
- Worker download được file NKC test từ MinIO.

---

# Evidence 2 - Worker nhận payload và đi vào Excel pipeline

Sau khi publish payload vào queue `aifa-process-file-upload-queue-dev`, worker xử lý và phát sinh lỗi sau:

```text
API Gateway client not initialized, message not sent
Excel pipeline failed: 'PipelineContext' object has no attribute 'classified_excel_type'
[local] Pipeline execution error: 'PipelineContext' object has no attribute 'classified_excel_type'
Traceback (most recent call last):
  File "/app/lambda_function.py", line 219, in lambda_handler
    result_context = pipeline.execute(context_data)
                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/app/data_pipeline_implementations/pipelines/excel_upload_pipeline.py", line 160, in execute
    f"Step ClassifyExcelTypeStep completed - classified as: {context.classified_excel_type}"
                                                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
AttributeError: 'PipelineContext' object has no attribute 'classified_excel_type'
[PIPELINE ERROR] 'PipelineContext' object has no attribute 'classified_excel_type'
Stack trace: Traceback (most recent call last):
  File "/app/lambda_function.py", line 219, in lambda_handler
    result_context = pipeline.execute(context_data)
                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/app/data_pipeline_implementations/pipelines/excel_upload_pipeline.py", line 160, in execute
    f"Step ClassifyExcelTypeStep completed - classified as: {context.classified_excel_type}"
                                                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
AttributeError: 'PipelineContext' object has no attribute 'classified_excel_type'

API Gateway client not initialized, message not sent
{"statusCode": 500, "errorMessage": "'PipelineContext' object has no attribute 'classified_excel_type'", "errorType": "AttributeError", "stackTrace": "Traceback (most recent call last):\n  File \"/app/lambda_function.py\", line 219, in lambda_handler\n    result_context = pipeline.execute(context_data)\n                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^\n  File \"/app/app/data_pipeline_implementations/pipelines/excel_upload_pipeline.py\", line 160, in execute\n    f\"Step ClassifyExcelTypeStep completed - classified as: {context.classified_excel_type}\"\n                                                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^\nAttributeError: 'PipelineContext' object has no attribute 'classified_excel_type'\n"}
```

Ghi chú:

- Dòng `API Gateway client not initialized, message not sent` hiện đang được xem là warning vì test payload dùng `connection_id` giả.
- Lỗi chính làm pipeline fail là `AttributeError: 'PipelineContext' object has no attribute 'classified_excel_type'`.

---

# Evidence 3 - Kiểm tra `PipelineContext`

Command:

```bash
kubectl -n aifa-dev exec "$UPLOAD_POD" -- sh -c '
echo "===== PipelineContext fields quick check ====="
python - <<PY
from app.data_pipeline_interfaces.pipeline_interface import PipelineContext
print(PipelineContext.__dataclass_fields__.keys())
PY
'
```

Output:

```text
===== PipelineContext fields quick check =====
dict_keys(['file_path', 'file_type', 'user_id', 'project_id', 'file_name', 'file_bytes', 'client_id', 'message_id', 'content', 'search_metadata', 'file_metadata', 'storage_url', 'vector_ids', 'document_id', 'summary', 'start_time', 'end_time', 'detected_file_type', 'structured_data', 'structured_document_id', 'metadata', 'results', 'errors', 'warnings', 'step_timings', 'status'])
```

Nhận xét:

`PipelineContext` hiện chưa khai báo các field sau:

```text
classified_excel_type
classification_confidence
classification_reason
original_file_type
```

---

# Evidence 4 - Đoạn code `excel_upload_pipeline.py` tại vị trí lỗi

Command:

```bash
kubectl -n aifa-dev exec "$UPLOAD_POD" -- sh -c '
echo "===== Excel upload pipeline around failing line ====="
sed -n "120,190p" /app/app/data_pipeline_implementations/pipelines/excel_upload_pipeline.py
'
```

Relevant output:

```python
# Phase 2: Classify type
classify_step = ClassifyExcelTypeStep()
logger.info("Executing step: ClassifyExcelTypeStep")
classify_step.execute(context)
executed_steps.append(classify_step)
logger.info(
    f"Step ClassifyExcelTypeStep completed - classified as: {context.classified_excel_type}"
)

# Phase 2.5: Validate classified type
# Chỉ chấp nhận NKC (Sổ nhật ký chung) và BCTC (Báo cáo tài chính)
classified_type = getattr(context, "classified_excel_type", "excel")
if classified_type not in ["nkc", "bctc", "cdps"]:
    raise ValueError(
        f"File Excel '{context.file_name}' không được hỗ trợ. "
    )

# Phase 3: Route based on classified type
if context.classified_excel_type in ["nkc", "excel_nkc"]:
    # NKC FLOW: Independent steps, no vector store
    logger.info("Routing to NKC flow (new independent steps)")
    remaining_steps = [
        NKCReadExcelStep(),
        NKCDetectTablesStep(),
        NKCMapHeadersStep(),
        NKCProcessDataStep(),
        NKCUploadToDatabaseStep(),
        UploadToStorageStep(storage_provider=self._storage_provider),
    ]
```

Nhận xét:

- Code đã có fallback ở đoạn `classified_type = getattr(...)`.
- Nhưng trước đó vẫn truy cập trực tiếp `context.classified_excel_type` trong `logger.info(...)`.
- Sau đó tiếp tục truy cập trực tiếp `context.classified_excel_type` tại điều kiện route NKC.

---

# Evidence 5 - Kiểm tra `ClassifyExcelTypeStep`

Command:

```bash
kubectl -n aifa-dev exec "$UPLOAD_POD" -- sh -c '
echo "===== classify_excel_type_step.py ====="
sed -n "1,220p" /app/app/data_pipeline_implementations/steps/classify_excel_type_step.py
'
```

Relevant output:

```python
def execute(self, context: PipelineContext) -> PipelineContext:
    """
    Classify Excel file type using LLM with structured output.

    Only runs if file_type is 'excel' (generic).
    Updates context.file_type to classified type.

    Args:
        context: Pipeline context with content

    Returns:
        Updated context with classified file_type
    """
    start_time = time.time()

    try:
        # Skip if not generic excel
        if context.file_type not in ["excel"]:
            logger.info(
                f"Skipping classification - file_type already specific: {context.file_type}"
            )
            return context

        # Skip if no content
        if not context.content or not context.content.strip():
            logger.warning("No content to classify - keeping generic excel type")
            return context

        # Get sample content for classification
        sample = context.content[: self.SAMPLE_SIZE]

        # Call LLM with structured output
        logger.info(
            "Classifying Excel file type using LangChain structured output..."
        )

        result = self.llm_provider.generate_structured(
            prompt=self.USER_PROMPT.format(content=sample),
            response_model=ExcelClassificationResult,
            system_prompt=self.SYSTEM_PROMPT,
        )

        # Store classification result
        context.original_file_type = context.file_type
        context.classified_excel_type = result.type
        context.classification_confidence = result.confidence
        context.classification_reason = result.reason

        # Map to detected_file_type for consistency with other pipelines
        if result.type == "nkc":
            context.detected_file_type = "nkc"
        elif result.type == "bctc":
            context.detected_file_type = "bctc"
        else:
            context.detected_file_type = "other"

        logger.info(
            f"Classified Excel type: {result.type} "
            f"(confidence: {result.confidence:.2f}, reason: {result.reason})"
        )

    except Exception as e:
        logger.error(f"Failed to classify Excel type: {str(e)}")
        context.warnings.append(f"ClassifyExcelTypeStep: {str(e)}")
        # Fallback to basic classification
        context.classified_excel_type = self._fallback_classification(
            context.content
        )
    finally:
        context.step_timings["ClassifyExcelTypeStep"] = time.time() - start_time

    return context
```

Nhận xét:

Payload test hiện đang truyền:

```json
"file_type": "nkc"
```

Do đó khi vào `ClassifyExcelTypeStep`, điều kiện sau sẽ đúng:

```python
if context.file_type not in ["excel"]:
    logger.info(
        f"Skipping classification - file_type already specific: {context.file_type}"
    )
    return context
```

Vì step return sớm nên không set:

```python
context.classified_excel_type
context.classification_confidence
context.classification_reason
context.original_file_type
```

Sau đó `excel_upload_pipeline.py` lại truy cập trực tiếp:

```python
context.classified_excel_type
```

nên phát sinh `AttributeError`.

---

# Root cause hiện tại

Payload test truyền `file_type="nkc"`.

`ClassifyExcelTypeStep` chỉ classify khi `file_type == "excel"`. Nếu `file_type` đã là `nkc`, step sẽ skip và return ngay.

Tuy nhiên `excel_upload_pipeline.py` lại mặc định rằng `context.classified_excel_type` luôn tồn tại sau `classify_step.execute(context)`, nên khi step bị skip, attribute này không tồn tại và pipeline crash.

Đây là lỗi logic/source code trong image upload worker v2, không còn là lỗi hạ tầng migration.

---

# Đề xuất fix

## Phương án 1 - Khuyến nghị

Thêm field mặc định vào `PipelineContext`:

```python
classified_excel_type: str = ""
classification_confidence: float = 0.0
classification_reason: str = ""
original_file_type: str = ""
```

Trong `ClassifyExcelTypeStep`, khi skip vì `file_type` đã specific, vẫn nên set `classified_excel_type`:

```python
if context.file_type not in ["excel"]:
    logger.info(
        f"Skipping classification - file_type already specific: {context.file_type}"
    )
    context.original_file_type = context.file_type
    context.classified_excel_type = context.file_type
    context.classification_confidence = 1.0
    context.classification_reason = "File type was provided by caller"
    if context.file_type in ["nkc", "bctc", "cdps"]:
        context.detected_file_type = context.file_type
    return context
```

Trong `excel_upload_pipeline.py`, không nên truy cập trực tiếp `context.classified_excel_type` khi chưa đảm bảo field tồn tại. Nên dùng biến fallback:

```python
classified_type = getattr(context, "classified_excel_type", context.file_type or "excel")

logger.info(
    f"Step ClassifyExcelTypeStep completed - classified as: {classified_type}"
)

if classified_type not in ["nkc", "bctc", "cdps"]:
    raise ValueError(
        f"File Excel '{context.file_name}' không được hỗ trợ. "
    )

if classified_type in ["nkc", "excel_nkc"]:
    ...
```

## Phương án 2 - Workaround phía caller

Nếu logic hiện tại bắt buộc `ClassifyExcelTypeStep` phải chạy, backend/publisher có thể luôn gửi:

```json
"file_type": "excel"
```

để worker tự classify lại thành `nkc`, `bctc`, `cdps` hoặc `excel`.

Tuy nhiên bên em vẫn khuyến nghị fix code theo hướng defensive ở phương án 1 để tránh lỗi tương tự khi caller đã xác định sẵn `file_type`.

---

# Kết luận

Hiện tại bên mình đã xác nhận các phần hạ tầng/migration đã đi qua:

```text
Kubernetes worker image v2: OK
RabbitMQ consumer: OK
RabbitMQ DLQ: OK
Payload schema mới: OK
MinIO/S3 endpoint/path-style/bucket/key/download: OK
Runtime secrets OpenAI/Langfuse/Vertex: OK
USE_NEW_PIPELINE=true: OK
```

Lỗi còn lại hiện nằm ở source code Excel pipeline của image upload worker v2:

```text
'PipelineContext' object has no attribute 'classified_excel_type'
```

Nhờ phía AIFA kiểm tra và rebuild lại image upload worker sau khi fix phần `PipelineContext`, `ClassifyExcelTypeStep`, và `excel_upload_pipeline.py` như đề xuất ở trên.
