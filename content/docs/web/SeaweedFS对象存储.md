# SeaweedFS 对象存储说明

本文说明本项目为什么用 SeaweedFS、它和 PostgreSQL / 本地磁盘的差别，以及本地开发时 URL、桶、Key 怎么对应。更完整的技术选型见 [项目技术栈文档.md](./项目技术栈文档.md)；端口与账号见 [docker/README.md](../docker/README.md)。

---

## 1. 它在架构里做什么

业务数据分两摊：

| 存什么 | 放哪里 | 怎么读写 |
|--------|--------|----------|
| 结构化信息（用户、手术元数据、权限、文件名） | PostgreSQL | SQL |
| 大文件本体（手术视频、知识库 PDF、头像） | SeaweedFS | HTTP / S3 API |

PostgreSQL 表里只记一份「名片」，核心字段是 `storage_key`（对象键）。真正的字节在 SeaweedFS 里。后端封装见 `backend/internal/storage/s3.go`，通过 **S3 兼容 API** 访问，而不是 SQL。

当前 Docker 使用镜像 `chrislusf/seaweedfs:4.29`，单节点 `mini` 模式，开启 S3。选用原因包括：Apache 2.0 协议适合私有化商用，以及 Volume 结构适合以后 HLS 切片产生的大量小文件。

---

## 2. 为什么不直接写服务器磁盘

小文件、单机项目可以把文件放到 `./uploads/`。本项目以手术录像为主，不用业务机本地路径当主存储，原因是：

1. **大文件不应打满 Go 进程。** 几 GB 视频若 `POST` 进后端，内存、超时、带宽都压在 API 上。预签名直传让浏览器把文件送到 SeaweedFS，Go 只做鉴权和记账。
2. **路径不能绑死在一台机器。** `D:\data\a.mp4` 换机、多开几份 Go、单独起转码服务都会乱。对象存储用统一的 Key，应用始终 `PutObject` / `GetObject`。
3. **多服务要共享同一批文件。** 本地盘属于某一台机器；对象存储是网络服务。
4. **海量小文件（后续 HLS `.ts`）** 更适合 Volume，而不是普通文件系统里堆积百万个文件。

底层 SeaweedFS 仍然落在磁盘上（Docker volume `sqa_seaweed_data`），只是对外提供「按对象访问」的接口，不让 Web 程序自己管物理路径。

---

## 3. 对象怎么定位：桶 + Key + 内容

对「一份文件」而言，逻辑上是 **Key + 内容**。S3 还要求一个 **Bucket（桶）**，作为命名空间，用来分组、授权、设生命周期。Key 只在桶内唯一。完整定位：

```
桶 + Key  →  那一份内容
```

本项目开发阶段只用一个桶 `surgical-videos`。单桶时桶几乎是固定前缀，但 S3 API 的 Put/Get **必须带 Bucket**，SDK 不允许只传 Key。

Key 里的 `/` 只是字符串习惯写法，对象存储里通常没有真实的多级文件夹。用前缀区分业务即可：

| 业务 | Key 格式（代码生成） |
|------|----------------------|
| 手术源视频 | `surgeries/{surgeryId}/source/{随机}.mp4` |
| 知识库文档 | `knowledge/{userId}/{时间戳}_{随机}{ext}` |
| 用户头像 | `avatars/{userId}/{时间戳}_{随机}{ext}` |

---

## 4. 本地地址与 URL 长什么样

`docker/docker-compose.yml` 映射：

| 用途 | 本机地址 | 说明 |
|------|----------|------|
| S3 API | `http://localhost:18333` | 业务上传/下载走这条（容器内 8333） |
| Filer 网页 | `http://localhost:18088` | 浏览器里逛文件，给人看的 |
| Master 网页 | `http://localhost:19333` | 集群/Volume 状态 |

后端默认环境变量：`S3_ENDPOINT=http://localhost:18333`，`S3_BUCKET=surgical-videos`，并启用 **path-style**（路径风格）。对象逻辑地址是：

```
http://localhost:18333/surgical-videos/<key>
```

示例：

```
http://localhost:18333/surgical-videos/surgeries/12/source/a1b2c3....mp4
http://localhost:18333/surgical-videos/knowledge/<userId>/1786691787_ae4805748a749901.pdf
```

Filer 界面浏览同一对象时，路径会带 `buckets` 前缀，例如：

```
http://localhost:18088/buckets/surgical-videos/knowledge/<userId>/
```

这是管理 UI 的路径，**不是** 前端直传使用的 S3 URL。

公有云常见另一种写法（虚拟主机风格）：`http://桶名.s3.amazonaws.com/key`。本地为避免 `桶.localhost` 的麻烦，使用路径风格。

---

## 5. 预签名直传（业务真正用的 URL）

为避免大视频经过 Go，上传流程是：

1. 前端向后端申请上传（已登录）。
2. Go 鉴权后调用 S3 `PresignPutObject`，生成限时 URL（默认约 20 分钟，`UploadURLTTL`）。
3. 浏览器按返回的 method（一般为 PUT）把文件 **直接传到 SeaweedFS**。
4. 前端再通知后端「传完了」；Go 把 `storage_key` 写入 PostgreSQL。

预签名 URL 的路径仍是「桶 + Key」，查询串是通行证，例如：

```
http://localhost:18333/surgical-videos/surgeries/12/source/a1b2....mp4
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=...
  &X-Amz-Date=...
  &X-Amz-Expires=1200
  &X-Amz-Signature=...
```

本地开发凭据（仅开发，交付必须更换）：

- Access Key：`sqa_access_key`
- Secret Key：`sqa_secret_key_change_me`

大视频还可走分片上传（`CreateMultipartUpload` + 各 part 的预签名），见 `surgeryfile` 模块。

---

## 6. Filer 页眉里的「30GB」是什么

Filer 标题旁类似 `30GB 4.29 1355c7a10` 的字样，来自 `weed version`，**不是已用容量**：

| 片段 | 含义 |
|------|------|
| 30GB | 普通构建：每个 Volume 上限约 30GB |
| 4.29 | 软件版本 |
| 1355c7a10 | 构建对应的 git commit 短哈希 |

SeaweedFS 把许多对象打进较大的 Volume 文件。标准版每个 Volume 大约最多 30GB，满了再开新 Volume。这是编译能力上限，不是 Docker 磁盘配额，也不是当前剩余空间。页面上几个几百 KB 的 PDF 与这个数字无关。

另有 `large_disk` 构建可支持更大 Volume，与普通 30GB 版数据不兼容。本项目用的是标准镜像。

---

## 7. 和本仓库代码的对应

| 项 | 位置 |
|----|------|
| Compose 与端口 | `docker/docker-compose.yml` |
| 本地连接说明 | `docker/README.md` |
| 后端 S3 客户端 / 预签名 | `backend/internal/storage/s3.go` |
| 默认 endpoint、桶、TTL | `backend/internal/config/config.go` |
| 手术文件 Key 与分片 | `backend/internal/module/surgeryfile/` |
| 知识库文档 Key | `backend/internal/module/knowledge/` |
| 头像 Key | `backend/internal/module/personal/avatar_service.go` |

数据在 volume `sqa_seaweed_data` 中。普通 `docker compose down` 不删数据；`docker compose down -v` 会清空对象存储，不可恢复。
