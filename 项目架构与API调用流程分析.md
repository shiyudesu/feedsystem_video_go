# feedsystem_video_go 项目架构与 API 调用流程分析

## 1. 项目整体定位

这是一个“短视频 Feed 系统”全栈项目，仓库里同时包含：

- `backend/`：Go 后端，负责账号、视频、点赞、评论、关注、Feed 流、鉴权、缓存、消息队列。
- `frontend/`：Vue 3 + Pinia + Vue Router 前端，负责页面展示、登录态管理、接口调用。
- `docker-compose.yml`：用于一键拉起 MySQL、Redis、RabbitMQ、后端 API、Worker、前端。
- `picture/`：架构图、流程图、表关系图。

这个项目不是单体“只查数据库”的简单 CRUD，而是一个带有缓存、异步消息、Worker、降级策略的分层系统。

---

## 2. 代码目录与职责划分

### 2.1 后端目录

后端主要结构如下：

```text
backend/
  cmd/
    main.go                 API 进程入口
    worker/main.go          Worker 进程入口
  internal/
    account/                账号模块
    social/                 关注模块
    video/                  视频、点赞、评论、热度、Outbox
    feed/                   Feed 流模块
    auth/                   JWT 生成与解析
    middleware/
      jwt/                  Gin 鉴权中间件
      redis/                Redis 封装
      rabbitmq/             RabbitMQ 封装
    http/
      router.go             依赖装配与路由注册
    db/
      db.go                 MySQL 初始化与迁移
    config/
      loadconfig.go         配置加载
    worker/                 MQ 消费者、Outbox 轮询器、时间线消费者
```

### 2.2 前端目录

前端主要结构如下：

```text
frontend/src/
  api/                      所有 HTTP API 封装
  stores/                   Pinia 状态管理
  router/                   路由配置
  views/                    页面级组件
  components/               复用组件
  utils/                    JWT 解析等工具
```

前端分层非常清晰：

- `views/` 负责页面交互。
- `api/` 负责请求后端。
- `stores/` 负责保存登录态、社交状态、Toast 状态。

---

## 3. 后端总体架构

### 3.1 典型分层

每个业务模块基本都遵循：

```text
Gin Router -> Handler -> Service -> Repository -> MySQL
                              |
                              +-> Redis
                              |
                              +-> RabbitMQ
```

含义：

- `Handler`：解析请求、校验参数、返回 HTTP 响应。
- `Service`：写业务规则、容错策略、缓存策略、异步策略。
- `Repository`：只关心数据访问，直接操作 GORM/MySQL。

这是项目最核心的结构。

### 3.2 两个进程

项目运行时分成两个后端进程：

#### 1. API 进程

入口：`backend/cmd/main.go`

职责：

- 加载配置
- 连接 MySQL
- 尝试连接 Redis
- 尝试连接 RabbitMQ
- 组装路由并启动 Gin Server

#### 2. Worker 进程

入口：`backend/cmd/worker/main.go`

职责：

- 单独连接 MySQL、Redis、RabbitMQ
- 监听多个 MQ 队列
- 异步处理点赞、评论、关注、热度更新

所以这个项目的写路径很多时候是“API 快速接入 -> 发 MQ -> Worker 异步落地”。

---

## 4. 基础设施层

### 4.1 MySQL

数据库连接和迁移在 `backend/internal/db/db.go`：

- 使用 GORM + MySQL driver
- 启动时自动迁移这些模型：
  - `account.Account`
  - `video.Video`
  - `video.Like`
  - `video.Comment`
  - `social.Social`
  - `video.OutboxMsg`

说明这个系统的主存储仍然是 MySQL。

### 4.2 Redis

Redis 主要承担三类职责：

#### 1. Token 缓存

- 键：`account:{accountID}`
- 用于 JWT 中间件快速校验 token 是否为当前有效 token
- Redis 失效时会回退到 DB 查询

#### 2. 视频详情 / Feed 缓存

