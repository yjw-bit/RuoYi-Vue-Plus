# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**RuoYi-Vue-Plus** (`org.dromara:ruoyi-vue-plus:5.6.2`) — Dromara's rewrite of RuoYi-Vue targeting **distributed clusters + multi-tenant** scenarios. It is a Spring Boot 3.5.15 backend (JDK 17/21) with a separately-hosted Vue3 + TS + ElementPlus frontend (not in this repo — see [plus-ui](https://gitee.com/JavaLionLi/plus-ui)).

The codebase is **intentionally modular**: ~25 starter-style `ruoyi-common-*` jars are auto-loaded via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Business features live in `ruoyi-modules/*` (system, workflow, generator, job, demo). Two extra standalone services live in `ruoyi-extend/*` (Spring-Boot-Admin monitor, SnailJob server). The only runnable web app is `ruoyi-admin` (entry: `org.dromara.DromaraApplication`).

Docs: https://plus-doc.dromara.org · Demo: https://plus-doc.dromara.org/#/common/demo_system

## Build & Run Commands

Maven profiles map directly to Spring `spring.profiles.active` and are declared in the parent `pom.xml`:

| Profile | Property `profiles.active` | Use |
|---|---|---|
| `local` | `local` | Local dev — `@logging.level@=info` |
| `dev` *(default)* | `dev` | Default (`<activeByDefault>true</activeByDefault>`) |
| `prod` | `prod` | Production — `@logging.level@=warn` |

```bash
# Compile everything (tests skipped by default — see <skipTests>true</skipTests> in parent pom)
mvn clean install

# Run a single module with hot-reload (use the profile you need)
mvn -pl ruoyi-admin -am spring-boot:run -P dev

# Run a single test class
mvn -pl ruoyi-admin test -Dtest=org.dromara.test.DemoUnitTest

# Run a single test method
mvn -pl ruoyi-admin test -Dtest=org.dromara.test.DemoUnitTest#testTest

# Run all tests across the reactor (overrides skipTests)
mvn test -DskipTests=false

# Build the runnable jar (output: ruoyi-admin/target/ruoyi-admin.jar)
mvn -pl ruoyi-admin package -DskipTests
```

`.run/ruoyi-*.run.xml` are IntelliJ Docker-deploy run configs that build images via the per-module `Dockerfile` (`ruoyi/ruoyi-server:5.6.2`, etc.). Use `script/bin/ry.sh start|stop|restart|status` (or `ry.bat` on Windows) to manage a deployed jar. `script/docker/docker-compose.yml` brings up the full middleware stack (mysql, redis, nginx, etc.).

**Database init scripts** live in `script/sql/`:
- `ry_vue_5.X.sql` — main business schema (MySQL)
- `ry_job.sql` — SnailJob tables
- `ry_workflow.sql` — warm-flow tables
- `oracle/`, `postgres/`, `sqlserver/` — vendor ports; `update/` for incremental migrations

## Module Map

