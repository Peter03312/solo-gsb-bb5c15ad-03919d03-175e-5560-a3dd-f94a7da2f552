# 水质实验室 · 样本分装与混样溯源工作台

基于 **Node.js + Express + SQLite（better-sqlite3）** 的 Web 工作台，用于管理水质样本的
原瓶登记、分装 / 混样操作、**批次预演与两阶段提交**、余量守恒校验、完整来源谱系追溯、
**更正预演与凭据确认** 及历史操作更正。

## 一键启动（Docker Compose）

```bash
docker compose up --build
```

容器启动时会自动安装依赖、建表；数据库为空时写入示例数据（3 个原始瓶 + 2 条操作）。

浏览器访问：**http://localhost:3000**

停止并删除数据：`docker compose down -v`（数据保存在 named volume `lab-data` 中，普通 `down` 不丢数据）。

## 配置项（环境变量）

| 变量 | 默认值 | 说明 |
|---|---|---|
| `PORT` | `3000` | HTTP 监听端口（Compose 已映射 3000:3000） |
| `DB_PATH` | 容器内 `/app/data/lab.db`，本地 `./data/lab.db` | SQLite 数据库文件路径，启动时自动建目录、建表、初始化 |

## 本地运行（不用 Docker）

```bash
npm install
npm start          # 访问 http://localhost:3000
```

要求 Node.js ≥ 18。better-sqlite3 为原生模块，标准 node 镜像 / 常见桌面环境均可直接安装。

## 页面功能

- **瓶子总览**：每个瓶的类型（原始瓶 / 衍生瓶）、创建或采样时间、初始体积、当前余量；
  点击任意行查看该瓶的**完整来源谱系**（向上递归至原始瓶）、**各原始瓶的精确贡献量（分数 mL）**
  及相关操作记录。
- **登记原始瓶**：录入瓶编号、采样时间、初始体积（mL，正整数）。
- **新建分装 / 混样**：录入唯一操作号、目标瓶编号（须为新编号）、操作时间、
  一个或多个来源瓶及各自取用体积；目标瓶体积 = 取用量之和。
- **批量预演 / 提交**：把连续分装 / 混样作为一个批次处理。录入批次号与按录入顺序排列的
  多条操作（每条沿用操作号、目标瓶、操作时间、来源瓶与取用毫升数），先**预演**查看逐条可行性、
  受影响瓶预计余量与提交凭据（不写入数据），再凭凭据**提交**整批；批内可引用前项生成的瓶。
- **操作记录与更正**：列出全部操作（含所属批次号）；点“更正”可修改该操作的来源与取用量。
  支持**预演 → 确认**两阶段：先预演查看差异（不写库）——因重放而变化的瓶余量、目标瓶及
  下游瓶更正前后的原瓶精确贡献（仅列变化项）——再凭确认凭据原子应用，结果与预演一致；
  也可选择“立即更正”直接提交（沿用原接口，导致后续超用将整体回滚）。

## 批次处理（本轮新增）

实验室把一批连续分装 / 混样作为一个批次，分两步完成：

1. **预演** `POST /api/batches/preview`：只校验与推演，**不写入任何数据**。返回逐条可行性、
   受影响瓶的预计余量，以及一枚**提交凭据**（绑定本批计划内容与当前台账版本）。
2. **提交** `POST /api/batches/commit`：在**单个事务**中核对凭据、重跑全部校验并落库，
   随后重放全量台账确认任一时刻无超用；任一环节失败**整批回滚，不落库**。

核算规则与单条操作一致：按（操作时间, 入库顺序）重放；**同一操作时间内，批内按录入顺序**；
来源瓶可引用**批内录入顺序在前**的条目生成的瓶；操作时间不得早于来源瓶创建时间；
任一时刻任何瓶不得超用。

- **幂等**：已成功提交的批次，相同批次号 + 相同内容重复提交 → HTTP 200 幂等返回，不重复扣量；
  相同批次号 + 不同内容 → 409 `BATCH_CONFLICT`。
