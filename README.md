# 经营指标语义层与自助分析平台

面向连锁零售集团的经营指标语义层与自助分析平台。解决总部、区域、门店三方对"本月销售额""同店增长"口径不一致的问题：通过集中语义定义、版本化运行、血缘下钻，让经营会上同名指标数字一致且可解释。

## 核心能力

- **集中语义定义**：指标用受控 DSL（表达式 AST）定义，表达来源字段、过滤、分组、聚合、四则组合、其他指标引用，**不接受任意 SQL**，也不只存一段口径说明
- **定义校验**：先识别引用环路，再校验金额、数量、比例、时间粒度；不同币种未经显式换算不能相加，门店日指标不能与门店月指标逐行相乘
- **运行引擎**：固定引用定义版本；标记数据截止点与覆盖范围，**不把未到数据当成零销售**；比率上卷时重新汇总分子分母
- **同店队列**：按两个比较期共同满足的营业状态与覆盖规则确定，跨期新开、闭店、缺资料门店各自说明，不把缺失期销售额填零后加入增长率
- **迟到退款**：影响计算生成新运行候选，已发布报表继续引用原运行，经营负责人决定是否发布更正
- **血缘下钻**：每个结果格可展开参与记录、过滤排除及最终算式

## 技术栈

- **后端**：Spring Boot 3.5 + Java 21 + Spring Data JPA + Flyway
- **前端**：Vue 3 + Vite + Element Plus
- **数据库**：PostgreSQL 16
- **部署**：Docker Compose 一键启动

## 快速开始

### 方式一：Docker Compose（推荐）

```bash
docker-compose up --build
```

启动后访问：
- 前端：http://localhost
- 后端 API：http://localhost:8080/api

### 方式二：本地开发

**前置要求**：Java 21+、Node 22+、PostgreSQL 16（库/用户/密码均为 `ccdev`）

```bash
# 1. 启动后端（自动建表 + 播种样例数据）
cd backend
mvn spring-boot:run

# 2. 启动前端（另开终端）
cd frontend
npm install
npm run dev
```

访问 http://localhost:5173（前端开发服务器，代理 API 到 8080）

> 也可以只启动后端：后端会托管前端构建产物（`backend/src/main/resources/static`），访问 http://localhost:8080

## 测试账号

| 账号 | 密码 | 角色 |
|------|------|------|
| admin | admin123 | 指标管理员 |
| analyst | analyst123 | 分析员 |
| manager | manager123 | 经营负责人 |

## 验证方式

### 一键验证脚本

```bash
# 确保后端已启动（localhost:8080）
bash scripts/verify.sh
```

覆盖：登录、币种校验、粒度校验、缺批次覆盖、跨营业日订单、迟到退款、同店队列。

### 关键场景手动验证

#### 1. 跨营业日订单
订单 O002 于 2026-01-10 01:15 下单（S001 营业到次日 02:00），计入 1/9 营业日。
```bash
# 运行 net_sales 2026-01-09（日粒度），S001 净销售额应为 800（O001 500 + O002 300）
curl -X POST http://localhost:8080/api/runs -H "X-Token: $TOKEN" -H "Content-Type: application/json" \
  -d '{"metricKey":"net_sales","metricVersionId":3,"from":"2026-01-09","to":"2026-01-09","grain":"day"}'
```

#### 2. 缺批次数据（不把未到数据当成零销售）
S003 门店 2/1-2/7 订单批次仍在补数（backfilling）。
```bash
# 运行 net_sales 2月（月粒度），S003 单元格状态应为 partial（部分覆盖）
curl -X POST http://localhost:8080/api/runs -H "X-Token: $TOKEN" -H "Content-Type: application/json" \
  -d '{"metricKey":"net_sales","metricVersionId":3,"from":"2026-02-01","to":"2026-02-28","grain":"month"}'
```

#### 3. 迟到退款
```bash
# 回放迟到退款（O001 的退款 150 元，1/10 营业日）
curl -X POST http://localhost:8080/api/replay/late-refund -H "X-Token: $TOKEN" -H "Content-Type: application/json" \
  -d '{"refundId":"R003","orderId":"O001","storeId":"S001","businessDate":"2026-01-10","amount":"150.00"}'
# 影响计算生成新运行候选；已发布报表继续引用原运行，经营负责人在报表中心发布更正
```