```
ruoyi-admin          Spring-Boot entry. Pulls in common-* + all modules. application.yml/@profiles.active@
ruoyi-common/
  ruoyi-common-bom   Centralised version BOM imported by parent <dependencyManagement>
  ruoyi-common-core        R, BaseEntity, exceptions, utils (the "kernel")
  ruoyi-common-web         BaseController, Captcha, I18n, Undertow, Filter, Resources config
  ruoyi-common-mybatis     MyBatis-Plus config, DataPermission (annotation-driven SQL rewriting), PageQuery/TableDataInfo
  ruoyi-common-redis       Redisson (Lock4j, distributed lock, rate limiter)
  ruoyi-common-satoken     Sa-Token auth, JWT, multi-device login, StpInterface
  ruoyi-common-security    Spring Security glue (kept thin — Sa-Token is the real auth)
  ruoyi-common-tenant      Multi-tenant interceptor + exclude-table list
  ruoyi-common-log         @Log + async operlog (operate/visit/login logging)
  ruoyi-common-idempotent  @RepeatSubmit
  ruoyi-common-ratelimiter @RateLimiter
  ruoyi-common-encrypt     API request/response body AES+RSA, plus mybatis field encryptor
  ruoyi-common-sensitive   Jackson-based desensitisation (idCard, phone, bankCard, …)
  ruoyi-common-translation @Translation + DictService for runtime dictionary lookups
  ruoyi-common-excel       FastExcel (EasyExcel fork) with auto-merge, dict translation
  ruoyi-common-oss         S3-protocol client (MinIO default; 七牛/阿里/腾讯 via config)
  ruoyi-common-sms         sms4j multi-vendor gateway
  ruoyi-common-mail        mail-api unified mail client
  ruoyi-common-social      JustAuth third-party login
  ruoyi-common-sse         Spring SSE with token auth + cluster sync
  ruoyi-common-websocket   Spring WebSocket with token auth + cluster sync
  ruoyi-common-json        Jackson customised for the project
  ruoyi-common-doc         SpringDoc + therapi-runtime-javadoc (javadoc → OpenAPI)
  ruoyi-common-job         SnailJob client starter
  ruoyi-common-validate    Validation groups + Chinese i18n messages
  ruoyi-common-xss         XSS request-body filter (configurable exclude URLs)
ruoyi-modules/
  ruoyi-system        Core admin: user/role/menu/dept/post/dict/notice/oss/tenant/client/social/config
  ruoyi-generator     Velocity-based CRUD scaffolding — reads `gen_table` + `gen_table_column`, writes Java/SQL/HTML/TS. Triggers at `/tool/gen`
  ruoyi-workflow      warm-flow integration: leave demo + category/instance/task/spel APIs
  ruoyi-job           @SnailJob annotated job handlers consumed by the SnailJob server
  ruoyi-demo          Live feature samples: Excel import/export, Redis lock, websocket, ratelimit, sse, sensitive, translation, mail, sms
ruoyi-extend/
  ruoyi-monitor-admin  Spring-Boot-Admin server (UI at port 9090)
  ruoyi-snailjob-server  Embedded SnailJob server (port 17888)
```

## Architecture & Conventions

### How a request flows
1. UI hits a controller in `org.dromara.{module}.controller` (all extend `BaseController` from `ruoyi-common-web`).
2. Controller uses `@SaCheckPermission("module:entity:action")` — Sa-Token validates the token + permission string against the user's roles/permissions (loaded by `StpInterface`).
3. Tenant filter (`ruoyi-common-tenant`) injects `tenantId` into MyBatis-Plus context; data-permission handler rewrites SQL based on `@DataPermission`/`@DataColumn` annotations from `ruoyi-common-mybatis`.
4. `IService` interface + `impl` pattern (MyBatis-Plus `IService`). Mappers are `BaseMapper<X>` subclasses — almost zero hand-written SQL.
5. Return: `R.ok(data)` / `R.fail()` for single objects, `TableDataInfo<T>` for paginated lists, plain `void` for export/download responses.
6. `@Log(title, businessType=...)` writes to `sys_oper_log` async; `@RepeatSubmit` short-circuits duplicates.

### Package layout inside every business module
```
org.dromara.{module}
├── controller/         @RestController, @RequestMapping("/xxx"), extends BaseController
├── service/            IXxxService interface
│   └── impl/           XxxServiceImpl @Service @RequiredArgsConstructor
├── mapper/             XxxMapper extends BaseMapper<Xxx> (MyBatis-Plus)
├── domain/             @TableName entities
│   ├── bo/             Business objects — form/request DTOs (incoming)
│   └── vo/             View objects — response shapes (outgoing)
├── listener/           Spring event listeners (e.g. dict refresh)
└── runner/             ApplicationRunner / CommandLineRunner startup tasks
```
MapStruct-Plus (`mapstruct-plus`) auto-generates `bo↔entity` and `entity↔vo` mappers — no hand-written copy code.

