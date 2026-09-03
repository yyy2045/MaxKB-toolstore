## 2.4.0 版本说明

**注意**：当前适用的 MaxKB 版本为 v2.10.4-lts 及以上版本，MinerU（API 服务）版本为 3.4.4。

本版本较 2.3.0 的主要变化：

- 解析服务由「本地 MinerU Gradio」改为「**本地 MinerU API（`/tasks` 异步批量）**」
- 支持格式由「仅 PDF」扩展到 **PDF / Word(docx) / PPT / Excel / 图片**（MinerU 3.4.4 原生支持）
- 新增 **Word 转 PDF 开关**：开启时先经 unoserver 把 Word 转成 PDF 再交给 MinerU；关闭时原样透传，直接解析

## 简介

**MinerU 本地 PDF 入库工作流模板（2.4.0）** 是一个面向知识库构建场景的工作流模板。它调用本地 MinerU API 服务解析用户上传的文档，直接获取 Markdown 文本，并在 MaxKB 内继续完成文档分段和知识库入库。


## 工作流能力

- 接收用户上传的多种文件：PDF / Word(docx) / PPT(pptx) / Excel(xlsx) / 图片
- 调用 MinerU API（`http://<mineru-ip>:9000`）完成文档转 Markdown
- 可选开启 **Word 转 PDF** 开关：`word_pdf_enable=true` 时先经 unoserver 转换，`false` 时直接透传解析
- 将分段结果写入指定知识库，完成 RAG 入库

## 前置条件

1. 已部署可访问的 MinerU API 服务（默认 `http://<mineru-ip>:9000`，支持 `/tasks` 接口）
2. 若开启 Word 转 PDF 开关，需已部署可访问的 **unoserver** 服务（默认 `http://<unoserver-ip>:2003`）

## 工作流结构

该模板包含以下核心节点：

1. **本地文件**（数据源）：接收用户上传的文件列表
2. **Word转PDF**（工具节点）：按 `word_pdf_enable` 开关决定「转 PDF」还是「透传原文件」
3. **MinerU 解析**（工具节点）：调用 MinerU API 解析为 Markdown
4. **文档分段**：对 Markdown 文本进行切分
5. **知识库写入**：将切分结果写入知识库

## 关键参数

### 开关（工作流全局参数）

| 参数名 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `word_pdf_enable` | Boolean | ❌ | Word 转 PDF 开关；`true` 先转 PDF 再解析，`false` 透传原文件直接解析 |

### Word转PDF 工具节点启动参数

| 参数名 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `unoserver_url` | String | ✅ | unoserver 服务地址，如 `http://<unoserver-ip>:2003` |
| `api_base_url` | String | ✅ | MaxKB 访问地址，如 `http://<maxkb-ip>:8080/` |
| `api_auth_token` | String | ✅ | MaxKB 用户 API Token（`user-xxx`） |
| `username` | String | ✅ | MaxKB 登录用户名，用于获取文件下载 Cookie |
| `password` | String | ✅ | MaxKB 登录密码 |
| `source_id` | String | ✅ | 知识库 ID，用于上传归属（防止转出的 PDF 被临时清理） |

### MinerU 解析工具节点启动参数

| 参数名 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `mineru_api_base_url` | String | ✅ | MinerU API 服务地址，如 `http://<mineru-ip>:9000` |
| `url_prefix` | String | ✅ | MaxKB 基础地址前缀，用于拼接文件下载地址，如 `http://<maxkb-ip>:8080/admin` |
| `upload_token` | String | ✅ | MaxKB 用户 API Token（`user-xxx`），用于 OSS 接口鉴权 |
| `username` | String | ✅ | MaxKB 登录用户名 |
| `password` | String | ✅ | MaxKB 登录密码 |
| `knowledge_id` | String | ✅ | 当前工作流知识库 ID，用于图片上传归属 |
| `backend` | String | ✅ | MinerU 处理引擎，如 `pipeline`、`hybrid-engine` |
| `parse_method` | String | ✅ | 解析方式，如 `auto`、`txt`、`ocr` |
| `effort` | String | ✅ | 解析力度，如 `medium`、`high` |
| `formula_enable` | Boolean | ✅ | 是否解析公式 |
| `table_enable` | Boolean | ✅ | 是否解析表格 |
| `image_analysis` | Boolean | ✅ | 是否进行图片/图表分析 |

## 使用说明

1. 导入该 `kbwf` 模板到 MaxKB
2. 确认 `unoserver_url`、`api_base_url`、`mineru_api_base_url`、`url_prefix` 填写正确
3. 填写两个工具节点的 `api_auth_token`、`username`、`password`、`knowledge_id` / `source_id`
4. 按需设置 MinerU 的 `backend`、`parse_method` 等
5. 上传测试文件
6. 按需开启 / 关闭 `word_pdf_enable` 开关
7. 检查工具节点输出中的 `content`
8. 确认文档分段与知识库写入结果正常

## 注意事项

- 若在 ON 状态混入非 Word 文件，Word转PDF 节点会因「不支持的文件格式」报错——请按文件类型分开上传
- MinerU 3.4.4 原生支持 docx / pptx / xlsx / 图片，无需必经 Word 转 PDF

- **无 GPU 建议**：没有 GPU 时建议开启 `word_pdf_enable`，开启后可达与 GPU 相同的解析效果。

## 关联工具

- `tool_unoserver_file_converter`：Word 转 PDF 工具（依赖 unoserver 服务）
- `mineru-parser`（MinerU API 批量解析）：调用本地 MinerU `/tasks` 接口解析
