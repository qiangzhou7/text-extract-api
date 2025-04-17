# text-extract-api

将任何图像、PDF 或 Office 文档以超高精度转换为 Markdown *文本* 或结构化 JSON 文档，支持表格数据、数字或数学公式。

该 API 使用 FastAPI 构建，并使用 Celery 进行异步任务处理，Redis 用于缓存 OCR 结果。

![文档提取展示](ocr-hero.webp)

## 功能特点：
- **无需云服务或外部依赖**：所有组件包括基于 PyTorch 的 OCR（EasyOCR）和 Ollama 均通过 `docker-compose` 进行配置，数据不会离开您的开发或服务器环境。
- **PDF/Office 转换为 Markdown**：支持多种 OCR 策略，具有极高的准确率，例如 [llama3.2-vision](https://ai.meta.com/blog/llama-3-2-connect-2024-vision-edge-mobile-devices/)、[easyOCR](https://github.com/JaidedAI/EasyOCR)、[minicpm-v](https://github.com/OpenBMB/MiniCPM-o?tab=readme-ov-file#minicpm-v-26)、以及远程 URL 策略如 [marker-pdf](https://github.com/VikParuchuri/marker)。
- **PDF/Office 转换为 JSON**：使用 Ollama 支持的模型（例如 LLama 3.1）进行结构化数据提取。
- **使用 LLM 提升 OCR 结果质量**：LLama 可用于修复 OCR 中的拼写及文本识别问题。
- **去除敏感信息（PII）**：可用于从文档中删除个人身份信息，详见 `examples` 示例。
- **使用 [Celery](https://docs.celeryq.dev/en/stable/getting-started/introduction.html) 实现分布式队列处理**
- **缓存支持**：OCR 结果可在进入 LLM 处理前使用 Redis 进行缓存。
- **可切换存储策略**：支持 Google Drive、本地文件系统等多种存储方式。
- **命令行工具（CLI）**：可用于提交任务和处理结果。

## 截图示例

将 MRI 报告转换为 Markdown + JSON：

```bash
python client/cli.py ocr_upload --file examples/example-mri.pdf --prompt_file examples/example-mri-2-json-prompt.txt
```

运行前请参考 [快速开始](#快速开始)

![MRI 转换示例](./screenshots/example-1.png)

将发票转换为 JSON 并移除敏感信息：

```bash
python client/cli.py ocr_upload --file examples/example-invoice.pdf --prompt_file examples/example-invoice-remove-pii.txt
```

运行前请参考 [快速开始](#快速开始)

![发票转换示例](./screenshots/example-2.png)

## 快速开始

您可以直接在本地运行该应用，方便开发调试，或启用 Apple GPU（Docker 当前不支持）。

### 前置条件

请按以下步骤进行安装：

- [下载并安装 Ollama](https://ollama.com/download)
- [下载并安装 Docker](https://www.docker.com/products/docker-desktop)

> ### 在远程主机上设置 Ollama
> 若需连接外部 Ollama 实例，请设置环境变量：
> ```bash
> OLLAMA_HOST=http(s)://127.0.0.1:5000
> ```
> 如需禁用本地 Ollama 模型：
> ```bash
> DISABLE_LOCAL_OLLAMA=1 make install
> ```
> **注意：** 目前 `DISABLE_LOCAL_OLLAMA` 变量无法在 Docker 中使用。如需禁用 Docker 中的 Ollama，请从 `docker-compose.yml` 中移除相关服务配置。

（下略）


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

