# Kochi 管理系统 · 系统说明（SYSTEM.md）

> 本文档是这个系统的「总说明书」。每开一个新对话做新功能，先把本文件交给 Claude。
> 每次上线改动后，同步更新本文件（版本记录、数据库、规则）。
> 最后更新：2026-09-27（数据库 v12 · 识别服务 bon v2 · 网页 v12）

---

## 1. 系统概况

两家餐厅（Kochi Flingern、Kochi Bismarck，GLL International GmbH）的内部管理系统。
员工每天用它交 Bon、看小费、看排班；老板和小费管理员用它点收信封、周结、审批。

目前的模块：

| 模块 | 状态 | 说明 |
|---|---|---|
| 小费分配（Trinkgeld） | 已上线 | 拍照识别 Lightspeed Nutzerbericht、信封、点收、周结 |
| 排班（Dienstplan） | 第一步已上线 | 员工标不能上的班、排班员排班发布 |

规划中的模块（按顺序）：排班第二步（和小费打通）→ 班次任务清单 + 值班负责人 → HACCP 记录 → 物品位置和最低库存 → 工时和合规 → 新人培训。

## 2. 技术结构

| 部分 | 位置 | 说明 |
|---|---|---|
| 网页（前端） | GitHub 仓库 `kochi-trinkgeld`，GitHub Pages | 目前是一个 `index.html`（CSS + JS 都在里面），中德双语 |
| 数据库 + 接口 | Supabase 项目 `hxegfkyviumkdbxcuokw`（单独项目，不和电费 App 共用） | PostgreSQL，所有读写都通过 RPC 函数 |
| 识别服务 | Supabase Edge Function `bon` | 调 Claude 读 Bon 照片（先 Haiku，检查不通过再用 Sonnet）；照片存在私有存储桶 `bons` |
| 密钥 | Supabase → Edge Functions → Secrets | `ANTHROPIC_API_KEY` |

重要设置：Edge Function `bon` 的 **Verify JWT 必须关闭**（系统用自己的 PIN 登录，不用 Supabase 登录）。

### 2.1 安全模型

- 所有表开启 RLS 且没有任何策略：网页**不能直接读写表**，只能调用 `security definer` 的 RPC 函数。
- 每个 RPC 第一步用 session token 校验身份（`_auth` / `_auth_any`），再检查权限（`_require_admin`、`_require_manager`、`_require_scheduler`）。
- 金额一律以「分」（整数）存储。
- 所有重要操作写入 `audit_log`（后台「日志」可查）。

### 2.2 登录

- 员工选名字 + 6 位 PIN。新员工初始 PIN `888888`，第一次登录必须改成自己的（输入两次）。
- PIN 连错 5 次锁定 15 分钟。
- 登录页按主岗位分组：服务 / Runner / 吧台；管理员和厨房账号在「管理员登录」页。
- **店里共用设备（默认）**：员工 90 秒无操作自动退出（管理员、小费管理员、排班员 5 分钟），关页面即失效，服务器端登录最长 12 小时。
- **个人设备**：登录时勾选「这是我自己的手机，保持登录」，一直保持登录，最长 30 天。每次登录默认不勾选；勾选会记日志。

### 2.3 角色

| 角色 | 字段 | 能做什么 |
|---|---|---|
| 员工 | — | 交 Bon、看自己的账本、看排班、标不能上的班 |
| 主岗位 | `staff.main_role` = service / runner / bar | 登录页分组、选人排序；**Runner 和吧台（非管理员）登录后只看账本，不能录入** |
| 厨房账号 | `is_kitchen` | 只有一个在用；领取厨房份额（交给厨师长内部分）；只看账本 |
| 小费管理员 | `is_tip_manager` | 点收信封、周结、看周总览 |
| 排班员 | `is_scheduler` | 排班、发布 |
| 超级管理员（老板） | `is_admin` | 全部功能，包括审批、核对 Bon、员工管理、以员工视角查看 |

## 3. 小费模块规则

### 3.1 员工交 Bon（拍照录入）

