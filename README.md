# 🎭 舞台演出提示编排台（Stage Cue Board）

面向舞台技术人员的提示（Cue）编排 Web 应用：在时间轴上创建**灯光 / 音效**提示，设置持续时长与
**“完成后触发”**依赖；拖动提示时按依赖拓扑**级联移动**后继项，遇到**锁定项**或**同轨时间冲突**时
整次操作自动**回滚**并明确标出阻塞链。支持按轨道筛选、撤销最近一次成功调整。

演出现场可直接在时间轴上对提示**打点“开始 / 完成”**，提示随之在 **待执行 → 执行中 → 已完成**
三种状态间流转，实际开始/完成时间由**服务端时钟**记录、持久化保存；未完成全部前置的提示无法开始，
已开始的提示即冻结（不可编辑/删除/移动/撤销）。

- 后端：Node.js + Express + SQLite（better-sqlite3，同步事务保证级联移动的原子性）
- 前端：原生 HTML / CSS / JavaScript，无框架、无构建步骤
- 数据库表结构初始化与示例数据播种在容器首次启动时自动完成

## 🚀 一条命令启动（Docker Compose）

```bash
docker compose up --build -d
```

启动后浏览器访问：<http://localhost:3000>

数据持久化在 Docker 卷 `cue-data`（容器内 `/data/app.db`）中，停止后再次启动数据保留；
如需全新示例数据，执行 `docker compose down -v` 后重新启动即可。

> 无 Docker 环境时也可本地运行（自动完成依赖安装与数据库初始化）：
> `npm install && npm start`，同样访问 <http://localhost:3000>。

## 功能说明

| 功能 | 操作方式 |
| --- | --- |
| 新建提示 | 左栏填写名称、类型（灯光/音效）、轨道、开始时间、持续时长；可勾选“锁定” |
| 编辑 / 设依赖 | 点击时间轴上的提示，在右侧详情中修改，勾选“前置提示”即建立“完成后触发”边（不能成环） |
| 级联拖动 | 按住提示左右拖动，所有沿依赖边可达的后继项整体平移（按 0.25s 吸附） |
| 回滚与阻塞链 | 拖动路径上存在**锁定提示**、级联后**同轨时间重叠**、或跨越依赖边界违反“完成后触发”时，整体不写入；界面红色闪烁标出阻塞链，底部提示具体链条，如 `主角追光 → 跑灯序列 → 终场和弦` |
| 轨道筛选 | 顶部点击轨道名切换显示（仅影响展示，冲突检测始终覆盖全部轨道） |
| 撤销 | 顶部“↩ 撤销最近调整”按钮，将最近一次成功的级联移动整体还原（撤销本身也做冲突检测，被阻塞时保留记录） |
| 删除 | 详情面板删除提示，相关依赖边级联移除 |
| 演出打点 | 点击提示上的 **▶**（开始）/ **✅**（完成），或在右侧详情面板打点；提示块与详情中显示 ⏸ 待执行 / 🟢 执行中 / ✅ 已完成，以及服务端记录的**实际开始、实际完成时间**（刷新页面后仍在） |

## 时间与冲突规则

- 所有时间以毫秒为单位存储，前端以秒展示；同轨上两段提示按半开区间 `[开始, 开始+时长)` 检测重叠。
- 依赖边 `A → B` 表示“**A 完成后触发 B**”，约束 `B.start >= A.start + A.duration`。
- 拖动 X 的级联集合 = {X} ∪ X 的全部后继（传递闭包），集合内统一平移相同的 Δ；
  校验顺序：①锁定项 ②时间轴下界（≥0）③同轨重叠 ④跨集合依赖约束。任一失败即事务回滚。

## 演出执行（打点）规则

- 状态由服务端根据 `started_at / completed_at` 派生：两者皆空为 **待执行**；已开始未完成为
  **执行中**；两者皆有为 **已完成**。打点时间一律取**服务端时钟**（epoch 毫秒），前端按本地时区展示。
- **开始**：只有所有前置（“完成后触发”依赖的来源提示）均已**完成**，待执行提示才能开始。
  前置缺失时返回 409，`blockers` 逐条列出未完成前置链。