- `video:detail:id={id}`：视频详情缓存
- `feed:listByFollowing:...`：关注流缓存
- `video:entity:{id}`：视频实体缓存
- 本地 L1 缓存 + Redis L2 缓存 + MySQL L3 查询

#### 3. 时间线 / 热榜

- `feed:global_timeline`：全局最新视频时间线 ZSET
- `hot:video:1m:{yyyyMMddHHmm}`：一分钟热度窗口
- `hot:video:merge:1m:{yyyyMMddHHmm}`：热榜快照

因此 Redis 不只是普通缓存，还承担“排序索引”和“读优化”的角色。

### 4.3 RabbitMQ

RabbitMQ 主要承担异步写任务：

- `social.events`：关注 / 取关
- `like.events`：点赞 / 取消点赞
- `comment.events`：发表评论 / 删除评论
- `video.popularity.events`：视频热度变化
- `video.timeline.events`：新发布视频进入全局时间线

也就是说，项目把“写扩散”和“缓存更新”从 API 线程中拆了出去。

---

## 5. 各业务模块架构

### 5.1 account 模块

文件：

- `backend/internal/account/entity.go`
- `backend/internal/account/repo.go`
- `backend/internal/account/service.go`
- `backend/internal/account/handler.go`

职责：

- 注册
- 登录
- 登出
- 改名
- 改密码
- 按 ID / 用户名查账号

关键点：

- 密码使用 `bcrypt` 哈希。
- 登录成功后生成 JWT，并把 token 同时写 DB 和 Redis。
- 登出会删除 Redis 中的 token 并清空 DB 中保存的 token。
- 改名会重新签发 token，因为 JWT claims 中带有 `username`。

### 5.2 JWT 鉴权模块

文件：

- `backend/internal/auth/jwt.go`
- `backend/internal/middleware/jwt/jwt.go`

职责：

- 生成和解析 JWT
- Gin 中间件做强鉴权 `JWTAuth`
- Gin 中间件做软鉴权 `SoftJWTAuth`

关键机制：

1. 从 `Authorization: Bearer xxx` 读 token
2. 解析 JWT claims
3. 优先去 Redis 里查 `account:{id}`
4. Redis 未命中或不可用时，回退到 MySQL 查账号表里的当前 token
5. 校验通过后，把 `accountID` 和 `username` 写入 Gin Context

这套设计支持“单点登录 / token 撤销”，因为系统不是只验 JWT 签名，还验证“它是不是当前最新 token”。

### 5.3 video 模块

文件：

- `video_handler.go`
- `video_service.go`
- `video_repo.go`
- `video_entity.go`

职责：

- 上传视频文件
- 上传封面
- 发布视频
- 查询视频详情
- 查作者视频列表

关键点：

- 文件上传保存到 `backend/.run/uploads`，再由 Gin `r.Static("/static", "./.run/uploads")` 暴露。
- 发布视频采用“事务 + Outbox”：
  - 先写 `videos`
  - 再写 `outbox_msgs`
  - 后台轮询器把 Outbox 事件投递到 `video.timeline.events`
- 视频详情读路径带缓存与防击穿锁。

### 5.4 like 模块

文件：

- `like_handler.go`
- `like_service.go`
- `like_repo.go`
- `like_entity.go`

职责：

- 点赞
- 取消点赞
- 查询是否已点赞
- 查询我点赞过的视频

关键点：

- 优先异步：
  - 发 `like.events`，让 Worker 落 MySQL
  - 发 `video.popularity.events`，让 Worker 更新 Redis 热榜
- 如果 MQ 发失败，Service 会同步兜底：
  - 直接事务写 MySQL
  - 直接更新 Redis 热榜窗口

这是项目里最典型的“异步优先 + 同步降级”实现。

### 5.5 comment 模块

文件：

- `comment_handler.go`
- `comment_service.go`
- `comment_repo.go`
- `comment_entity.go`

职责：

- 发布评论
- 删除评论
- 查询视频下全部评论

关键点：

