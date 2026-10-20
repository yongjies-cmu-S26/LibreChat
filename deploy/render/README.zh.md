# 在 Render 免费版上部署 LibreChat（全功能，个人使用）

三个 Render 免费服务 + 一个 MongoDB Atlas 免费数据库：

| 服务 | 作用 |
|---|---|
| `yongjie-librechat` | 主程序（官方 v0.8.8 镜像 + 本仓库的 `librechat.yaml`） |
| `yongjie-librechat-meili` | Meilisearch，搜索所有历史对话 |
| `yongjie-librechat-rag` | RAG API，和上传的文件对话（向量存进 Atlas） |
| MongoDB Atlas M0 | 用户、对话、记忆、文件向量 |

已开启：长期记忆、Projects、Agents、Artifacts、联网搜索（Keenable，免 Key）、图片生成、语音朗读/语音输入、文件检索、对话搜索、MCP、分享链接、临时对话、书签、Prompts、Skills、定时对话、长对话自动摘要、付费/免费两个 OpenAI Key。

---

## 第 1 步：创建 MongoDB Atlas 免费集群

1. 登录 https://cloud.mongodb.com ，新建 **M0 Free** 集群，区域选 **AWS / Oregon (us-west-2)**（和 Render 同区，速度快）。
2. **Database Access**：新建一个数据库用户，记下用户名和密码。
3. **Network Access**：添加 `0.0.0.0/0`（Render 免费版没有固定 IP）。
4. **Connect → Drivers**，复制连接串，把密码填进去，并在 `/?` 之间加上库名 `LibreChat`：
   ```
   mongodb+srv://用户名:密码@cluster0.xxxxx.mongodb.net/LibreChat?retryWrites=true&w=majority
   ```

### 1b. 给 RAG 建向量索引（文件检索需要）

1. 在集群里 **Browse Collections → Create Database**：库名 `LibreChat`，集合名 `rag_vectors`。
2. 进入 **Atlas Search → Create Search Index → Atlas Vector Search → JSON Editor**，
   选择 `LibreChat.rag_vectors`，索引名填 `vector_index`，粘贴：
   ```json
   {
     "fields": [
       { "type": "vector", "path": "embedding", "numDimensions": 1536, "similarity": "cosine" },
       { "type": "filter", "path": "file_id" },
       { "type": "filter", "path": "user_id" }
     ]
   }
   ```

## 第 2 步：用 Blueprint 部署到 Render

1. 打开 https://dashboard.render.com/blueprints → **New Blueprint Instance**，选择这个仓库（`yongjies-cmu-S26/LibreChat`），分支 `main`。
2. Render 会读取根目录的 `render.yaml`，列出三个服务，并让你填写这些值：

| 变量 | 填什么 |
|---|---|
| `MONGO_URI` | 第 1 步的连接串 |
| `CREDS_KEY` | 64 位十六进制（Claude 已在聊天里发给你，或用 https://www.librechat.ai/toolkit/creds_generator 生成） |
| `CREDS_IV` | 32 位十六进制（同上） |
| `OPENAI_API_KEY` | 付费 OpenAI Key |
| `OPENAI_FREE_API_KEY` | 免费 OpenAI Key |
| `IMAGE_GEN_OAI_API_KEY` | 和 `OPENAI_API_KEY` 相同 |

3. 点 **Apply**，等三个服务都变成 **Live**（第一次拉镜像要几分钟）。

## 第 3 步：检查网址

如果 Render 给某个服务的网址加了后缀（比如 `yongjie-librechat-abcd.onrender.com`，因为名字被占用），到对应服务的 **Environment** 里改：

- `yongjie-librechat`：`DOMAIN_CLIENT`、`DOMAIN_SERVER` = 主程序的真实网址；`MEILI_HOST` = meili 服务网址；`RAG_API_URL` = rag 服务网址。

## 第 4 步：注册并关闭注册

1. 打开主程序网址，注册你自己的账号（第一个注册的账号是管理员）。
2. 回到 Render → `yongjie-librechat` → Environment，把 `ALLOW_REGISTRATION` 改成 `false`，保存（会自动重启）。这样别人就不能注册了。

---

## 可选功能（需要额外的 Key）

在 `yongjie-librechat` 的 Environment 里加上即可：

| 功能 | 变量 |
|---|---|
| 代码执行（Code Interpreter，付费服务 https://code.librechat.ai ） | `LIBRECHAT_CODE_API_KEY` |
| 联网搜索提高频率限制 | `KEENABLE_API_KEY` |
| Claude / Gemini 固定 Key（不想在界面里填） | 把 `ANTHROPIC_API_KEY` / `GOOGLE_KEY` 从 `user_provided` 改成真实 Key |
| 文件 OCR（扫描件识别） | Mistral OCR Key，见官方文档 `ocr:` 配置 |

## 修改配置

- 功能开关在 `deploy/render/librechat.yaml`。改完 push 到 `main`，Render 会自动重新部署。
- 升级 LibreChat：修改 `deploy/render/Dockerfile` 第一行的版本号（如 `v0.8.9`）。

## 免费版的已知限制

- **休眠**：15 分钟没人访问会休眠，再打开要等 30–60 秒。搜索和 RAG 服务也会各自休眠，第一次搜索或上传文件可能要再等一下。
- **750 小时/月**：三个服务共用这个额度。个人使用时服务大部分时间在休眠，一般够用；不够的话可以先把 meili 或 rag 服务暂停。
- **上传的文件不持久**：Render 免费版没有硬盘，重启后原始上传文件会丢（对话、记忆、向量都在 Atlas，不受影响）。需要长期保存文件时，可以接 Cloudflare R2 或 S3（`fileStrategy: s3`）。
- **搜索索引**：Meilisearch 重启后索引为空，LibreChat 下次启动时会从数据库重新同步。
- **内存**：每个服务只有 512MB。如果主程序经常因内存不足（OOM）重启，可以升级到 Starter（约 7 美元/月）。
- **定时对话**：只在服务醒着的时候触发。
