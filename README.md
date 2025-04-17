[内容保留至此]

---

## API 接口说明

### 文件上传 OCR 接口（multipart 表单）
- **地址**：`/ocr/upload`
- **方法**：POST
- **参数**：
  - `file`：要处理的 PDF、图像或 Office 文件。
  - `strategy`：OCR 策略（如 `llama_vision`、`minicpm_v`、`remote` 或 `easyocr`）。
  - `ocr_cache`：是否缓存 OCR 结果（`true/false`）。
  - `prompt`：用于 LLM 的提示词。
  - `model`：用于提示处理的模型名。
  - `storage_profile`：存储配置（默认使用 `default` 存储策略）。
  - `storage_filename`：输出文件路径，可使用 `{file_name}`、`{Y}` 等动态变量。
  - `language`：OCR 语言（如 `en` 或 `en,zh`）。

示例：
```bash
curl -X POST -H "Content-Type: multipart/form-data" \
  -F "file=@examples/example-mri.pdf" \
  -F "strategy=easyocr" \
  -F "ocr_cache=true" \
  "http://localhost:8000/ocr/upload"
```

### JSON 请求 OCR 接口
- **地址**：`/ocr/request`
- **方法**：POST
- **参数（JSON 格式）**：
  - `file`：文件内容（Base64 编码）。
  - 其余参数与上传接口一致。

示例：
```bash
curl -X POST http://localhost:8000/ocr/request \
  -H "Content-Type: application/json" \
  -d '{
    "file": "<base64内容>",
    "strategy": "easyocr",
    "ocr_cache": true
  }'
```

### 获取 OCR 结果
- **地址**：`/ocr/result/{task_id}`
- **方法**：GET

### 清理 OCR 缓存
- **地址**：`/ocr/clear_cache`
- **方法**：POST

### 拉取 Ollama 模型
- **地址**：`/llm/pull`
- **方法**：POST
- **参数**：`model`（模型名称）

示例：
```bash
curl -X POST http://localhost:8000/llm/pull \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3.1"}'
```

### 调用 Ollama LLM
- **地址**：`/llm/generate`
- **方法**：POST
- **参数**：`prompt`（提示词），`model`（模型名）

---

## 存储策略支持

OCR 结果可以通过不同的存储策略自动保存，配置位于 `/storage_profiles` 文件夹下。

### 本地文件系统（Local File System）
```yaml
strategy: local_filesystem
settings:
  root_path: /storage
  subfolder_names_format: ""
  create_subfolders: true
```

### Google Drive
```yaml
strategy: google_drive
settings:
  service_account_file: /storage/client_secret.json
  folder_id: YOUR_FOLDER_ID
```
（需要启用 Google Drive API 并生成服务账号授权文件）

### AWS S3（亚马逊对象存储）
```yaml
strategy: aws_s3
settings:
  bucket_name: your-bucket
  region: us-east-1
  access_key: YOUR_ACCESS_KEY
  secret_access_key: YOUR_SECRET
```

请确保 IAM 权限正确配置，具备上传/读取/删除权限。

示例策略：
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject", "s3:GetObject", "s3:ListBucket", "s3:DeleteObject"
  ],
  "Resource": [
    "arn:aws:s3:::your-bucket", "arn:aws:s3:::your-bucket/*"
  ]
}
```

在 `.env` 文件中配置以下环境变量：
```bash
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=...
AWS_S3_BUCKET_NAME=...
```

---

## 许可证 & 联系方式

本项目遵循 MIT 协议。更多详情请参阅 [LICENSE](LICENSE) 文件。

如有任何问题、建议或合作意向，请联系：info@catchthetornado.com

