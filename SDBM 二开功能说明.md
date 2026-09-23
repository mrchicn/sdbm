# SDBM 二开功能说明

> 基于上游 [Archery](https://github.com/hhyo/Archery) 的二次开发文档。  
> 截图环境：`http://192.xx.xx.xx:9123`（账号 `sdbm`），仅展示说明，**未改任何运行配置**。  
> 截图工具：ego-lite（`ego-browser`）。图中红色框为二开标注。  
> 整理日期：2026-09-23  
> 仓库：[mrchicn/sdbm](https://github.com/mrchicn/sdbm)

---

## 目录

1. [与 Archery 对比总览](#1-与-archery-对比总览)
2. [品牌与登录](#2-品牌与登录)
3. [SQL 提交体验（快捷窗口 / 文件清空）](#3-sql-提交体验快捷窗口--文件清空)
4. [工单状态实时刷新](#4-工单状态实时刷新)
5. [SQL 审核与 JSON 字面量预检](#5-sql-审核与-json-字面量预检)
6. [Lark 协作与催办](#6-lark-协作与催办)
7. [系统配置分区（登录 / SQL审批通知）](#7-系统配置分区登录--sql审批通知)
8. [用户类型与 Submitter 权限组](#8-用户类型与-submitter-权限组)
9. [数据源引擎裁剪](#9-数据源引擎裁剪)
10. [前端 Vue 化与其它运维体验](#10-前端-vue-化与其它运维体验)
11. [功能一览表](#11-功能一览表)

---

## 1. 与 Archery 对比总览

上游：[hhyo/Archery](https://github.com/hhyo/Archery)。下表聚焦 SDBM **相对上游的增量与取舍**（不是重复列 Archery 已有的查询/审核/执行能力）。

| 维度 | Archery（上游） | SDBM（本二开） | 便捷点 |
|------|-----------------|---------------|--------|
| 品牌 / 登录 | Archery；本地 / LDAP / 钉钉等 | **SDBM**；本地 + **Lark / 飞书** Tab，可配置默认优先 SSO | 国际版 Lark 与国内飞书分通道，登录页一眼切换 |
| 执行时间窗 | 手填起止时间 | 快捷 **「1 天」**（此刻～+24h 北京时间）+ 清空 | 减少填错窗口、避免短于 60 分钟 |
| SQL 文件上传 | 选文件后不易一键还原 | 右侧 **×** 清空文件名，并**同步清空编辑器** | 换文件 / 改手写时不用手动全选删除 |
| 工单状态 | 详情页需手动刷新 | 活跃状态 **2.5s 轮询**，变更自动整页刷新；切回标签立即再查 | 审批/执行中不用狂点刷新 |
| SQL 字面量 | 主要依赖 goInception | **JSON_PRECHECK** 默认开启，非法 JSON 转义短路线号报错可复制 | 脏 JSON 在送 inception 前就被拦住 |
| 审批通知 | 钉钉 / 企微 / 邮件等混在系统配置 | **独立「SQL审批通知」菜单** + Lark 卡片 | 配置职责清晰，改通知不影响登录项 |
| 催办 | 无周期催审 | **Lark 周期催办**（开关 + 间隔分钟；引用回复绕开卡片去重） | 待审超时自动提醒当前审批组 |
| 登录配置 | 与其它开关挤在一页 | **独立「登录配置」**（方式多选、凭证测试、人员日同步） | 保存登录不会误清空通知配置 |
| 用户身份 | 无 Lark/飞书类型列 | Admin **用户类型**徽章 + 按类型筛选 | 一眼区分本地 / Lark / 飞书，便于通知排障 |
| 权限种子 | Default / RD / DBA 等 | 新增 **Submitter**（可提交+查看组内工单，不可审批/执行） | 外包/只读提交角色开箱即用 |
| 权限组运维 | 手工重建 | **一键复制**权限组（不含成员）+ 描述必填 | 多业务线同权不同人快速克隆 |
| 数据源类型 | 十余种（Oracle/Redis/MsSQL/ES…） | **仅** MySQL / PgSQL / Doris / Mongo / ClickHouse / goInception | 界面更干净，镜像更瘦 |
| 前端主路径 | jQuery + Bootstrap 为主 | 工单/提交/查询/实例等 **Vue 化** | 交互与暗色主题统一 |
| 2FA / AI | 短信 2FA、OpenAI 生成 SQL 等 | 去掉短信与 OpenAI，保留 TOTP | 依赖更少、攻击面更小 |

---

## 2. 品牌与登录

### 2.1 全仓更名 Archery → SDBM

- 产品名：**SDBM / Simple Database Management**。
- 默认管理员：`sdbm`；静态资源、配置键、容器路径已从 archery 迁出。
- 导航 GitHub 指向 [mrchicn/sdbm](https://github.com/mrchicn/sdbm)。

### 2.2 本地用户 + Lark 双登录

登录页 Tab：**本地用户** / **Lark SSO**。默认可优先展示 Lark（由登录配置决定）。

![登录页默认 Lark](images/01-login-lark-default.png)

![本地用户登录 Tab](images/02-login-local-tab.png)

### 2.3 公告横幅与超管标识

- 标题后缀、顶部公告可配置。
- 超管右上角显示：`你好，sdbm（超管）`。

---

## 3. SQL 提交体验（快捷窗口 / 文件清空）

路径：`/submitsql/`（Vue 提交页）。

### 3.1 执行范围快捷「1 天」

- 按钮位置：可执行时间范围标题旁。
- 行为：填入**当前北京时间** → **+24 小时**；高亮表示已是约一天窗口。
- 另有「清空」；系统仍要求窗口 **≥ 60 分钟**。

**相对 Archery**：上游需手工点选日期时间；SDBM 一键填常用「一天内可执行」窗口，降低漏填/填反。

![「1 天」快捷按钮](images/22-submit-1day-only.png)

### 3.2 上传后点 × 清空 SQL 编辑栏

- 选择 `.sql` 后显示文件名与 **×**。
- 点 ×：清除文件选择，并 **`clear` 事件联动清空左侧编辑器内容**（避免旧文件 SQL 残留）。

**相对 Archery**：上游选错文件后常需手动清空编辑器；SDBM 一键还原。

![文件 × 清空](images/23-submit-fileclear-x.png)

![1 天 + 文件清空同屏](images/21-submit-1day-and-fileclear.png)

![SQL 提交总览](images/04-sql-submit.png)

![提交页标注（检测等）](images/19-sql-submit-annotated.png)

---

## 4. 工单状态实时刷新

路径：工单详情 `/detail/<id>/`。

- 活跃状态集合：等待审核、审核通过、定时执行、排队中、执行中。
- 前台轮询 `/getWorkflowStatus/`：**前台页约 2.5s**，标签页隐藏时约 10s；状态变化则 **整页 reload**（按钮/日志与模板一致）。
- `visibilitychange`：切回标签页立即再查。

**相对 Archery**：上游详情多为静态展示，执行进度需手动刷新；SDBM 适合盯审批与执行过程。

![工单状态实时轮询](images/34-workflow-status-poll.png)

---

## 5. SQL 审核与 JSON 字面量预检

### 5.1 JSON_PRECHECK_ENABLED（默认开启）

系统配置 → goInception 区域：

| 配置项 | 说明 |
|--------|------|
| `JSON_PRECHECK_ENABLED`（`enable_json_precheck`） | 是否开启 SQL 字面量 JSON 预检，**缺省视为 true** |
| `JSON_PRECHECK_MIN_LENGTH` | 字面量长度阈值（默认 200），超过才试解析 |

- 审核阶段拦截非法 JSON 转义（如 URL 未写成 `http:\/\/`）。
- 失败带**行号/列号**与上下文；弹窗可**复制**出错片段。
- 预检失败则**不送** goInception，节省往返。

**相对 Archery**：上游无此层短检；脏 JSON 往往拖到 inception 才暴露，排障成本更高。

![JSON 预检配置](images/24-config-json-precheck.png)

### 5.2 SQL 检测按钮

提交页「SQL检测」→ `/api/v1/workflow/sqlcheck/`；MySQL 侧由 goInception 审核。

---

## 6. Lark 协作与催办

| 能力 | 说明 |
|------|------|
| Lark 登录 | SSO：`/lark/login/` |
| 工单卡片 | 待审等阶段发 Lark 卡片（操作按钮） |
| **周期催办** | 超时后以**文本引用回复**首次通知催办，避开卡片去重发不出 |
| 催办标题 | 含催办次数火焰标记；**发送成功才 +1** |
| 截止 | 超过工单 `run_date_end` 不再催；无结束时间不催 |

配置入口：**系统管理 → SQL审批通知**（`/config/?value=3`）

- `LARK_REMIND_ENABLED`：催办总开关（默认关；图示环境已开仅作展示）。
- `LARK_REMIND_INTERVAL_MINUTES`：间隔分钟，改后**下次分钟级扫描即生效**。

**相对 Archery**：上游无 Lark 国际版通道与周期催审；待审依赖人工跟催。

![催办开关与间隔](images/28-config-lark-remind.png)

![SQL审批通知页](images/27-config-sql-notify.png)

---

## 7. 系统配置分区（登录 / SQL审批通知）

菜单拆分为同级四项，避免「改一处清空另一处」：

| 菜单 | 路径 | 内容 |
|------|------|------|
| 系统配置 | `/config/?value=0` | goInception、JSON 预检、功能开关等 |
| 工单审核流配置 | `/config/?value=1` | 审批流相关 |
| **登录配置** | `/config/?value=2` | 本地 / Lark / 飞书多选、凭证测试、人员日同步 |
| **SQL审批通知** | `/config/?value=3` | 各通道通知 + Lark 催办 |

保存策略：**增量更新**（保存系统配置不会清空登录配置）。

![侧栏独立菜单](images/35-sysmenu-login-notify.png)

![登录配置](images/25-config-login.png)

![登录配置标注](images/26-config-login-annotated.png)

![SQL审批通知顶部](images/29-config-sql-notify-top.png)

![系统配置总览](images/08-system-config.png)

**相对 Archery**：上游通知与登录项多挤在同一大页；SDBM 分区后职责清楚，运维改错率更低。

---

## 8. 用户类型与 Submitter 权限组

### 8.1 用户管理：Lark 用户与用户类型

路径：`/admin/sql/users/`

- 列表增加 **用户类型**列：本地用户 / Lark / Lark（飞书）徽章。
- 右侧过滤器可按类型筛选。
- 与个人通知通道（`lark_open_id` 等）一致，便于核对「谁能收到 Lark 私信」。

**相对 Archery**：上游用户表无 SSO 身份列；混用本地与 IM 账号时难排查。

![用户类型列与筛选](images/30-admin-user-type.png)

### 8.2 权限组：Submitter

种子组：`Default` / `RD` / `DBA` / `PM` / `QA` / **`Submitter`**。

- **Submitter**：可提交并查看**组内**上线工单；**不可审批、不可执行**。
- 另支持权限组**描述必填**与**一键复制**（不含成员）。

**相对 Archery**：无开箱 Submitter 包；「只许提单不许批」需手工抠权限。

![Submitter 权限组](images/31-auth-submitter.png)

![Submitter 编辑页](images/32-auth-submitter-detail.png)

![权限组复制](images/14-auth-group-copy.png)

![权限组编辑复制](images/15-auth-group-edit-copy.png)

---

## 9. 数据源引擎裁剪

`ENABLED_ENGINES` / `DB_TYPE_CHOICES` 仅保留：

`mysql` · `pgsql` · `doris` · `mongo` · `clickhouse` · `goinception`

已移除上游常见的 Oracle、Redis、MsSQL、ES、Cassandra、TDengine 等入口，实例新增下拉里不再出现。

**相对 Archery**（上游 README 支持十余种库）：SDBM 面向当前业务栈瘦身，减少误选与镜像体积。

![实例数据库类型下拉](images/33-instance-engines-trimmed.png)

---

## 10. 前端 Vue 化与其它运维体验

### 10.1 Vue 页面

| 页面 | 路径 |
|------|------|
| SQL 上线列表 | `/sqlworkflow/` |
| SQL 提交 | `/submitsql/` |
| SQL 分析 | `/sqlanalyze/` |
| 在线查询 | `/sqlquery/` |
| 实例列表 | `/instance/` |
| 资源组 | `/group/` |

![Vue 工单列表](images/03-sqlworkflow-list.png)

![SQL 分析](images/05-sql-analyze.png)

![在线查询](images/06-sql-query.png)

![实例列表](images/10-instance-list.png)

![资源组](images/09-resource-group.png)

![Dashboard](images/07-dashboard.png)

### 10.2 工具与运维

- My2SQL / SOAR 双架构镜像；SchemaSync 修复等。
- goInception `config.toml` 中文注释；备份保留天数清理。
- Admin 深色主题；去掉短信 2FA 与 OpenAI 生成 SQL。

![My2SQL](images/11-my2sql.png)

![SQL 优化](images/17-sql-optimize.png)

![Admin 深色](images/20-admin-dark.png)

![相关文档](images/16-docs.png)

---

## 11. 功能一览表

| # | 二开点 | 相对 Archery 的优点 | 截图 |
|---|--------|---------------------|------|
| 1 | 执行窗口「1 天」 | 一键填 24h 北京时间窗 | 21 / 22 |
| 2 | 上传 × 清空编辑器 | 换文件不残留旧 SQL | 21 / 23 |
| 3 | 工单状态实时刷新 | 活跃态自动轮询+变更刷新 | 34 |
| 4 | Lark 催办 | 周期引用催审，可配间隔 | 27 / 28 |
| 5 | SQL审批通知独立页 | 通知与系统项解耦 | 27 / 29 / 35 |
| 6 | 用户类型（含 Lark） | 列表徽章 + 筛选 | 30 |
| 7 | Submitter 权限组 | 只提交/查看的种子角色 | 31 / 32 |
| 8 | 登录配置独立页 | SSO 凭证与人员同步集中 | 25 / 26 / 35 |
| 9 | 引擎裁剪 | 仅常用 5+1 种，界面更简 | 33 |
| 10 | JSON_PRECHECK | 默认开启字面量预检 | 24 |
| — | 权限组复制 / 描述 | 多环境克隆角色 | 14 / 15 |
| — | Vue 主路径 / 暗色 | 交互统一 | 04 / 07 / 19 |

---

## 截图文件清单

路径：`md/images/`

```
01-login-lark-default.png … 20-admin-dark.png   （既有总览图）
21-submit-1day-and-fileclear.png
22-submit-1day-only.png
23-submit-fileclear-x.png
24-config-json-precheck.png
25-config-login.png
26-config-login-annotated.png
27-config-sql-notify.png
28-config-lark-remind.png
29-config-sql-notify-top.png
30-admin-user-type.png
31-auth-submitter.png
32-auth-submitter-detail.png
33-instance-engines-trimmed.png
34-workflow-status-poll.png
35-sysmenu-login-notify.png
```

---

