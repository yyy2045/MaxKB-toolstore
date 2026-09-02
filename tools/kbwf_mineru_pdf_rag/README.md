# PDF 解析 RAG 工作流

## 📖 简介

**PDF 解析 RAG 工作流** 是一个专为 **检索增强生成 (RAG)** 场景设计的高效文档处理模板。它基于在线 **MinerU** API，将复杂的 PDF / DOCX / PPTX / XLSX / 图片文档批量结构化为 Markdown 格式，并自动将提取的图片资源上传至 MaxKB 对象存储，实现真正的“图文并茂”知识库构建。

> **📌 最新版本：[2.3.0](2.3.0/README.md)** —— 基于在线 MinerU API 的**批量多文件**解析版本。适用于 MaxKB v2.10.4-lts 及以上版本。

---

## ✨ 核心功能

* **批量多文件解析**：单次最多提交 50 个文件，超出自动分批，一次登录即可处理整个文件列表，无需为每个文件单独触发。
* **多格式支持**：PDF、DOCX、PPTX、XLSX、图片（PNG / JPG / JPEG / GIF / BMP）。
* **图文混排 Markdown**：自动提取解析结果中的图片，上传至 MaxKB OSS 并回填远程链接，保留完整视觉上下文。
* **直接可入库的输出**：工具直接返回 `[{name, id, content}]`，可被「智能分段 → 知识库写入」节点直接消费，无需流程内二次组装。
* **网络健壮性**：所有 HTTP 请求统一带重试与指数退避，偶发网络抖动不中断流程。
* **可选 OCR 与多模型**：支持 `enable_ocr` 强制 OCR 开关，以及 `vlm` / `pipeline` / `MinerU-HTML` 三种解析模型切换。

---

## 🛠️ 使用说明

### 1. 前置准备

在运行本工作流之前，请确保您已拥有以下服务的访问权限，并完成必要的环境配置：

* **MaxKB 实例**：用于托管工作流及存储解析后的知识库。
* **MinerU API Token**：用于调用 MinerU 的在线解析服务。可前往 [MinerU 官网](https://mineru.net/) 个人中心获取。
* **MaxKB 登录账号**：RSA 登录方式获取下载文件 / 上传图片所需的凭证（cookie + token）。
* **网络连通性**：
    * MaxKB 服务器必须能够访问公网（连接 MinerU API）。
    * MinerU 服务器必须能够访问 MaxKB 的 `maxkb_base_url`（下载源 PDF / 回填图片）。请确保该地址是**公网可达**的。

### 2. 工作流概览

本工作流为一条直线链路，四个环节形成完整的文档处理闭环：

1. **文件列表**（`data-source-local-node`）：接收用户上传的文件，输出 `file_list`。
2. **MinerU 工具**（`tool-lib-node`）：批量下载文件 → 分批提交 MinerU → 轮询等待解析 → 下载 ZIP → 上传图片并回填链接 → 返回 `[{name, id, content}]`。
3. **智能分段**（`document-split-node`）：按 Markdown 标题层级对 `content` 进行切分。
4. **知识库写入**（`knowledge-write-node`）：将分段结果写入向量数据库，完成 RAG 索引构建。

```
文件列表 → MinerU 工具 → 智能分段 → 知识库写入
```

---

### 3. MinerU 工具节点参数

**初始化参数（init）：**

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `api_token` | Password | ✅ | MinerU 平台 API Token（https://mineru.net 个人中心获取） |
| `maxkb_base_url` | String | ✅ | MaxKB 访问地址（含 `/admin`），用于拼接文件下载链接与图片上传。示例：`https://xx.maxkb.com/admin` |
| `kb_id` | String | ✅ | 知识库 id，作为图片上传的 `source_id`，防止 OSS 图片被定期清理 |
| `username` | String | ✅ | MaxKB 登录用户名（RSA 登录获取下载文件/上传图片的凭证） |
| `password` | String | ✅ | MaxKB 登录密码 |
| `enable_ocr` | Switch | ❌ | 是否强制 OCR。`true` 所有文件强 OCR（慢、费额度）；`false` 有文字层的文件用文字层识别（快） |
| `model_version` | Select | ❌ | MinerU 解析模型：`vlm`（最准最慢，默认）/ `pipeline`（快、精度略低）/ `MinerU-HTML` |

**输入参数（input）：**

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `pdf_file_list` | reference | ✅ | 来自「文件列表」节点的 `file_list`，每项含 `file_id`、`name` |

**输出结果：**

* `result`：文档列表 `[{name, id, content}]`，其中 `content` 为清洗后（图片路径已回填为 MaxKB OSS 链接）的 Markdown 全文。

---

## ⚠️ 注意事项

1. **凭证字段为占位符**：模板中内置的 `api_token`、`username`、`password`、`maxkb_base_url` 等均为 `*` 占位，导入后请务必替换为你自己的真实凭证，否则解析会失败。
2. **批量上限**：单批最多提交 50 个文件（MinerU 官方批量接口上限为 200），超出会自动分批。
3. **图片处理**：单文档最多处理 200 张图片，单张超过 10MB 跳过；超出上限的图片不再入库，只保留前 N 张的链接。
4. **MaxKB 需公网可访问**：MinerU 服务端需要能访问 `maxkb_base_url` 以下载源文件，请填写公网可达地址，而非 `localhost`。
5. **解析失败处理**：单个文件下载/解析失败仅跳过该文件并记录日志，不影响整批其他文件入库。
6. **多模型选型**：追求准确度用默认 `vlm`；追求速度且对精度要求不高的场景可切 `pipeline`。

---


## ❓ 常见问题

**Q1: MinerU 报错 "Download failed" 或一直卡在解析中？**
> **原因**：MinerU 服务器无法访问你提供的 `maxkb_base_url` 下载源文件。
> **解决**：请确保 `maxkb_base_url` 填写的是 **公网可访问** 的地址（例如 `https://kb.yourcompany.com/admin`），而不是 `http://localhost` 或 `http://127.0.0.1`。

**Q2: 为什么解析出来的图片在 MaxKB 预览里看不到？**
> **原因**：图片可能上传成功，但 `kb_id` / `maxkb_base_url` 配置错误，导致上传失败或链接不可访问。
> **解决**：检查工具节点的 `kb_id`（知识库 id）与 `maxkb_base_url`，确保 `maxkb_base_url` 生成的图片 URL 是浏览器可直接访问的地址。

**Q3: 上传多个文件时，为什么会跳过某些文件？**
> **原因**：单个文件的下载或解析失败（网络波动、权限不足、文件损坏等），会跳过该文件并记录日志。
> **解决**：查看运行日志中 `[!] ... 跳过` / `处理失败` 的记录，针对失败文件单独排查。

---