### Key patterns to follow
- **Lombok everywhere**: `@RequiredArgsConstructor` for DI, `@Slf4j` for logging, `@Data`/`@EqualsAndHashCode(callSuper = true)` on entities. Never write getters/setters or constructors by hand.
- **Permission strings**: `module:entity:action` (e.g. `system:user:list`, `system:user:export`). Registered in `sys_menu` per role.
- **Pagination**: pass `PageQuery pageQuery` parameter + a query BO, return `TableDataInfo<T>`. Never use `PageHelper`.
- **Multi-tenant**: the tenant interceptor handles most tables. Add table names to `tenant.excludes` in `application.yml` for shared/system tables.
- **Response wrapper**: `R<T>` from `org.dromara.common.core.domain.R`. `R.ok()` / `R.ok(data)` / `R.fail()` / `R.fail(msg)`. Use `TableDataInfo<T>` for paged responses.
- **Validation**: `@Validated` on controller, `jakarta.validation.constraints.*` on BO fields. i18n messages via `ruoyi-common-validate`.
- **Code style**: 4-space indent (Java), 2-space (yml/json) per `.editorconfig`. Strict Alibaba style.
- **Author header**: `* @author Lion Li` on new public types is the convention.

### Adding a new CRUD table
1. Create the table + columns in `gen_table` / `gen_table_column` (or via the `/tool/gen` UI imported from the system menu).
2. Use the generator to scaffold Controller / Service / Mapper / Domain(BO+VO) / SQL / frontend page — it produces MyBatis-Plus + SpringDoc + multi-tenant-aware code.
3. Register a `sys_menu` row with the `xxx:entity:list/add/edit/remove/export` permission codes, attach to the appropriate role.

### Modifying a common module
- Every `ruoyi-common-*/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` already lists its `*Config` classes. Adding a new auto-config = add a `XxxConfig` + a line in that file.
- Bean configuration classes are `<feature>Config` (e.g. `CaptchaConfig`, `RedisConfig`, `MybatisPlusConfig`) and live alongside the implementation.
- Properties classes go in `org.dromara.common.<feature>.config.properties` and are bound via `@ConfigurationProperties(prefix = "…")`.

## Testing

Tests live in `ruoyi-admin/src/test/java/org/dromara/test/`. They are pure unit/integration tests using **JUnit 5** (`@DisplayName`, `@Test`, `@Nested`, `@Timeout`, `@RepeatedTest`, `@Disabled`). Because tests run inside `ruoyi-admin`, they have access to the full Spring context, including the MyBatis-Plus + Redisson + Sa-Token wiring. `DemoUnitTest` shows the canonical `@SpringBootTest` + autowire pattern.

There is no test in the common or feature modules — test through `ruoyi-admin`.

## Things NOT to do
- Do not write a `Controller` that returns `Map<String,Object>` or a raw entity — always use `R<T>` / `TableDataInfo<T>`.
- Do not introduce a new `ThreadPoolExecutor` — Spring Boot 3.5's built-in `TaskExecutionAutoConfiguration` is used everywhere (see `spring.task.execution` in `application.yml`).
- Do not write hand-written `LIMIT` SQL — extend `BaseMapper<T>` and use `PageQuery`/`Page<T>`.
- Do not use `fastjson` or `Gson` — Jackson only (already a global `ObjectMapper` bean in `ruoyi-common-json`).
- Do not commit the default `api-decrypt` RSA key pair in `application.yml` — replace it before production.
- Do not bypass `BaseController.toAjax(int rows)` for write ops; the framework relies on it for consistent error semantics.
- Do not add a dependency without checking `ruoyi-common-bom/pom.xml` first — versions are centrally managed.

---

## 本地项目进度（非上游内容 · 2026-08-31 起追加）

> ⚠️ **本节为本地项目记录**，与上方上游官方说明无关。请勿向上游提交此节。pull 上游时如遇冲突，本节内容以「本地为准」。

### 项目背景

本仓库用于承接「**材料导入进展跟踪表的流程审批**」独立服务，源自用户对一份 15 列 Excel 跟踪表（含导入状态、开发理由、材料名称、申请日期、SOP 编号、库存编码、规格型号、订单号、到货信息、测试计划、3 项测试结果）做多级审批 + 节点附件上传的需求。

