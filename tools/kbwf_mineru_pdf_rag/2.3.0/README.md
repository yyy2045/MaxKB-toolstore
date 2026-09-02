## 2.3.0 版本说明

**注意**：当前适用的 MaxKB 版本为 v2.10.4-lts 及以上版本。

---

### ✨ 本版本核心变化

**2.3.0 相比之前的版本，重写了 MinerU 解析工具，从「单文件」升级为「批量多文件」架构，并移除了 LLM 标题增强链路。**

| 维度 | 之前的 2.x 版本 | **2.3.0（本版本）** |
| :--- | :--- | :--- |
| 文件处理 | 单文件（`pdf_file[0]`，多文件也只处理第一个） | **批量多文件**（最多 50 个/批，自动分批提交） |
| LLM 标题增强 | 有（提取标题 → AI 对话 → 内容替换 → 判断器） | **移除**，结构更简洁，省 Token、省耗时 |
| 工具返回 | 单个 Markdown 字符串，需流程内组装 | **直接返回 `[{name, id, content}]`（B 形）**，可被「智能分段 → 知识库写入」直接消费 |
| 节点数量 | 多（含判断器 / AI 对话 / 多个自定义工具 / 变量聚合） | **4 步链路**：文件列表 → MinerU 工具 → 智能分段 → 知识库写入 |
| 网络健壮性 | 裸 `requests` 调用 | `_request_with_retry` 统一带网络重试与退避 |
| 解析参数 | 仅 `model_version` | 可选 `enable_ocr`、`model_version`（vlm / pipeline） |


---

### 🔧 MinerU 工具节点参数

**初始化参数（init）：**

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `api_token` | Password | ✅ | MinerU 平台 API Token（https://mineru.net 个人中心获取） |
| `maxkb_base_url` | String | ✅ | MaxKB 访问地址（含 `/admin`），用于拼接文件下载链接与图片上传。示例：`https://xx.maxkb.com/admin` |
| `kb_id` | String | ✅ | 知识库 id，作为图片上传的 `source_id`，防止 OSS 图片被定期清理 |
| `username` | String | ✅ | MaxKB 登录用户名（RSA 登录获取下载文件/上传图片的凭证） |
| `password` | String | ✅ | MaxKB 登录密码 |
| `enable_ocr` | Switch | ❌ | 是否强制 OCR。`true` 所有文件强 OCR（慢、费额度）；`false` 有文字层的文件用文字层识别（快） |
| `model_version` | Select | ❌ | MinerU 解析模型：`vlm`（最准最慢，默认）/ `pipeline`（快、精度略低）|

**输入参数（input）：**

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `pdf_file_list` | reference | ✅ | 来自「文件列表」节点的 `file_list`，每项含 `file_id`、`name` |

**输出结果：**

* `result`：文档列表 `[{name, id, content}]`，其中 `content` 为清洗后（图片路径已回填为 MaxKB OSS 链接）的 Markdown 全文。

---

### ⚠️ 注意事项

1. **凭证字段为占位符**：模板中内置的 `api_token`、`username`、`password`、`maxkb_base_url` 等均为 `*` 占位，导入后请务必替换为你自己的真实凭证，否则解析会失败。
2. **批量上限**：单批最多提交 50 个文件（MinerU 官方批量接口上限为 200），超出会自动分批。
3. **图片处理**：单文档最多处理 200 张图片，单张超过 10MB 跳过；超出上限的图片不再入库，只保留前 N 张的链接。
4. **MaxKB 需公网可访问**：MinerU 服务端需要能访问 `maxkb_base_url` 以下载源文件，请填写公网可达地址，而非 `localhost`。
5. **解析失败处理**：单个文件下载/解析失败仅跳过该文件并记录日志，不影响整批其他文件入库。
6. **多模型选型**：追求准确度用默认 `vlm`；追求速度且对精度要求不高的场景可切 `pipeline`。