1. 下班先在收银机 **Entstempeln**，打印 **Abschlussbericht**（Nutzerbericht，带「Identifikationsnummer für Bericht」）。
2. 拍顶部（名字、工号、日期、报告编号）和底部（Einnahmen）两张照片，系统自动开始识别。
3. 员工点钱，输入**钱包现金合计**（不扣零钱底）。输完之后才显示单据数字和分配明细（点钱时看不到应收数，避免受影响）。
4. 选当天的 Runner 和吧台（可多选；可选「无」）。第一次点「提交」只展示明细，再点一次才提交。
5. 确认页显示放入信封的金额：**钱包里全部现金**放进信封，写名字、日期、金额，放进保险柜。

计算：
- 交店里 = Schlussbestand (Bar)（可为负：纯刷卡日店里要从 Kasse 补钱进小费池）
- 刷卡小费 = Bar 区块「Trinkgeld in Bargeld」= 除 Bar 以外所有付款区块（Lightspeed Payments、Kartezahlung Backup 等）的「Summe Trinkgeld」之和
- 小费合计 = 钱包 − Schlussbestand；现金小费 = 小费合计 − 刷卡小费
- 现金小费 < 0 = 差钱：先提醒重新清点并阻止提交；员工确认后可提交，差额默认从该员工小费扣，老板可一键改成店里承担

### 3.2 识别检查（Edge Function `bon`）

- 报告编号、工号、日期、门店必须读到。门店按邮编判断（40233 = Flingern，40210 = Bismarck）。
- 刷卡小费：每个区块自身一致；各区块之和 = Trinkgeld in Bargeld = 汇总里「Vom Bar-Umsatz abzuziehen」。
- Schlussbestand (Bar) = 现金营业额 + 现金小费 − Trinkgeld in Bargeld。
- 没有报告编号 → 报告未关闭：提示先 Entstempeln，**不能提交**。
- 检查不通过 → 前两次要求重拍；重拍 2 次仍不通过可以提交，服务端自动标记「待核对」，老板在「审批」看照片更正。
- 员工**不能修改**识别出的数字。
- 同一张 Bon（报告编号）只能上传一次；同一天可以交多张（午班、晚班）。

### 3.3 Lightspeed 工号绑定

- 每个员工在两家店各有一个工号（`staff_ls_ids`，按「门店 + 工号」）。
- 在某家店第一次上传时自动绑定，老板在「审批」确认。员工编辑页可分店解除绑定。
- 同一家店的一个工号只属于一个人；拿别人的 Bon 上传会被拒绝。

### 3.4 分配

- 比例（`rates`，按生效日期，不追溯）：厨房 15%、Runner 10%、吧台 5%，其余归服务员。
- 基数是小费合计（刷卡 + 现金）。
- 同岗位多人平分；除不尽的分归服务员；没有 Runner / 吧台时该份额归服务员。
- 两家店同一套比例。

### 3.5 点收、周结

- 老板和小费管理员登录后直接进入「周结」页：顶部是「待点收的信封」（上周还有没点完的先显示上周）。
- 输入实际点出的金额；不一致标红，可重点。「每日交接」表按天按店汇总：信封合计、放入 / 取出 Kasse、小费池。
- 锁定周结的条件：这周已结束、没有未点收的信封、没有待审批的修改、没有未处理的异议、（网页检查）没有待核对的 Bon。
- 周结后每人从小费池领取自己的份额；按整欧元发放，零头结转到下周。

### 3.6 修改

- 录入后 10 分钟内本人可直接改（只能改钱包金额、Runner、吧台）。
- 之后修改需要写原因，由老板审批后生效。
- 信封点收后只有老板能改；周结锁定后谁都不能改（只能记调整）。

## 4. 排班模块规则

- 班次模板（`shift_templates`）：Flingern 每天晚班 17–23，周六周日另有午班 11–15；Bismarck 每天午班 11–15、晚班 17–23。模板时间只是默认值，具体上班时间由排班员按人安排。
- **默认都能上**：员工只标下周哪些午班 / 晚班**不能上**（红色），没标的都算能上（绿色）。不需要每周确认。
- 截止：每周四 20:00（柏林时间）填下一周；可提前往后翻周标假期、考试周。周一到周四登录时顶部提醒。
- 规则（写在页面上）：排了你的班就要来；来不了自己找人换，并提前告诉排班员。
- 排班员按「日期 × 门店 × 班次」排人、指定岗位和具体时间；同一时段一个人只能排在一家店。选人弹窗分「能上 / 不能上」两组。
- 发布后员工能看到；发布后再改会标「有变更」，按钮变成「重新发布」，改动记日志。
- 底部显示每人本周计划工时。

