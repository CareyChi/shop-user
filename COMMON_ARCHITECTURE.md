# 多租户电商系统：三仓库架构与接口约定

版本：1.0.0；编写日期：2026-09-27。本文是三仓库共同的设计基线，三份内容必须保持一致。

## 交付范围与状态

本次仅交付开发文档及 frontend/、backend/ 目录占位，不包含应用代码、数据库脚本、构建配置或可运行服务。文中的接口、表结构、测试与部署均为开发目标，不代表已经实现。

功能状态统一：⚪ 待实现；✔ 已实现。只有代码合并、对应验收通过并记录提交号后才能把功能改为 ✔。编号在每份文档内从 A 开始，跨仓库引用使用“仓库/端/编号”，例如 shop-seller/后端/C02。

## 已作出的业务设计决定

1. 一个商家对应一个 tenant_id，首期一个商家主账号，不扩展商家员工与复杂组织。消费者属于平台，可以浏览多个店铺，订单严格归属一个商家。
2. 购物车允许跨店存放，但结算时按店铺分别提交；不提供跨店原子结算。页面必须让用户选择一个店铺结算，不能悄悄拆分后声称全部成功。
3. 订单提交后进入 PENDING_CONFIRM（待商家确认收款）。商家手动确认后进入 COMPLETED（已完成）。这是演示收款记录，无支付网关、支付链接、真实扣款、支付回调、退款或物流流程。
4. 待确认订单可由本人取消、所属商家拒绝，或默认 24 小时到期自动关闭。终态为 CANCELLED、REJECTED、EXPIRED、COMPLETED，不允许回退或硬删除；关闭原因留存。过期时长为部署配置，创建时固化 expires_at。
5. 商家注册后 PENDING_APPROVAL；审批通过 ACTIVE，驳回 REJECTED，可修改资料重新提交；ACTIVE 可停用为 SUSPENDED，管理员恢复后为 ACTIVE。待审批/驳回身份仅能查看和修改入驻申请，停用身份只显示停用提示，不进入业务后台。
6. 停用店铺后立即禁止新增订单、确认收款、商品及库存变更；用户仍能查询/取消已有待确认订单，超时任务仍可释放库存。恢复不自动解除平台商品封禁，也不自动上架草稿。
7. 商品商家状态 DRAFT/ON_SALE/OFF_SALE 与平台治理状态 NORMAL/BLOCKED 分开。购买条件：商家 ACTIVE、店铺 sales_blocked=false、商品 ON_SALE 且 NORMAL、SKU 可售。部分下架锁定所选商品；全部下架设置店铺级销售禁用开关，覆盖现有和未来商品，商家无法自行解除。
8. 下架只影响新成交；普通商品下架不取消历史订单，商家 ACTIVE 时仍可确认旧订单。停用商家则冻结确认。无商品物理删除，只允许归档，历史订单快照不变。
9. 币种固定 CNY；价格数据库 DECIMAL(12,2)，Java BigDecimal，API 使用两位小数字符串；前端按整数分展示合计，最终金额始终以后端为准。运费固定 0，不包含优惠券、税费、营销及售后。
10. 日期存储 UTC DATETIME(3)，API ISO-8601 带 Z，界面 Asia/Shanghai。报表按北京时间自然日，筛选转换成 UTC 的 [start,end) 区间。

## 服务与数据归属

| 仓库 | frontend/ | backend/ 规划模块 | 业务数据所有者 |
|---|---|---|---|
| shop-user | 消费者商城 SPA | user-api、gateway | shop_user：购物车、收货地址；gateway 无业务库 |
| shop-seller | 商家后台 SPA | seller-api、commerce-service | shop_commerce：商家入驻资料及状态、商品、SKU、库存、订单、销售、商家审计、导出任务 |
| shop-admin | 平台后台 SPA | admin-api、identity-service | shop_identity：三端身份、会话、权限；shop_admin：管理员本地操作日志 |

三个仓库是协作系统，不能各自复制一套订单和库存。seller-api、user-api、admin-api 是各端 BFF（面向前端的 API 层）；commerce-service 是唯一交易写入者，identity-service 是唯一身份与凭据写入者。每个服务只能持有自身数据库账号；BFF 不跨库 SELECT/UPDATE。消费者商品、订单查询及提交通过 user-api 调用 commerce-service；商家业务通过 seller-api 调用 commerce-service；管理员通过 admin-api 调用 commerce-service 的显式平台治理接口。身份服务向 commerce-service 查询入驻状态，不缓存 ACTIVE 授权结论；不可用时拒绝相关操作。