- **完成**：必须先开始；数据库以约束 `completed_at >= started_at` 保证完成发生在开始之后。
- **重复开始 / 重复完成 / 已完成后重开**：均返回 409 说明原因（含既有记录时间），**不改变任何记录**。
- **开始后冻结**：提示一旦开始便不可编辑、删除；拖动级联或撤销移动时，只要涉及集合内任一已开始
  （含已完成）的提示，**整次拒绝**并在 `blockers` 中用 `type: "started"` 指出具体阻塞项及链条。
  其他未开始提示仍遵循原有的锁定 / 同轨冲突 / 依赖边界编排规则。
- 执行记录与编排数据同在 SQLite（Docker 卷 `cue-data`），刷新页面或重启服务后仍可查看。
  v1 旧库首次启动时自动补齐 `started_at / completed_at` 两列（历史提示默认为待执行）。

## 项目结构

```
server.js            Express 入口（静态资源 + /api 路由）
src/db.js            SQLite 连接、建表、首次启动播种示例数据（含执行时间列的旧库迁移）
src/graph.js         依赖图：传递闭包、最短阻塞链、环检测、拓扑排序
src/routes.js        REST 接口与级联移动 / 撤销的事务逻辑
public/              原生前端（index.html / styles.css / app.js）
Dockerfile           多阶段构建（builder 内编译原生模块）
docker-compose.yml   一键启动，自动建库并挂载数据卷
```

## 主要 HTTP 接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/state` | 一次返回 tracks / cues / deps / canUndo；每个 cue 额外含 `status`（pending/running/completed）、`startedAt`、`completedAt`（服务端 epoch ms） |
| POST | `/api/cues` | 新建提示（含同轨冲突与依赖约束校验） |
| PATCH | `/api/cues/:id` | 编辑提示，`predecessorIds` 全量替换前置（含环检测）；**已开始的提示返回 409** |
| DELETE | `/api/cues/:id` | 删除提示及关联依赖；**已开始的提示返回 409** |
| POST | `/api/cues/:id/move` | **核心**：body `{start}`，拓扑级联移动，409 时返回 `blockers` 阻塞链；级联集合含已开始提示时整次拒绝 |
| POST | `/api/cues/:id/start` | **打点开始**：前置全部完成才允许，服务端记录 `startedAt` |
| POST | `/api/cues/:id/complete` | **打点完成**：仅执行中提示可调用，服务端记录 `completedAt` |
| POST | `/api/undo` | 撤销最近一次成功移动，同样可能 409 返回阻塞链；涉及已开始提示时整次拒绝且记录保留 |
| POST | `/api/tracks` | 新建轨道 |

冲突响应统一格式：`409 {"error": "...", "blockers": [{"type":"locked|started|track-conflict|dependency|cycle|execution", "chain":[id...], "detail":"..."}]}`。
打点类拒绝（重复开始、重复完成、重开、先完成后开始）使用 `type: "execution"`；
冻结类拒绝（编辑/删除/移动/撤销触及已开始提示）使用 `type: "started"`。
打点接口成功时与其它写接口一样返回完整 `state()`，便于调用方直接刷新。

### 打点接口调用示例（curl）

```bash
# 查看全部状态（status / startedAt / completedAt）
curl -s http://localhost:3000/api/state | jq '.cues[] | {id,name,status,startedAt,completedAt}'

# 开始 #4（前置未全部完成时会得到 409，blockers 指出缺哪条前置）
curl -s -X POST http://localhost:3000/api/cues/4/start \
  -H 'Content-Type: application/json' -d '{}'

# 完成 #4（必须已开始；重复完成 / 重开均 409 且记录不变）
curl -s -X POST http://localhost:3000/api/cues/4/complete \
  -H 'Content-Type: application/json' -d '{}'
```

典型 409 响应：

```json
{
  "error": "提示「主角追光」尚有未完成的前置提示，不能开始",
  "blockers": [
    { "type": "dependency", "chain": [2, 3], "detail": "前置提示「场灯渐暗」尚未开始，本提示不能开始" }
  ]
}
```

```bash
# 移动已开始提示（整次回滚，blockers 标出每一个已开始的阻塞项）
curl -s -X POST http://localhost:3000/api/cues/1/move \
  -H 'Content-Type: application/json' -d '{"start": 1}'
# {"error":"移动回滚：级联范围内存在已开始的提示","blockers":[{"type":"started","chain":[1],...}]}
```