- **凭据失效**：预演之后若台账发生任何变化（新登记原瓶 / 单条操作 / 更正 / 其它批次提交），
  或提交内容与预演计划不一致，提交将被拒绝（409 `STALE_TOKEN` / `PLAN_CHANGED`），需**重新预演**。
- **逐条报错**：编号冲突（批内或与现有台账）、重复来源、非法体积、缺失来源、超用等，
  均在响应中指出具体条目序号与操作号。

### 调用示例

```bash
# 1) 预演（不写库）
curl -s http://localhost:3000/api/batches/preview -H 'Content-Type: application/json' -d '{
  "batch_id": "BATCH-0001",
  "items": [
    {"op_id":"OP-0101","target_bottle":"MIX-101","op_time":"2026-10-04T09:00",
     "sources":[{"bottle":"W2026-001","volume":200},{"bottle":"W2026-002","volume":100}]},
    {"op_id":"OP-0102","target_bottle":"ALQ-101","op_time":"2026-10-04T10:00",
     "sources":[{"bottle":"MIX-101","volume":150}]}
  ]}'
# → {"ok":true,"items":[{"index":0,"op_id":"OP-0101","ok":true,"errors":[]},...],
#    "projected_balances":[{"bottle":"MIX-101","in_batch":true,"remaining":150},...],
#    "commit_token":"v1.3.<plan_hash>.<hmac>", ...}

# 2) 提交（携带预演返回的 commit_token，单事务整批落库）
curl -s http://localhost:3000/api/batches/commit -H 'Content-Type: application/json' -d '{
  "batch_id": "BATCH-0001",
  "items": [ …与预演完全相同的条目… ],
  "commit_token": "v1.3.<plan_hash>.<hmac>"
}'
# → 201 {"ok":true,"committed":2,"projected_balances":[...]}
#    重复提交相同内容 → 200 {"ok":true,"idempotent":true,...}

# 3) 查询批次
curl -s http://localhost:3000/api/batches            # 全部已提交批次
curl -s http://localhost:3000/api/batches/BATCH-0001 # 批次详情（按录入顺序的条目）
```

提交成功后，批内操作立即进入现有的余量（`GET /api/bottles`）、操作记录
（`GET /api/operations`，含 `batch_id` 字段）与谱系查询（`GET /api/bottles/:id`）。

## 原瓶贡献量查询（本轮新增）

按瓶编号查看**衍生瓶由各原始瓶贡献的精确体积**（工作台瓶详情同步展示）：

```bash
curl -s http://localhost:3000/api/bottles/ALQ-001/contributions
# 瓶详情接口 GET /api/bottles/:id 的响应也追加了同结构的 contributions 字段
```

```jsonc
{
  "bottle": "ALQ-001",
  "kind": "derived",
  "created_by_op": "OP-0002",
  "formation_volume": 100,                 // 目标瓶形成时的总体积（mL，整数）
  "contributions": [                       // 按原瓶编号排序；同一原瓶多路径汇入已合并
    {"bottle": "W2026-001", "contribution_ml": {"numerator": "400", "denominator": "7",
                                                "text": "400/7", "approx_ml": 57.142857142857146}},
    {"bottle": "W2026-002", "contribution_ml": {"numerator": "300", "denominator": "7",
                                                "text": "300/7", "approx_ml": 42.857142857142854}}
  ],
  "sum_ml": {"numerator": "100", "denominator": "1", "text": "100", "approx_ml": 100}
}
```

- **分摊规则**：原瓶贡献 = 自身初始体积；分装 / 混样从来源瓶 S 取 v mL 时，视 S 内容均匀，
  S 中各原瓶贡献按 **v / V_S**（V_S 为 S 形成时的总体积）分摊；同一原瓶经不同路径汇入须合并。