商家 tenant_profile 及审核/冻结状态在 commerce-service 内为唯一权威，审批、停用和交易在同一数据库围绕 tenant_profile 行锁串行化。身份服务只保存 SELLER 身份和 tenant_id 关联，不复制可交易状态。注册使用幂等 provisioning 流程：identity 生成 account_id 与不可复用的 tenant_id 并保存 PROVISIONING → 持久化任务幂等创建 PENDING_APPROVAL 商家资料 → identity 标为 READY。失败保持 PROVISIONING、重试或显示处理中，不返回已激活；tenant_id 唯一，重试不得生成第二家店。入驻审批也必须核实关联身份 READY。

## 技术选型与版本策略

| 层次 | 选型及版本基线 | 用途 |
|---|---|---|
| 前端 | Vue 3.5.x、Pinia 3.x、Vue Router 4.x、TypeScript 5.9.x | Composition API、状态、路由、类型 |
| 工程/UI | Vite 7.x、Node.js 22 LTS（至少 22.12）、pnpm 10.x、Element Plus 2.x | 三端统一组件和构建 |
| HTTP/验证 | 原生 fetch + AbortController；Vitest 3.x、Vue Test Utils 2.x、Playwright 1.x | 减少 HTTP 依赖；组件与浏览器验收 |
| Java | Java 21、Maven 3.9.x | 多模块 Maven 工程 |
| Spring 基线 | Spring Boot 3.5.0、Spring Cloud 2025.0.0、Spring Cloud Alibaba 2025.0.0.0 | 采用官方列明的兼容组合；不是宣称最新版 |
| 注册/治理 | Nacos 3.0.3、Sentinel 1.8.9 | 注册、配置、限流；客户端遵循 Alibaba BOM |
| Spring 配套 | Spring Security、Spring Data JPA、Hibernate、Validation、Actuator、Flyway、Connector/J 均由 Boot BOM 管理 | 不手写互相冲突版本；MySQL 的 Flyway 模块单独引入 |
| 服务调用 | Spring Cloud Gateway、OpenFeign、LoadBalancer，由 Cloud BOM 管理 | 网关路由及内部同步调用 |
| 存储 | MySQL 8.0.x（InnoDB、utf8mb4）、Redis 7.2.x | MySQL 为业务事实；Redis 限流/会话辅助，不做库存事实源 |
| 接口/导出 | springdoc-openapi 2.8.x、Apache POI 5.4.x | OpenAPI 3、SXSSF 流式 XLSX |
| 测试/运维 | JUnit 5、Spring Boot Test（BOM 管理）、Testcontainers 1.21.x；Docker Compose v2 | MySQL 8.0 集成测试与联合运行 |

版本段表示计划依赖线，编码首个提交必须解析并锁定精确补丁版本、提交 pnpm-lock.yaml，Maven 使用 dependencyManagement，容器锁定版本及 digest；不能使用 latest、SNAPSHOT 或动态范围。Spring 三件套以此兼容组合起步，编码前按漏洞扫描与兼容验证统一升级补丁，三个仓库同步改版，不能单独越级升 Boot 4。当前交付未运行依赖安装或兼容性实测，禁止将设计选型当作构建已通过。

