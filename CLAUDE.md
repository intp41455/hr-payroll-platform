# hr-payroll-platform · 一体化薪酬与人力资源管理平台

> 企业级 HR 平台：**数据字典驱动的规则即数据架构** + 前后端分离 + 5 分钟可跑的本地部署。

---

## 1. 项目目的

这是一个**把企业薪酬规则从代码里抽出来、变成可配置数据**的 HR 平台架构。

**核心价值**：
- 传统做法：调整薪级/津贴/社保系数需要改代码、测试、发版
- 本项目做法：薪级、津贴、社保系数、豁免策略都是**数据字典里的行**，改配置即时生效，历史版本可追溯
- 适用：中小型企业自建 HR 系统、外包实施项目交付、以及想理解 HR 系统应该如何正确建模的开发者

**本仓库已做合规清理**：代码中的真实姓名、公司名、品牌标识均已替换为示例占位，可直接用于学习或二次开发。

---

## 2. 技术栈

| 层级 | 技术 | 理由 |
|------|------|------|
| **前端** | Vue 3 (browser-only，无构建) | `file://` 直接可跑，企业 IT 部署门槛低 |
| **后端** | Node.js + Express (单文件 `backend_server.js`) | 零框架依赖，客户运维不需要装 npm 依赖 |
| **部署** | Netlify Functions (serverless) | 前端静态托管 + 后端 Serverless，免运维 |
| **数据字典** | 共享 JS 模块 (`src/common/data_dictionary.js`) | 前后端**同时 import**，一处定义处处生效 |
| **规则引擎** | 三层架构 (基础/覆盖/例外) | 规则即数据，版本化可追溯 |
| **测试** | 零依赖自制测试运行器 | 50+ 测试文件，覆盖核心逻辑 + 每个需求点回归 |

**关键依赖**：`express`、`dayjs`、`@vercel/node` (仅 Netlify 部署用)

---

## 3. 目录结构

```
hr-payroll-platform/
├── backend_server.js          # 单文件 HTTP 后端（Express，Vercel/Netlify 兼容）
├── src/
│   ├── api/                   # 前后端共享的 API 接口定义
│   │   ├── attendance_admin_api.js
│   │   └── master_data_api.js
│   ├── common/                # 共享的字典与配置（前后端复用，关键！）
│   │   ├── data_dictionary.js      # 核心枚举、编码生成、字典导出
│   │   └── kangyuan_brand_config.js # 品牌/板块/法人主体配置
│   ├── integrations/          # 外部集成（钉钉等）
│   ├── launch/                # 培训操作手册、培训数据
│   ├── models/                # 领域模型
│   ├── modules/               # 业务模块（按功能拆分）
│   │   ├── ai/                # AI 人资 Agent
│   │   ├── attendance/        # 考勤处理（月度汇总、快照校验）
│   │   ├── audit/             # 审计日志
│   │   ├── ci/                # CI 相关
│   │   ├── dashboard/         # 管理驾驶舱（高管视图）
│   │   ├── finance/           # 银行代发网关
│   │   ├── leave/             # 假期引擎
│   │   ├── master_data/       # 主数据（员工、社保、组织架构）
│   │   ├── payroll/           # 薪酬计算引擎 + 个税
│   │   ├── rag/               # AI 知识接入
│   │   ├── rules/             # 三层规则引擎（核心！）
│   │   ├── selfservice/       # 员工自助
│   │   └── workflow/          # 工作流/审批
│   ├── services/              # 通用服务层
│   └── tests/                 # 单元测试 + E2E（50+ 文件）
├── test_*.js                  # 专项回归测试（13 个，根目录）
├── docs/                      # 架构文档
├── exports/                   # 脱敏示例数据（无真实姓名）
├── netlify/                   # Netlify 部署配置
│   ├── functions/api.js       # Netlify Functions 适配层
│   └── netlify.toml
├── package.json
└── package-lock.json
```

---

## 4. 安装/构建/运行/测试

### 本地开发（5 分钟启动）

```bash
# 1. 安装依赖
npm install

# 2. 启动后端（默认 3000 端口）
node backend_server.js

# 3. 打开前端（任意静态服务器或直接双击）
npx serve src
# 或者直接双击 src/index.html（前端不依赖构建）
```

**无需配置**：没有密钥、数据库、环境变量。所有数据在内存中，刷新即重置，便于演示与测试。

### 运行测试

```bash
# 后端单元测试（零依赖测试运行器）
node src/tests/run-all-tests.js

# 专项回归测试（根目录）
node test_TR_6_3_and_6_4.js
node test_task21.js  # ... 到 test_task33.js
```

### 部署到 Netlify

```bash
# 推送到 GitHub，连接 Netlify 自动部署
# netlify.toml 已配置：
# - publish = "public" (前端静态文件目录)
# - functions = "netlify/functions" (后端 Serverless 函数)
# - API 路由重写 /api/* -> /.netlify/functions/api/*
```