- **精确分数**：全程 BigInt 有理数运算并在每一步约分，`numerator / denominator` 为约分后的
  精确值（字符串承载任意精度），`approx_ml` 仅供展示；**不做逐层四舍五入**，
  `sum_ml` 严格等于 `formation_volume`。
- **与余量无关**：计算只依赖瓶子形成图（原瓶初始体积 + 各形成操作的来源与取用量），
  来源瓶之后的剩余量变化不改变已形成瓶子的配比。
- **更正即重算**：历史来源或取用量更正成功后，查询依据当前已提交台账重建形成图，
  所有下游贡献自动重算；**被现有守恒规则拒绝的更正已回滚，形成图不变，查询结果不变**。
- 原始瓶 / 衍生瓶均可查询（原瓶返回自身初始体积一条贡献）；瓶不存在 →
  404 `BOTTLE_NOT_FOUND`。

### 批次预演中的预计贡献

`POST /api/batches/preview` 响应新增 **`projected_contributions`** 字段：在逐条可行性之外，
为每个**可行条目**给出其目标瓶的预计原瓶贡献（结构同上，含 `index / op_id / target_bottle`），
批内引用前项生成的瓶同样适用（把本批条目作为虚拟形成图叠加到当前台账）。
预演仍不写库；提交后用 `GET /api/bottles/:id/contributions` 查到的结果与预演逐项一致。
引用了谱系不完整前项（如前项因来源不存在而不可行）的条目跳过预计贡献，不影响其余条目；
台账变化导致凭据失效仍按既有 `STALE_TOKEN` / `PLAN_CHANGED` 规则处理。

## 更正预演与确认（本轮新增）

更正历史分装 / 混样操作前，可先**预演**该更正对当前台账的影响，确认无误后再**凭据确认**、
原子应用；预演不写库，确认结果与预演一致。

1. **预演** `POST /api/operations/:op_id/correct/preview`：提交操作号与拟替换的来源瓶、
   取用毫升数，**不写入任何数据**。返回：
   - `balance_changes`：因重放而**变化**的瓶余量（`bottle / before / after`，无变化项不列出）；
   - `contribution_changes`：**目标瓶及其下游瓶**（形成图中传递依赖目标瓶的衍生瓶）更正前后的
     原瓶精确贡献对照（`before / after`，沿用约分分数 mL 格式，仅列变化项）；
   - `correction_token`：确认凭据（HMAC 签名，绑定 更正计划哈希 + 当前台账版本）。
2. **确认** `POST /api/operations/:op_id/correct/confirm`：携带与预演**完全一致**的
   `sources` 和 `correction_token`，在**单个事务**中核对凭据、重跑校验与差异计算、
   替换来源并全账重放复核，原子应用更正。

规则与约定：

- **预演不通过不发凭据**：操作号不存在（404 `OP_NOT_FOUND`）、来源瓶不存在
  （400 `SOURCE_NOT_FOUND`）、来源重复（400 `DUP_SOURCE`）、体积非正整数（400 `BAD_VOLUME`）、
  操作时间早于来源创建时间（400 `TIME_BEFORE_SOURCE`）、重放导致任一操作超用
  （409 `CORRECTION_OVERDRAW`，`details` 逐条给出冲突的 **操作号、瓶、需取量、可用量**）。
- **确认只接受原预演内容与未变化的台账版本**：内容被改动 → 409 `PLAN_CHANGED`；
  预演后有新登记 / 操作 / 更正 / 批次提交 → 409 `STALE_TOKEN`；凭据缺失或无效 →
  400 `TOKEN_REQUIRED` / 409 `BAD_TOKEN`。均需**重新预演**。
- **凭据幂等**：已确认的同一凭据重试 → 返回**首次结果**（`idempotent: true`），不重复更正；
  确认记录持久化于 `correction_confirmations` 表。
- **无变化即空差异**：若更正不改变某瓶余量或某瓶贡献（如等效替换），该项不出现在差异列表中。
- 现有**立即更正** `POST /api/operations/:op_id/correct` 与**批次预演 / 提交**约定继续可用，
  互不影响（任何写成功都会使未确认的更正凭据失效）。

