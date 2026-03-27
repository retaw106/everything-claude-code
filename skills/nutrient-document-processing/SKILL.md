---
name: nutrient-document-processing
description: 使用 Nutrient DWS API 处理、转换、OCR、提取、编辑、签名和填充文档。支持 PDF、DOCX、XLSX、PPTX、HTML 和图片。
origin: ECC
---

# Nutrient 文档处理

> **注意：** 此技能与 Nutrient 商业 API 集成。使用前请查看其条款。

使用 [Nutrient DWS Processor API](https://www.nutrient.io/api/) 处理文档。转换格式、提取文本和表格、OCR 扫描文档、编辑 PII、添加水印、数字签名和填充 PDF 表单。

## 设置

在 **[nutrient.io](https://dashboard.nutrient.io/sign_up/?product=processor)** 获取免费 API 密钥

```bash
export NUTRIENT_API_KEY="pdf_live_..."
```

所有请求作为 multipart POST 发送到 `https://api.nutrient.io/build`，带有 `instructions` JSON 字段。

## 操作

### 转换文档

```bash
# DOCX 转 PDF
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.docx=@document.docx" \
  -F 'instructions={"parts":[{"file":"document.docx"}]}' \
  -o output.pdf

# PDF 转 DOCX
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"output":{"type":"docx"}}' \
  -o output.docx

# HTML 转 PDF
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "index.html=@index.html" \
  -F 'instructions={"parts":[{"html":"index.html"}]}' \
  -o output.pdf
```

支持的输入：PDF、DOCX、XLSX、PPTX、DOC、XLS、PPT、PPS、PPSX、ODT、RTF、HTML、JPG、PNG、TIFF、HEIC、GIF、WebP、SVG、TGA、EPS。

### 提取文本和数据

```bash
# 提取纯文本
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"output":{"type":"text"}}' \
  -o output.txt

# 提取表格为 Excel
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"output":{"type":"xlsx"}}' \
  -o tables.xlsx
```

### OCR 扫描文档

```bash
# OCR 为可搜索 PDF（支持 100+ 语言）
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "scanned.pdf=@scanned.pdf" \
  -F 'instructions={"parts":[{"file":"scanned.pdf"}],"actions":[{"type":"ocr","language":"english"}]}' \
  -o searchable.pdf
```

语言：通过 ISO 639-2 代码支持 100+ 语言（如 `eng`、`deu`、`fra`、`spa`、`jpn`、`kor`、`chi_sim`、`chi_tra`、`ara`、`hin`、`rus`）。完整语言名称如 `english` 或 `german` 也可以。查看[完整 OCR 语言表](https://www.nutrient.io/guides/document-engine/ocr/language-support/)了解所有支持的代码。

### 编辑敏感信息

```bash
# 基于模式（SSN、邮箱）
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"actions":[{"type":"redaction","strategy":"preset","strategyOptions":{"preset":"social-security-number"}},{"type":"redaction","strategy":"preset","strategyOptions":{"preset":"email-address"}}]}' \
  -o redacted.pdf

# 基于正则表达式
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"actions":[{"type":"redaction","strategy":"regex","strategyOptions":{"regex":"\\b[A-Z]{2}\\d{6}\\b"}}]}' \
  -o redacted.pdf
```

预设：`social-security-number`、`email-address`、`credit-card-number`、`international-phone-number`、`north-american-phone-number`、`date`、`time`、`url`、`ipv4`、`ipv6`、`mac-address`、`us-zip-code`、`vin`。

### 添加水印

```bash
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"actions":[{"type":"watermark","text":"CONFIDENTIAL","fontSize":72,"opacity":0.3,"rotation":-45}]}' \
  -o watermarked.pdf
```

### 数字签名

```bash
# 自签名 CMS 签名
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "document.pdf=@document.pdf" \
  -F 'instructions={"parts":[{"file":"document.pdf"}],"actions":[{"type":"sign","signatureType":"cms"}]}' \
  -o signed.pdf
```

### 填充 PDF 表单

```bash
curl -X POST https://api.nutrient.io/build \
  -H "Authorization: Bearer $NUTRIENT_API_KEY" \
  -F "form.pdf=@form.pdf" \
  -F 'instructions={"parts":[{"file":"form.pdf"}],"actions":[{"type":"fillForm","formFields":{"name":"Jane Smith","email":"jane@example.com","date":"2026-02-06"}}]}' \
  -o filled.pdf
```

## MCP 服务器（替代方案）

对于原生工具集成，使用 MCP 服务器而不是 curl：

```json
{
  "mcpServers": {
    "nutrient-dws": {
      "command": "npx",
      "args": ["-y", "@nutrient-sdk/dws-mcp-server"],
      "env": {
        "NUTRIENT_DWS_API_KEY": "YOUR_API_KEY",
        "SANDBOX_PATH": "/path/to/working/directory"
      }
    }
  }
}
```

## 何时使用

- 在格式之间转换文档（PDF、DOCX、XLSX、PPTX、HTML、图片）
- 从 PDF 中提取文本、表格或键值对
- 对扫描文档或图片进行 OCR
- 分享文档前编辑 PII
- 为草稿或机密文档添加水印
- 数字签署合同或协议
- 编程方式填充 PDF 表单

## 链接

- [API Playground](https://dashboard.nutrient.io/processor-api/playground/)
- [完整 API 文档](https://www.nutrient.io/guides/dws-processor/)
- [npm MCP 服务器](https://www.npmjs.com/package/@nutrient-sdk/dws-mcp-server)