- 发布评论和点赞类似，也走：
  - `comment.events`
  - `video.popularity.events`
- 如果 MQ 不可用，则同步写评论表并更新视频热度。

### 5.6 social 模块

文件：

- `social_handler.go`
- `social_service.go`
- `social_repo.go`
- `social_entity.go`

职责：

- 关注
- 取关
- 查询粉丝列表
- 查询关注列表

特点：

- 关注/取关会先尝试发 MQ
- 但当前实现里 API 线程仍然会直接操作 DB，所以 MQ 更像“冗余异步事件”而不是纯异步唯一写路径
- Worker 端对重复 follow 做了幂等处理

### 5.7 feed 模块

文件：

- `feed_handler.go`
- `feed_service.go`
- `feed_repo.go`
- `feed_entity.go`

职责：

- 最新视频流 `/feed/listLatest`
- 点赞数排序 `/feed/listLikesCount`
- 关注流 `/feed/listByFollowing`
- 热榜 `/feed/listByPopularity`

这是项目最复杂的读取模块，核心是“多级缓存 + 游标分页 + 热冷分离”。

#### `ListLatest`

- 热数据优先从 Redis ZSET `feed:global_timeline` 读取
- 冷数据回退 MySQL
- 如果全局时间线空了，会用 singleflight 触发一次重建

#### `GetVideoByIDs`

- L1：本地内存缓存 `go-cache`
- L2：Redis
- L3：MySQL
- 并用 `singleflight` 避免并发击穿

#### `ListByFollowing`

- 读 MySQL 关注关系和视频
- 用 Redis 做结果缓存
- 用分布式锁避免缓存击穿

#### `ListByPopularity`

- 优先聚合最近 60 个一分钟热度窗口的 ZSET
- 形成稳定分页快照 `as_of + offset`
- Redis 不可用时再回退数据库 `popularity DESC, create_time DESC, id DESC`

---

## 6. Worker 与异步链路

### 6.1 Worker 进程监听的消费者

`backend/cmd/worker/main.go` 启动后会注册：

- `SocialWorker`
- `LikeWorker`
- `CommentWorker`
- `PopularityWorker`

### 6.2 各 Worker 的职责

#### SocialWorker

- 消费 `social.events`
- 执行 follow / unfollow
- 对重复关注做幂等处理

#### LikeWorker

- 消费 `like.events`
- 真正创建或删除 `likes` 记录
- 更新 `videos.likes_count`
- 更新 `videos.popularity`

#### CommentWorker

- 消费 `comment.events`
- 创建 / 删除评论
- 发布评论时增加视频热度

#### PopularityWorker

- 消费 `video.popularity.events`
- 更新 Redis 中的热榜时间窗 ZSET
- 同时失效视频详情缓存

### 6.3 Outbox + Timeline

还有一条单独链路：

#### OutboxPoller

- 周期扫描 `outbox_msgs` 表里 `pending` 的记录
- 把“视频发布事件”投递到 `video.timeline.events`
- 成功后删除 Outbox 记录

#### Timeline Consumer

- 消费 `video.timeline.update.queue`
- 把新视频写入 Redis `feed:global_timeline`
- 保留最近 1000 条

这条链路解决了“新发布视频如何快速进入推荐最新流”的问题。

---

## 7. 前端架构

### 7.1 技术栈

- Vue 3
- Pinia
- Vue Router
- 原生 `fetch`
- Vite

### 7.2 前端层次

#### 1. `api/`

负责所有 HTTP 请求：

- `account.ts`
- `feed.ts`
- `video.ts`
- `like.ts`
- `comment.ts`
- `social.ts`

所有请求最终都走 `api/client.ts`：

- 自动拼接 `/api`
- 自动附带 JWT
- 统一处理错误
- 401 时自动清掉本地 token

#### 2. `stores/`

- `auth.ts`：JWT 的本地存储与 claims 解析
- `social.ts`：我的关注 / 粉丝关系状态
- `toast.ts`：全局提示

#### 3. `views/`

主要页面：