核查依据（2026-09-27）：[Alibaba 版本矩阵](https://sca.aliyun.com/docs/2025.x/overview/version-explain/)、[Vite 7 Node 要求](https://vite.dev/blog/announcing-vite7)、[springdoc v2 文档](https://springdoc.org/v2/)。首期不引入 Elasticsearch、Seata、RocketMQ：商品/订单使用 MySQL 索引查询，交易原子性由 commerce 本地事务保证；可靠后台工作采用数据库任务表与重试。后续需求确有必要时另行设计。

## 安全、租户隔离与统一协议

- 三种 audience：USER、SELLER、ADMIN，权限分域；不允许用消费者会话访问商家或管理员接口。管理员没有公开注册入口，只允许部署时一次性初始化，首次登录强制改密，凭据来自环境/密钥管理。
- 登录采用 identity-service 维护的随机 opaque session，浏览器 HttpOnly、Secure、SameSite=Lax cookie，各端分别命名并限制 /api/v1/user、/seller、/admin 路径；Cookie 不进入 Pinia/localStorage。会话绝对期限 8 小时、闲置 30 分钟；改密/停用身份/退出使会话失效。登录前匿名 CSRF 会话，所有修改请求验证 CSRF token 与 Origin，token 从各端 /auth/csrf 获取并只放内存。密码 BCrypt，12～72 UTF-8 字节，禁止日志记录。
- BFF 向 identity 验证 cookie 后获取短期内部签名用户上下文（issuer、audience、subject、role、tenant_id、session_version、exp、request_id），服务间 mTLS 或等价 workload credential。commerce 同时验证调用服务身份、用户上下文和操作范围；不能信任浏览器 X-Tenant-Id/X-User-Id/X-Role。gateway 删除外部伪造的内部头，内部端点不注册公网路由。
- 商家 tenant_id 必须来自可信主体，SQL 查询、JOIN、更新、日志、报表、缓存键与文件下载均带 tenant_id。订单明细和 SKU 关系使用 (tenant_id,id) 复合唯一键/外键，避免异租户关联。
- USER 的订单数据以 user_id 限定；可以跨商家查自己的订单，不能查其他消费者。地址与购物车也仅本人。公开商品读模型只返回可见字段，无成本、采购、库存流水、联系方式。
- ADMIN 不通过普通请求“忽略租户拦截器”；使用专用平台查询方法及权限 SELLER_REVIEW、SELLER_MANAGE、PRODUCT_MODERATE、AUDIT_READ、SALES_READ。每次跨租户操作记录管理员、范围、理由、request_id。
- 统一前缀 /api/v1/{user|seller|admin}。成功 JSON：{"code":"OK","message":"成功","data":{},"requestId":"req-001"}；错误使用同一外壳，data=null 或结构化字段错误。201 创建，202 后台任务，200 查询/修改；400 参数、401 未登录、403 禁用/无权、404 资源不存在或非所属、409 库存/版本/状态冲突、422 业务规则、429 限流、503 下游不可用。
- ID 全部 JSON 字符串；分页 page 从 1 开始，pageSize 默认 20 最大 100，返回 items/page/pageSize/total；排序字段白名单并以 id 作稳定第二排序。商品/订单搜索 keyword 最长 100，使用参数化查询并转义 LIKE 特殊字符。
- 创建订单、确认收款、取消/拒绝、库存调整、审批及批量治理必须带 Idempotency-Key。唯一范围 (actor_id,operation,key)，保存规范化请求哈希和结果；相同键不同参数返回 IDEMPOTENCY_CONFLICT。核心订单创建关联键永久保留，其他操作结果至少 7 天。
- 写操作带 version；乐观版本冲突返回 VERSION_CONFLICT，不允许前端覆盖别人的更新。内部写请求超时不能盲目换键重试，使用原键查结果或重试；查询可以有限重试，所有下游有连接/读取超时。
- 头像/商品图仅允许经过校验的上传资源 ID；最多 5MB/张、最多 8 张，验证文件魔数和 MIME（JPEG/PNG/WebP），重编码去除元数据，不接收 SVG、任意远程 URL 或富文本脚本。上传文件存受控持久卷，公开只读资源路由；导出文件走独立受鉴权下载，禁止放在公开资源目录。

## 交易状态与并发算法

| 原状态 | 动作/角色 | 新状态 | 库存变化 |
|---|---|---|---|
| 无 | USER 创建 | PENDING_CONFIRM | available -= qty，reserved += qty |
| PENDING_CONFIRM | 所属 ACTIVE 商家确认演示收款 | COMPLETED | reserved -= qty，on_hand -= qty |
| PENDING_CONFIRM | 本人取消 | CANCELLED | reserved -= qty，available += qty |
| PENDING_CONFIRM | 所属 ACTIVE 商家拒绝 | REJECTED | 同上 |
| PENDING_CONFIRM | 过期任务 | EXPIRED | 同上 |

库存不变量：on_hand = available + reserved，三者非负；用户看到的是 available。创建订单按 tenant_profile → 已排序的 product → 已排序的 SKU/inventory 加锁；生命周期变更按 tenant_profile → order → 已排序的 inventory 加锁。订单初建没有既有 order 锁。所有路径遵循此顺序，避免反向拿锁。商品编辑/治理同样先锁 tenant，再锁商品；库存调整先锁 tenant 再按 SKU 排序锁库存。

创建时同一事务完成：幂等占位、租户及商品可售校验、SKU 数量合并、后端重新计价、核对 quote 内容哈希（商品/价格/版本/数量/有效期）、库存预占、订单及明细快照、库存流水、业务审计、保存幂等结果。任一失败全部回滚。一个订单最多 50 个 SKU，每 SKU 1～999 件。客户端金额、tenant_id、user_id 不能作为可信事实。

确认时锁定订单后检查状态、商家、有效期；以数据库时间判断 now < expires_at。若已到期，在事务内关闭释放库存，事务提交后返回 ORDER_EXPIRED，不能抛异常导致释放回滚。其他终态返回状态冲突；相同幂等请求返回已有结果。更新订单、库存、唯一 sales_record(order_id)、审计在同一事务，故确认/取消/超时竞态最多一个成功。过期扫描每分钟批量领取，逐单事务加锁重新判定，多实例不得重复释放。

所有租户治理写入也锁 tenant_profile。这样“全部下架”成功后的下单不能越过开关；在锁前已提交的订单为合法历史订单，不回滚。普通读可不加锁，但展示缓存不参与交易授权；一期不缓存可售列表，以减少治理生效延迟。

## 销售、审计及后台任务

销售记录只在 COMPLETED 产生，时间取 confirmed_at，金额取订单快照 total_amount；每单一条主记录，明细按订单商品关联。销售金额不是支付平台结算金额，所有报表标题注明“演示确认销售”。待确认/取消/拒绝/过期订单不计入销售。统计订单数 count(distinct order_id)，件数 sum(quantity)，金额对订单或明细择一汇总，禁止 JOIN 放大。

业务审计与 mutation 同事务写 audit_log，字段 id/tenant_id/actor_id/actor_type/action/resource_type/resource_id/before_json/after_json/reason/request_id/result/created_at。脱敏密码、session、手机号、收货地址；应用账号只 INSERT/SELECT 审计，不提供删除/编辑接口。失败认证进入独立 security_log，不因业务回滚丢失。管理员只读聚合 commerce 商家审计及自身日志，不复制所有原文。

销售导出：创建 export_job（owner_id、tenant_id、filter_json、cutoff、status、expires_at）→ worker 使用租约领取 → 固定过滤条件及 cutoff 下读取不可变销售记录 → keyset 分页、POI SXSSF 写 XLSX → 原子保存文件并标记 SUCCEEDED。使用同一个 MySQL REPEATABLE READ 只读快照完成数据读取，避免尚未提交的早时间戳记录穿插；记录实际行数与生成时刻。最大 100000 明细行，超过要求缩小区间；最多重试 3 次，临时文件清理；租约过期可恢复，任务和文件 24 小时失效。下载再次核验租户/身份/状态/过期，不能凭 taskId 下载他人文件。单元格以字符串写用户文本，防止公式注入；金额使用数值单元格、两位小数。

## 联调与验收门槛

开发顺序：身份和租户申请 → 商品库存 → 用户购物车/订单 → 商家确认 → 平台治理 → 日志报表 → 故障及隔离验证。先约定 OpenAPI contract 1.0.0，再实现消费者与提供者契约测试；不兼容改动升级 API 主版本，三仓库 README 记录互相兼容版本。

拟定本地端口：user 前端 5173、seller 5174、admin 5175；gateway 8080、user-api 8081、seller-api 8082、admin-api 8083、identity 8090、commerce 8091；仅 gateway 和前端对外，内部服务/MySQL/Redis/Nacos 不对公网暴露。启动顺序为 MySQL/Redis/Nacos → identity 与 commerce（注册编排使用持久任务等待就绪）→ 各 API 与 gateway → 前端。联合 Compose 与环境示例后续归 shop-admin/backend/infra，其他仓库引用，不复制基础设施。当前不创建这些程序文件。

验收必须覆盖：两个租户商品/库存/订单/审计/导出互不可见；两个消费者互不可见；未审批账号不能营业；管理员部分/全部下架后无法新下单；100 个并发请求争抢最后 1 件最多成功 1 单；重复提交只产生 1 单；确认与取消/过期并发仅一次库存变更；数据库故障回滚；下游超时使用原幂等键恢复；停用后旧会话不能继续操作；统计与明细金额可对账。以上均为 ⚪ 待执行，不填写虚构测试结果。