### 调用示例

```bash
# 1) 预演（不写库）：拟把 OP-0001 的来源改为 W2026-001×100 + W2026-003×120
curl -s http://localhost:3000/api/operations/OP-0001/correct/preview \
  -H 'Content-Type: application/json' -d '{
    "sources": [{"bottle":"W2026-001","volume":100},{"bottle":"W2026-003","volume":120}]
  }'
# → {"ok":true,"ledger_version":1,
#    "balance_changes":[{"bottle":"MIX-001","before":250,"after":120},
#                       {"bottle":"W2026-001","before":800,"after":900}, ...],
#    "contribution_changes":[
#      {"bottle":"MIX-001","is_target":true,
#        "before":{"formation_volume":350,"contributions":[{"bottle":"W2026-001",
#          "contribution_ml":{"numerator":"200","denominator":"1","text":"200",...}}, ...], ...},
#        "after":{"formation_volume":220,"contributions":[...], ...}},
#      {"bottle":"ALQ-001","is_target":false,   ← 下游瓶
#        "before":{... "text":"400/7" ...}, "after":{... "text":"500/11" ...}}],
#    "correction_token":"c1.1.<plan_hash>.<hmac>", ...}

# 2) 确认（与预演完全相同的 sources + 凭据，单事务原子应用）
curl -s http://localhost:3000/api/operations/OP-0001/correct/confirm \
  -H 'Content-Type: application/json' -d '{
    "sources": [{"bottle":"W2026-001","volume":100},{"bottle":"W2026-003","volume":120}],
    "correction_token": "c1.1.<plan_hash>.<hmac>"
  }'
# → {"ok":true,"idempotent":false,"balance_changes":[...与预演一致...],
#    "contribution_changes":[...与预演一致...], ...}
#    同一凭据重试 → {"ok":true,"idempotent":true, ...首次结果...}，不重复更正

# 3) 预演失败示例：更正导致下游操作超用
curl -s http://localhost:3000/api/operations/OP-0001/correct/preview \
  -H 'Content-Type: application/json' -d '{"sources":[{"bottle":"W2026-003","volume":60}]}'
# → 409 {"error":{"code":"CORRECTION_OVERDRAW","message":"更正将导致以下操作超用…",
#        "details":[{"op_id":"OP-0002","bottle":"MIX-001","requested":100,"available":60}]}}
#    （无 correction_token）
```

确认成功后，相关瓶余量（`GET /api/bottles`）、谱系与原瓶贡献
（`GET /api/bottles/:id/contributions`）即为更正后状态，与预演差异逐项一致。

## 业务规则与一致性设计

- **唯一操作号 + 幂等**：`op_id` 为主键。同一操作号重复提交**相同内容**时幂等返回
  （HTTP 200，不重复扣量）；内容不一致返回 409 冲突。
- **来源约束**：同一操作内来源瓶不得重复；取用体积必须为正整数；来源瓶必须已存在；
  操作时间不得早于任一来源瓶的创建（采样）时间。
- **任一时刻不超用**：每次写入（新操作 / 更正）后，在事务内按
  （操作时间, 入库顺序）重放全量台账，任一瓶余量为负即整体回滚，
  返回 409 及冲突明细（操作号、瓶、需取量、可用量）。
- **历史更正**：更正取用量后按操作顺序重算所有受影响瓶；若导致后续操作超用，
  **整次更正拒绝并回滚，原数据保持不变**，响应中指出冲突记录。更正亦可走
  **预演 → 确认**两阶段：预演在内存中替换来源后模拟重放并对比形成图，返回余量差异、
  目标瓶及下游瓶的原瓶贡献前后对照与确认凭据（HMAC(更正计划哈希 + 台账版本)，`c1` 前缀，
  与批次凭据 `v1` 隔离）；确认在单事务中完成 凭据核对 → 校验与差异重算 → 落库 → 全账重放，
  已确认凭据持久化，重试幂等返回首次结果。