- `HomeView.vue`：推荐流、热门流、关注流，带短视频沉浸式滚动体验
- `HotView.vue`：热榜页
- `VideoDetailView.vue`：视频详情、点赞、评论、关注、分享
- `VideoView.vue`：上传视频并发布
- `AccountView.vue` / `RegisterView.vue` / `ChangePasswordView.vue`
- `UserProfileView.vue`：用户主页、作品、粉丝、关注

### 7.3 前后端通信方式

前端开发环境通过 `frontend/vite.config.ts` 把 `/api/*` 代理到 `http://127.0.0.1:8080/*`。

所以前端调用：

```text
/api/like/like
```

最终会打到后端：

```text
/like/like
```

---

## 8. 运行时主链路总结

### 8.1 登录链路

前端登录后：

1. 调 `POST /account/login`
2. 后端校验账号密码
3. 生成 JWT
4. 写入 MySQL `account.token`
5. 写入 Redis `account:{id}`
6. 前端把 token 存入 localStorage

之后前端所有请求由 `api/client.ts` 自动带上：

```text
Authorization: Bearer <token>
```

### 8.2 发布视频链路

1. 前端先上传封面
2. 再上传视频文件
3. 再调用 `/video/publish`
4. 后端事务写 `videos` 与 `outbox_msgs`
5. Outbox 轮询器发 MQ
6. Timeline Consumer 更新 Redis 全局时间线
7. `/feed/listLatest` 之后就能更快读到该视频

### 8.3 互动行为链路

点赞、评论、关注这些行为，都有类似特征：

- API 接口先做参数校验和身份校验
- 优先投递 MQ
- Worker 异步落库或更新缓存
- MQ 不可用时，再同步降级处理

---

## 9. 选取一个 API 做完整详细说明：`POST /like/like`

我选这个接口，是因为它最能代表本项目的真实复杂度：它不是单纯 insert 一条点赞记录，而是同时涉及前端状态、JWT、DB、MQ、Worker、Redis 热榜，以及读路径最终如何看到结果。

### 9.1 前端入口

这个接口的触发点主要出现在：

- `frontend/src/views/HomeView.vue`
- `frontend/src/views/HotView.vue`
- `frontend/src/views/VideoDetailView.vue`

以 `VideoDetailView.vue` 为例：

1. 用户点击点赞按钮
2. 触发 `toggleLike()`
3. 如果未登录，跳转登录页
4. 如果当前未点赞，调用 `likeApi.like(id)`
5. `frontend/src/api/like.ts` 会执行：

```ts
postJson('/like/like', { video_id: videoId }, { authRequired: true })
```

6. `frontend/src/api/client.ts` 自动：
   - 从 `auth` store 中取 token
   - 设置 `Authorization: Bearer xxx`
   - 把请求发送到 `/api/like/like`
7. Vite 代理把请求转发到后端 `/like/like`

### 9.2 后端路由接入

在 `backend/internal/http/router.go` 中：

- 创建 `LikeRepository`
- 创建 `LikeService`
- 创建 `LikeHandler`
- 注册路由组 `/like`
- 对 `/like/like` 挂 `JWTAuth`

所以该请求先进入：

```text
JWTAuth -> LikeHandler.Like -> LikeService.Like
```

### 9.3 JWT 中间件处理

`backend/internal/middleware/jwt/jwt.go`

处理步骤：

1. 读取 `Authorization` 请求头
2. 解析 Bearer Token
3. 调 `auth.ParseToken()` 校验 JWT 签名和过期时间
4. 用 claims 中的 `account_id` 拼出 Redis key：`account:{accountID}`
5. 优先查 Redis：
   - 命中且 token 相同 -> 通过
   - 不同 -> 认为 token 已撤销
6. Redis 不可用或未命中时，回退查 MySQL 账号表中的 `token`
7. 成功后向 Gin Context 注入：
   - `accountID`
   - `username`

于是后续 Handler 可以安全获取当前登录用户。

### 9.4 Handler 层