#### 4. 币种校验
```bash
# CNY 未过滤 + USD 直接相加，应报错"不同币种不能直接相加"
curl -X POST http://localhost:8080/api/metrics/validate -H "X-Token: $TOKEN" -H "Content-Type: application/json" \
  -d '{"metricKey":"t","definition":"{\"op\":\"arith\",\"operator\":\"add\",\"left\":{\"op\":\"agg\",\"measure\":\"sum\",\"expr\":{\"op\":\"field\",\"source\":\"orders\",\"field\":\"amount\"},\"by\":[\"store_id\"]},\"right\":{\"op\":\"agg\",\"measure\":\"sum\",\"expr\":{\"op\":\"field\",\"source\":\"orders\",\"field\":\"amount\"},\"filter\":[{\"source\":\"orders\",\"field\":\"currency\",\"operator\":\"eq\",\"value\":\"USD\"}],\"by\":[\"store_id\"]}}"}'
```

#### 5. 引用环路（并发发布）
创建两个相互引用的指标草稿，先发布一个成功，再发布另一个时因最终组合成环被拒绝（409）。

## 指标 DSL 说明

指标定义是 JSON 表达式树（AST），节点类型：

| 节点 | 说明 | 示例 |
|------|------|------|
| `field` | 取源字段 | `{"op":"field","source":"orders","field":"amount"}` |
| `const` | 常量 | `{"op":"const","value":0.05}` |
| `agg` | 聚合 | `{"op":"agg","measure":"sum","expr":{...},"filter":[...],"by":["store_id"]}` |
| `arith` | 四则运算 | `{"op":"arith","operator":"sub","left":{...},"right":{...}}` |
| `convert` | 币种换算 | `{"op":"convert","expr":{...},"fromCurrency":"USD","toCurrency":"CNY","rate":7.1}` |
| `ref` | 引用其他指标 | `{"op":"ref","metric":"gross_sales"}` |
| `cohort` | 同店增长 | `{"op":"cohort","base":{...},"compare":"prev_period","cohortKey":"store_id"}` |

**示例：净销售额**
```json
{
  "op": "arith", "operator": "sub",
  "left": {"op": "ref", "metric": "gross_sales"},
  "right": {"op": "ref", "metric": "refund_total"}
}
```

**校验规则**：
- 引用环路检测（DFS 三色标记）
- 金额/数量/比例类型检查
- 不同币种未经 `convert` 换算不能相加
- 门店日指标不能与门店月指标逐行相乘
- 金额与金额不能相乘（量纲错误）

## 样例数据

预置 5 家门店（S001-S005）、2026 年 1-3 月订单/退款/门店状态/节假日数据，覆盖：
- 跨营业日订单（O002，1/10 01:15 计入 1/9）
- 多商品订单（O003，3 个订单行）
- 多次部分退款（O005 → R001、R002）
- 临时闭店（S005，2/15-2/21）
- 跨期新店（S004，2/10 开业）
- 缺批次数据（S003，2/1-2/7 补数中）
- 外币订单（O004，USD，用于币种校验演示）

## 项目结构

```
├── backend/                 # Spring Boot 后端
│   ├── src/main/java/com/semantic/platform/
│   │   ├── dsl/            # 指标 DSL（表达式 AST、校验器、源注册表）
│   │   ├── engine/         # 运行引擎（求值器、覆盖、同店队列）
│   │   ├── metric/         # 指标服务（CRUD、发布、影响）
│   │   ├── run/            # 运行服务（执行、血缘、回放）
│   │   ├── report/         # 报表服务（发布、更正）
│   │   ├── approval/       # 审批服务
│   │   ├── source/         # 源数据与播种
│   │   └── api/            # REST 控制器
│   └── src/main/resources/
│       ├── db/migration/   # Flyway 建表脚本
│       └── static/         # 前端构建产物（后端托管）
├── frontend/               # Vue 3 前端
│   └── src/views/          # 指标目录/编辑器/运行/报表/审批/数据源/影响
├── scripts/verify.sh       # 端到端验证脚本
└── docker-compose.yml      # 一键部署
```

## 验证结论

本次交付实际执行过的验证：
- ✅ 后端编译通过（Spring Boot 3.5.6 + Java 21）
- ✅ 前端构建通过（Vue 3 + Vite）
- ✅ 端到端验证脚本 8/8 通过（登录、币种校验、粒度校验、缺批次覆盖、跨营业日订单、迟到退款、同店队列）
- ✅ 引用环路并发发布被拦截（cyc_a v2 发布成功，cyc_b v2 因组合成环返回 409）
- ✅ 迟到退款完整闭环（初始 1700 → 影响计算生成候选 → 报表仍引用原运行 → 发布更正后 1550）