- **批次两阶段提交**：预演（dry-run）与提交分离。预演在内存中把批次条目追加到当前台账
  末尾模拟重放（虚拟入库顺序与提交时的真实分配一致），返回逐条可行性与预计余量；
  提交凭据 = HMAC(计划哈希 + 台账版本)，台账每次写成功版本递增，凭据随之失效。
  提交在单事务中完成 凭据核对 → 业务校验 → 落库 → 全账重放，失败即整体回滚。
- **并发守恒**：better-sqlite3 为同步驱动，所有写操作包裹在事务中串行执行，
  并发争用同一来源瓶时不会出现超扣；已用并发压测验证（10 并发争用 400 mL，
  恰好 4 笔成功、余量归零不为负；5 并发重复提交同一操作号仅扣量一次；
  5 并发重复提交同一批次仅扣量一次，其余幂等返回）。
- **明确报错**：编号冲突（409）、来源不存在（400）、操作时间早于来源创建时间（400）、
  体积非正整数（400）、超用（409）等均以 JSON `error.code / message / details` 返回，
  页面顶部同步展示。

## API 摘要

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/api/bottles` | 登记原始瓶 `{bottle_id, sample_time, initial_volume}` |
| `POST` | `/api/operations` | 新建操作 `{op_id, target_bottle, op_time, sources:[{bottle, volume}]}` |
| `POST` | `/api/operations/:op_id/correct` | 立即更正取用量 `{sources:[{bottle, volume}]}` |
| `POST` | `/api/operations/:op_id/correct/preview` | **（新增）** 更正预演（不写库）`{sources}` → 余量差异 `balance_changes` + 目标瓶及下游瓶贡献前后对照 `contribution_changes` + `correction_token` |
| `POST` | `/api/operations/:op_id/correct/confirm` | **（新增）** 更正确认（单事务）`{sources, correction_token}` → 应用成功 / 幂等返回首次结果 / 冲突 409 |
| `POST` | `/api/batches/preview` | 批次预演（不写库）`{batch_id, items:[{op_id, target_bottle, op_time, sources}]}` → 逐条可行性 + 预计余量 + 可行条目目标瓶的预计原瓶贡献 `projected_contributions` + `commit_token` |
| `POST` | `/api/batches/commit` | 批次提交（单事务）`{batch_id, items, commit_token}` → 201 / 幂等 200 / 冲突 409 |
| `GET`  | `/api/batches` | 全部已提交批次 |
| `GET`  | `/api/batches/:batch_id` | 批次详情（按录入顺序的条目及来源） |
| `GET` | `/api/bottles` | 全部瓶及当前余量 |
| `GET` | `/api/bottles/:id` | 单瓶详情：余量、来源谱系、相关操作、原瓶精确贡献量 `contributions`（新增字段，不影响既有字段） |
| `GET` | `/api/bottles/:id/contributions` | **（新增）** 各原始瓶对该瓶的精确贡献量：原瓶编号、约分分数 mL、形成时总体积与合计；瓶不存在 404 |
| `GET` | `/api/operations` | 全部操作记录（含来源明细、目标体积与所属批次号 `batch_id`） |

时间格式统一为 `YYYY-MM-DDTHH:MM`（页面用 `datetime-local` 控件），体积单位为毫升（正整数）。

## 项目结构

```
├── Dockerfile            # node:20，构建时安装依赖，启动时确保依赖并初始化数据
├── docker-compose.yml    # 一键启动，映射 3000 端口，named volume 持久化数据
├── package.json
├── server.js             # Express 服务 + SQLite 台账 + 守恒校验 + 谱系查询 + 批次预演/提交 + 原瓶贡献量（精确分数）+ 更正预演/确认
└── public/index.html     # 单页工作台（总览 / 登记 / 操作 / 批量预演提交 / 记录与更正（预演差异+凭据确认）/ 贡献量展示）
```
