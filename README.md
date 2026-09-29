# 🎭 舞台演出提示编排台（Stage Cue Board）

面向舞台技术人员的提示（Cue）编排 Web 应用：在时间轴上创建**灯光 / 音效**提示，设置持续时长与
**“完成后触发”**依赖；拖动提示时按依赖拓扑**级联移动**后继项，遇到**锁定项**或**同轨时间冲突**时
整次操作自动**回滚**并明确标出阻塞链。支持按轨道筛选、撤销最近一次成功调整。

**演出现场打点**：技术人员可在时间轴上对提示标记 **▶ 开始 / ✔ 完成**，查看**待执行 / 执行中 / 已完成**
三种状态及**由服务端记录的实际时间**。只有全部前置提示已完成才能开始；完成必须发生在开始之后；
重复开始、重复完成或对已完成项“重开”都会被拒绝并说明原因，且不改变既有记录。提示一旦开始即冻结，
不可编辑/删除，级联移动与撤销只要波及已开始的提示也会整次拒绝。执行记录持久化在服务端，刷新页面依旧可见。

- 后端：Node.js + Express + SQLite（better-sqlite3，同步事务保证级联移动的原子性）
- 前端：原生 HTML / CSS / JavaScript，无框架、无构建步骤
- 数据库表结构初始化与示例数据播种在容器首次启动时自动完成（旧库启动时自动补齐打点字段）

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
| 现场打点 | 时间轴提示块右侧的 **▶**（待执行时）/ **✔**（执行中时）按钮，或详情面板中的“标记开始 / 标记完成”；时间以**服务端时钟**为准 |
| 执行状态 | 待执行（灰）→ 执行中（绿、脉动描边）→ 已完成（蓝、置灰划线）；详情面板显示实际开始/完成时间，刷新后仍在 |
| 执行冻结 | 已开始的提示不可编辑、删除、拖动；波及它的级联移动或撤销整次拒绝，阻塞项以红色标出 |

## 现场打点规则

- **状态机**：`pending`（待执行）→ `running`（执行中）→ `done`（已完成），只允许单向流转。
- **开始前置**：某提示的**全部“完成后触发”前置**均为 `done` 时才允许开始，否则 409 并在 `blockers`
  中列出未完成的前置（`type: "execution"`）。
- **完成时序**：完成必须晚于开始（时间由服务端记录）；未开始即完成会被拒绝。
- **幂等拒绝**：重复开始、对执行中/已完成项再次完成、对已完成项“重开”均返回 409
  （`type: "execution-state"`），响应中说明原因与既有时间，**记录不发生任何变化**。
- **冻结范围**：`running` / `done` 的提示不可 PATCH、DELETE、move；move 的级联闭包或 undo 的还原集合
  中一旦含此类提示，**整次操作拒绝**，阻塞项 `type: "started"`，其余提示仍按原有编排/锁定/冲突规则校验。
- **持久化**：状态与 `startedAt` / `completedAt`（epoch 毫秒）保存在 SQLite，随 `/api/state` 全量返回，
  也可通过 `GET /api/cues/:id/execution` 单独查询。

## 时间与冲突规则

- 所有时间以毫秒为单位存储，前端以秒展示；同轨上两段提示按半开区间 `[开始, 开始+时长)` 检测重叠。
- 依赖边 `A → B` 表示“**A 完成后触发 B**”，约束 `B.start >= A.start + A.duration`。
- 拖动 X 的级联集合 = {X} ∪ X 的全部后继（传递闭包），集合内统一平移相同的 Δ；
  校验顺序：①锁定项 ②时间轴下界（≥0）③同轨重叠 ④跨集合依赖约束。任一失败即事务回滚。

## 项目结构

```
server.js            Express 入口（静态资源 + /api 路由）
src/db.js            SQLite 连接、建表、首次启动播种示例数据
src/graph.js         依赖图：传递闭包、最短阻塞链、环检测、拓扑排序
src/routes.js        REST 接口与级联移动 / 撤销的事务逻辑
public/              原生前端（index.html / styles.css / app.js）
Dockerfile           多阶段构建（builder 内编译原生模块）
docker-compose.yml   一键启动，自动建库并挂载数据卷
```

## 主要 HTTP 接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/state` | 一次返回 tracks / cues / deps / canUndo |
| POST | `/api/cues` | 新建提示（含同轨冲突与依赖约束校验） |
| PATCH | `/api/cues/:id` | 编辑提示，`predecessorIds` 全量替换前置（含环检测） |
| DELETE | `/api/cues/:id` | 删除提示及关联依赖 |
| POST | `/api/cues/:id/move` | **核心**：body `{start}`，拓扑级联移动，409 时返回 `blockers` 阻塞链 |
| POST | `/api/undo` | 撤销最近一次成功移动，同样可能 409 返回阻塞链 |
| POST | `/api/tracks` | 新建轨道 |
| POST | `/api/cues/:id/start` | **现场打点**：标记开始（全部前置须已完成），成功返回最新 state |
| POST | `/api/cues/:id/complete` | **现场打点**：标记完成（须已开始），成功返回最新 state |
| GET | `/api/cues/:id/execution` | 查询执行记录：`{id, status, startedAt, completedAt}`（时间为 epoch 毫秒） |

`GET /api/state` 的每个 cue 在原有字段基础上**增量返回** `status`、`startedAt`、`completedAt`，
既有字段与响应结构保持不变。

冲突响应统一格式：`409 {"error": "...", "blockers": [{"type":"locked|started|track-conflict|dependency|cycle|execution|execution-state", "chain":[id...], "detail":"..."}]}`。
其中 `chain`（阻塞链）与 `blockerIds`（阻塞项）沿用原有两种写法，前端均可解析。

## HTTP API 调用示例（curl）

```bash
# 1. 查看整体状态（含每个提示的 status / startedAt / completedAt）
curl -s http://localhost:3000/api/state | jq '.cues[] | {id,name,status,startedAt,completedAt}'

# 2. 标记 id=1 的提示开始（若有前置未完成，返回 409 + blockers）
curl -s -X POST http://localhost:3000/api/cues/1/start -H 'Content-Type: application/json' -d '{}'

# 3. 标记完成（实际时间由服务端写入）
curl -s -X POST http://localhost:3000/api/cues/1/complete -H 'Content-Type: application/json' -d '{}'

# 4. 单独查询某条提示的执行记录
curl -s http://localhost:3000/api/cues/1/execution
# => {"id":1,"status":"done","startedAt":1727600000000,"completedAt":1727600012000}

# 5. 已开始后再尝试编辑 / 删除 / 移动 → 整次 409，记录不变
curl -s -X PATCH http://localhost:3000/api/cues/1 \
  -H 'Content-Type: application/json' -d '{"name":"新名字"}'
# => {"error":"提示「开场序曲」已开始，不能再编辑","blockers":[{"type":"started",...}]}

# 6. 前置未完成时开始后继（依赖 A→B，A 未完成）
curl -s -X POST http://localhost:3000/api/cues/2/start -d '{}'
# => 409，blockers[].type = "execution"

# 7. 重复完成 / 重开 → 409，说明原因且不改变记录
curl -s -X POST http://localhost:3000/api/cues/1/start  -d '{}'   # execution-state
curl -s -X POST http://localhost:3000/api/cues/1/complete -d '{}' # execution-state
```