`backend/internal/video/like_handler.go`

`LikeHandler.Like()` 的工作：

1. 解析 JSON：

```json
{ "video_id": 123 }
```

2. 校验 `video_id > 0`
3. 从 Gin Context 取出 `accountID`
4. 组装 `video.Like{VideoID, AccountID}`
5. 调用 `LikeService.Like(ctx, like)`

Handler 本身只做“协议层”的工作，不碰复杂业务。

### 9.5 Service 层核心逻辑

`backend/internal/video/like_service.go`

`LikeService.Like()` 是整条链路最关键的地方。

#### 第一步：参数与业务前置校验

它会先检查：

1. `like` 不能是 nil
2. `video_id` 和 `account_id` 必须存在
3. 视频是否存在
4. 当前用户是否已经点过赞

只有通过这些校验，才会继续。

#### 第二步：优先走异步消息

Service 会尝试做两件事：

1. 发 `like.events`
   - 通过 `likeMQ.Like(ctx, accountID, videoID)`
   - 事件内容包括 `event_id / action / user_id / video_id / occurred_at`

2. 发 `video.popularity.events`
   - 通过 `popularityMQ.Update(ctx, videoID, 1)`
   - 表示视频热度 +1

如果这两个消息都成功投递，那么 API 立即返回成功，不在当前请求线程里直接写库。

这是这个系统“写请求快返回”的核心原因。

#### 第三步：如果 MQ 失败，走同步降级

如果 `like.events` 投递失败，Service 会直接开事务同步处理：

1. 再次确认视频存在
2. 插入 `likes`
3. `videos.likes_count + 1`
4. `videos.popularity + 1`

如果 `video.popularity.events` 投递失败，则直接同步更新 Redis 热榜：

1. 删除 `video:detail:id={id}` 详情缓存
2. 取当前 UTC 分钟窗口
3. 对 `hot:video:1m:{minute}` 执行 `ZINCRBY +1`
4. 设置窗口过期时间 2 小时

因此，这个接口即使 MQ 有问题，也不会完全不可用。

### 9.6 Like MQ 的消息内容

`backend/internal/middleware/rabbitmq/likeMQ.go`

点赞事件会被发布到：

- exchange：`like.events`
- routing key：`like.like`

事件大致结构：

```json
{
  "event_id": "...",
  "action": "like",
  "user_id": 1001,
  "video_id": 123,
  "occurred_at": "..."
}
```

同时，热度事件会被发布到：

- exchange：`video.popularity.events`
- routing key：`video.popularity.update`

### 9.7 Worker 如何消费点赞消息

Worker 进程启动时，`backend/cmd/worker/main.go` 会创建 `LikeWorker`。

`backend/internal/worker/likeworker.go` 中：

1. 持续消费队列 `like.events`
2. 收到消息后反序列化 `LikeEvent`
3. 根据 `action` 分发：
   - `like` -> `applyLike`
   - `unlike` -> `applyUnlike`

#### `applyLike` 的动作

1. 检查视频是否存在
2. 调 `LikeRepository.LikeIgnoreDuplicate`
   - 如果已点赞，视为幂等成功
3. `VideoRepository.ChangeLikesCount(videoID, +1)`
4. `VideoRepository.ChangePopularity(videoID, +1)`

也就是说，真正的点赞记录写入和计数更新，很可能是在 Worker 线程里完成的。

### 9.8 Worker 如何更新 Redis 热榜

`backend/internal/worker/popularityworker.go`

PopularityWorker 消费 `video.popularity.events` 后会调用：

```text
video.UpdatePopularityCache(ctx, cache, videoID, +1)
```

这个函数会：

1. 删除视频详情缓存 `video:detail:id={id}`
2. 找到当前一分钟窗口 `hot:video:1m:{minute}`
3. 对该视频执行 `ZINCRBY +1`
4. 给窗口设置 2 小时 TTL

这意味着“热榜”并不是直接查 MySQL 的 `popularity` 列，而是优先使用 Redis 的分钟级热度窗口聚合结果。