---

## 5. 关键约定与坑点

### 5.1 前后端共享数据字典（核心约定）

**`src/common/data_dictionary.js` 被前后端同时 import**——这是消除前后端不一致 bug 的架构决策。

- 前端做 UI 校验（下拉框选项、字段提示）
- 后端做业务计算
- **改一处，两处生效**，绝不允许前后端各写一份枚举

```javascript
// 后端 backend_server.js
const dict = require('./src/common/data_dictionary.js');

// 前端（通过模块系统或全局变量引用同一文件）
import { BUSINESS_UNIT_CODE, validateEnum } from './common/data_dictionary.js';
```

### 5.2 规则即数据，三层引擎

| 层 | 模块 | 用途 |
|---|---|---|
| **基础规则** | `src/modules/rules/rule_engine.js` | 薪酬项、科目、基础计算规则 |
| **特殊覆盖** | `src/modules/rules/special_rule_override_engine.js` | 针对特定人员/岗位的规则覆盖 |
| **例外豁免** | `src/modules/rules/special_exception_rule.js` | 单人级别的豁免（如试用期不发年终奖） |

**版本管理**：规则配置有版本号（`tr-1.7.2-rule-versioning.test.js` 验证），修改不覆盖历史，可追溯审计。

### 5.3 零构建前端

- 前端是 **browser-only** 的——**不经过 npm build**
- 直接用 `file://` 或任何静态托管就能跑
- Vue 3 通过 ES 模块直接在浏览器运行
- 这是刻意选择的约束，让企业客户技术同事能 5 分钟内部署

### 5.4 后端单文件 + 懒加载

- `backend_server.js` 单文件包含所有路由
- 核心引擎模块**懒加载**（Vercel/Netlify 冷启动优化）：
  ```javascript
  const loaders = {
    payroll: () => require('./src/modules/payroll/payroll_engine.js'),
    // ...
  };
  ```

### 5.5 薪酬计算引擎 7 大模块

`src/modules/payroll/payroll_engine.js` 实现完整计算流程：
1. 数据加载（人员、薪级、岗位、绩效）
2. 基础工资计算（基本工资 + 岗位工资 + 绩效系数）
3. 各类津贴叠加（交通、餐补、住房、通讯）
4. 社保公积金扣除（基数、比例、上下限）
5. 个人所得税（累计预扣法）
6. 特殊规则应用（override + exception 层）
7. 结果输出与校验（应发、应扣、实发、明细）

### 5.6 银行代发网关边界

`src/modules/finance/bank_payment_gateway.js` 对接银行代发的**文件生成与校验**——**不是真的走银行接口**（那需要企业 CA 与专线）。这是刻意的设计边界：接入真实银行接口属于合规敏感操作，本项目不触碰。

### 5.7 测试文件命名规范

- `src/tests/tr-*.test.js` — 需求追踪测试（按 TR 编号）
- `test_*.js` (根目录) — 专项回归测试，对应客户具体需求变更
- **这份测试清单本身就是交付质量的证据**：每个功能点都有对应测试，改一处不担心影响别处

### 5.8 已知边界（诚实说明，非缺陷）

| 边界 | 说明 |
|------|------|
| **无持久化** | 内存存储，重启即丢。生产环境需接数据库 |
| **无鉴权** | 本仓库面向演示与学习，不含用户认证。生产需自行加 |
| **不接真实外部系统** | 银行、税务、社保都是模拟。真实接入涉及合规与专线 |
| **多租户未实现** | 单公司单租户设计 |
| **UI 实用风格** | 走"能看清、能操作"，非设计竞赛级别 |

**这些边界不是缺陷，是刻意的取舍**——本项目目标是让客户能在 5 分钟内跑起来并理解系统，而不是提供一个生产就绪的黑盒。

### 5.9 合规声明

已清理项（可直接用于学习/二开）：
- ✅ 真实姓名替换为通用占位（"审批人"、"HR管理员"）
- ✅ 公司名/品牌名泛化为"示例集团"
- ✅ 敏感文件排除（微信会话分析、`.vercel/`、`.trae/`）
- ✅ `exports/` 仅含脱敏 ID（`EMP-EXE-008`），无真实姓名金额

**基于本仓库开发真实业务系统时**，请自行实现：鉴权、数据加密、审计日志，并遵守所在行业的数据合规要求。

---

## 6. 常用命令速查

```bash
# 启动后端
node backend_server.js

# 启动前端静态服务
npx serve src

# 运行所有单元测试
node src/tests/run-all-tests.js

# 生成数据字典 Markdown
node -e "console.log(require('./src/common/data_dictionary.js').generateDictionaryMarkdown())"

# 验证枚举值
node -e "const d=require('./src/common/data_dictionary.js'); console.log(d.validateEnum('EMPLOYEE_STATUS', 'REGULAR'))"
```

---

## 7. 许可证

MIT License — 见 `LICENSE`