### 选型结论

| 候选 | 结论 | 原因 |
|---|---|---|
| warm-flow 独立使用 | ❌ 放弃 | 自带请假 demo、流程设计器、RBAC、MinIO 都要自己造轮子，开发量翻倍 |
| RuoYi-Cloud-Plus（微服务） | ❌ 放弃 | Nacos/Dubbo/Seata/MQ 中间件对单一审批流严重过度 |
| **RuoYi-Vue-Plus 5.X（单体）** | ✅ **采用** | 自带工作流 + 请假 demo + Vue 设计器 + MinIO + RBAC，一人可维护 |

### 技术栈

- **后端**：本仓库（Spring Boot 3.5.15 + JDK 17/21 + MyBatis-Plus + warm-flow + Sa-Token + Redisson + MinIO + dynamic-datasource）
- **前端**：`../plus-ui` 5.X 分支（Vue 3 + TypeScript + Element Plus + Vite）
- **数据库**：MySQL 8.0+（docker-compose 已配，root/root，库名 `ry-vue`）

### 仓库状态（2026-08-31）

| 仓库 | 分支 | 最新提交 | 版本 |
|---|---|---|---|
| `RuoYi-Vue-Plus/` | `5.X` | `ee3518ede fix 修复 多节点提交任务 变量覆盖问题` | 5.6.2 |
| `../plus-ui` | `5.X` | `d0d4519 !273 发布 v5.6.2-v2.6.2 版本 依赖升级` | 5.6.2-2.6.2 |

### 路线图

- [ ] **阶段 1 · 环境起服**：docker compose 起 MySQL/Redis/MinIO/Nginx → 跑 `script/sql/ry_vue.sql` + `ry_workflow.sql` → `./mvnw spring-boot:run -pl ruoyi-admin -P dev` → 跑通 `admin / admin123` 登录
- [ ] **阶段 2 · 验证 demo**：跑通请假 demo（菜单「工作流 → 请假申请」+ 待办审批 + 流程设计器 `leave1`），确认环境就绪
- [ ] **阶段 3 · 业务建表**：建 `material_import`（仿 `test_leave`）+ `material_attachment`（业务附件表，关联 `businessId`）
- [ ] **阶段 4 · 后端代码**：仿 `TestLeave` 写 `MaterialImport` 的 Entity/Bo/Vo/Controller/Service（含 `submitAndFlowStart`）
- [ ] **阶段 5 · 流程设计**：在设计器里画「材料导入」流程（待定节点：申请人 → 主管 → 质量 → 仓库）
- [ ] **阶段 6 · 前端代码**：仿 `src/views/workflow/leave` 写材料导入的列表/表单/详情/待办页面
- [ ] **阶段 7 · 附件上传接入**：复用 `ruoyi-resource` 的 `SysOssController` + 自建 `material_attachment` 表的 CRUD

### 关键参考

- **请假 demo 后端位置**：`ruoyi-modules/ruoyi-workflow/src/main/java/org/dromara/workflow/{controller,domain,service}/TestLeave*`
- **请假 demo 前端位置**：`../plus-ui/src/views/workflow/leave` + `../plus-ui/src/api/workflow/leave`
- **业务与流程关联表**：`FlowInstanceBizExt`（表名 `flow_instance_biz_ext`，字段 `businessId`/`businessCode`/`businessTitle`）
- **WorkflowService API**：`org.dromara.workflow.service.WorkflowService`（`startWorkFlow` / `completeTask` / `deleteInstance` / `getBusinessStatus`）
- **warm-flow 设计器**：前端菜单「流程管理 → 流程定义」；`script/leave/leave1.json` ~ `leave6.json` 是 JSON 模板
- **官方文档**：https://plus-doc.dromara.org

### 备注

- 当前 shell 启动目录约定：先 `cd ~/projects/RuoYi-Vue-Plus`
- 中间件启动命令：`cd script/docker && docker compose up -d`
- 后端启动命令：`./mvnw spring-boot:run -pl ruoyi-admin -P dev`