## 5. 数据库版本记录

执行顺序 v1 → v12，每个脚本都可重复执行（schema v1 除外，它只在新项目执行一次）。

| 版本 | 内容 |
|---|---|
| v1 schema | 基础表：staff、sessions、rates、weeks、tip_entries、entry_helpers、allocations、adjustments、disputes、settlement_lines、audit_log；登录、录入、账本、周结、管理 RPC |
| v2 | 初始 PIN 888888 + 首次强制改 PIN；员工排序；修改审批（change_requests）；10 分钟直接修改；管理员登录分开 |
| v3 | 拍照录入：bon_reads、照片存储、钱包 / Schlussbestand / 差额、信封点收、小费池周结、Lightspeed 绑定 |
| v4 | 老板以员工视角查看账本（只读，记日志） |
| v5 | 主岗位（main_role）；店里设备登录最长 12 小时 |
| v6 | 个人设备登录，最长 30 天（sessions.personal） |
| v7 | Runner / 吧台智能排序数据（helper_stats、day_helpers） |
| v8 | 识别未通过的 Bon 标记待核对（needs_review），老板核对更正 |
| v9 | 排班：shift_templates、availability、shifts、schedule_weeks；排班员权限 |
| v10 | 可上班时间「提交」记录（avail_submits；v11.1 起网页不再使用，表保留） |
| v11 | Lightspeed 工号按门店绑定（staff_ls_ids）；submit_bon_entry 按门店检查 |
| v12 | 去掉「同一员工同一天同一家店只能一条录入」，支持午班 + 晚班两张 Bon |

识别服务版本：bon v1（v3 时上线）→ bon v2（支持多种刷卡区块，如 Kartezahlung Backup）。

### 5.1 历史遗留（不影响使用，以后整理时处理）

- `staff.lightspeed_id / lightspeed_name / lightspeed_confirmed`：v11 起不再使用，数据保留。
- `admin_confirm_lightspeed()`：v11 起网页改用 `admin_ls_set()`。
- `avail_submits`、`submit_availability()`：v11.1 起网页不再使用。
- 「有待核对的 Bon 不能锁周结」目前只在网页检查，数据库的 `lock_week()` 还没加这条。
- 门店名称 `'Flingern'`、`'Bismarck'` 写死在表的检查约束里。

## 6. 上线流程

1. **数据库**：Supabase → SQL Editor → New query → 粘贴新版本 SQL → Run，看到 Success。
2. **识别服务**（有改动时）：Edge Functions → `bon` → 粘贴新代码 → Deploy；确认 Verify JWT 仍关闭。
3. **网页**：GitHub 仓库上传新的 `index.html`，提交时粘贴更新日志。
4. 顺序：**先数据库，再网页**。
5. 上线后用真实数据走一遍对应功能。

## 7. 合作约定

- 每次更新都附 **GitHub 提交日志**（纯文本块，可直接复制）。
- 每次更新都写明**对现有数据的影响**。加功能不能导致已有数据丢失；涉及修改或删除数据的改动，先问老板。
- 系统拒绝任何操作时，**必须说清楚原因和下一步怎么做**，不能用笼统的报错。
- 新功能先在本地测试数据库和浏览器里测过再交付；测不了的部分（真实 Claude 调用、真实 Supabase）明确说出来，让老板上线后验证。
- 沟通用中文；员工看到的界面中德双语；对外德语信件用正式德语。
- 每个新模块开一个新对话，先读本文件。

## 8. 接下来

地基整理（不加新功能、不改数据库、不动数据）：
1. 自己的域名：`app.kochide.com`（域名在 GoDaddy）
2. 仓库建 `migrations/` 文件夹，放入 v1–v12 SQL 和 `edge_function_bon.ts`
3. 数据库自动备份
4. 测试环境（第二个 Supabase 项目 + 测试网页）
5. 拆分网页文件：公共部分 + 各模块
6. 检查所有报错提示，每条写清原因和解决办法

然后：排班第二步（自动选当天排班的 Runner / 吧台；排了班没交 Bon 的提醒）→ 班次任务清单 + 值班负责人。