### 9.9 点赞后，哪些读接口会看到结果

#### 1. `/video/getDetail`

- 如果详情缓存被删掉
- 下次会回源 MySQL
- 能看到新的 `likes_count`

#### 2. `/feed/listLatest`

- 底层会批量取视频实体
- 最终从视频记录中读到新的 `likes_count`

#### 3. `/feed/listByPopularity`

- 优先读 Redis 热度窗口聚合
- 因为刚刚更新了 `hot:video:1m:*`
- 所以视频在热榜中的排序可能立刻变化

#### 4. `/like/isLiked`

- 直接查 `likes` 表
- Worker 落库后会返回 `true`

### 9.10 前端为什么能“立刻看到”点赞数变化

前端页面中，点赞成功后会做乐观更新：

- `item.is_liked = true`
- `item.likes_count += 1`

所以用户点击后立即看到 UI 变化，不必等待下一次重新拉取。

但真正的一致性仍由后端保证：

- MQ 正常时，最终由 Worker 异步落库
- MQ 故障时，由 Service 同步兜底

### 9.11 这条链路的本质

`POST /like/like` 实际是下面这条链：

```text
前端按钮点击
  -> frontend view
  -> frontend api client
  -> Vite /api 代理
  -> Gin Router
  -> JWT 中间件
  -> LikeHandler
  -> LikeService
      -> LikeMQ
      -> PopularityMQ
      -> (失败则同步降级)
  -> RabbitMQ
  -> LikeWorker 落 MySQL
  -> PopularityWorker 更新 Redis 热榜
  -> 后续 detail/feed/hot 接口读到新结果
```

这正是整个项目最典型的运行模式。

---

## 10. 项目架构的几个关键设计点

### 10.1 不是“所有请求都直接查库”

项目大量使用：

- Redis 缓存
- Redis ZSET 排序
- RabbitMQ 异步写
- Worker 解耦写扩散

说明作者在主动解决：

- 热点读
- 排序读
- 并发击穿
- 异步写扩散
- 中间件故障降级

### 10.2 读路径和写路径分离明显

写路径常见模式：

```text
API -> MQ -> Worker -> MySQL/Redis
```

读路径常见模式：

```text
API -> L1 本地缓存 -> L2 Redis -> L3 MySQL
```

### 10.3 Feed 是最核心的复杂模块

因为它同时处理：

- 最新流
- 关注流
- 热榜
- 点赞状态回填
- 分页游标
- 热冷分离
- 单飞锁

如果从“系统复杂度”看，`feed/service.go` 是整个项目最关键的中心代码。

### 10.4 发布视频链路用了 Outbox 思路

这说明项目不仅考虑“写视频成功”，还考虑：

- 如何把“视频发布事件”可靠传播到时间线
- 如何降低 DB 与 MQ 之间的不一致风险

这是比普通 Demo 更工程化的一点。

---

## 11. 一句话总结整个项目

这个项目的本质是：

> 一个以 Go 为后端、Vue 为前端，MySQL 为主存储，Redis 为缓存与排序层，RabbitMQ + Worker 为异步写扩散层的短视频 Feed 系统。

如果再进一步概括它的架构风格，可以说是：

> “经典三层业务架构 + 缓存层 + 消息队列 + Worker 异步处理 + Feed 场景专项优化”。

---

## 12. 我建议你接下来重点阅读的文件顺序

如果你想继续自己顺着代码深入，建议按这个顺序读：

1. `backend/internal/http/router.go`
2. `backend/cmd/main.go`
3. `backend/cmd/worker/main.go`
4. `backend/internal/feed/service.go`
5. `backend/internal/video/like_service.go`
6. `backend/internal/worker/likeworker.go`
7. `backend/internal/worker/outboxworker.go`
8. `frontend/src/views/HomeView.vue`
9. `frontend/src/views/VideoDetailView.vue`
10. `frontend/src/api/client.ts`

这样会最快建立全局视角。
