# shop-user · 用户商城 · 后端开发文档

版本 1.0.0；日期 2026-09-27；接口合同 1.0.0。

本次仅编写文档。功能状态：⚪ 待实现；✔ 已实现。本文所有功能均为 ⚪，未编写或运行应用代码。

先阅读 [共同架构与约定](COMMON_ARCHITECTURE.md)，再按本文模块开发。前后端相同编号代表同一业务能力，但实现状态独立维护。示例均为虚构演示数据，不是生产账户或实际响应。

本仓库后端模块规划：user-api、gateway。所有对外接口以下述路径为相对地址，统一前缀 `/api/v1/user`；完整路由由网关转发至本端 API。

## 后端结构及实现约束

所有后续 Java 程序放 backend/；Maven 聚合根位于 backend/pom.xml，每个规划服务独立子模块。服务内按 domain/application/infrastructure/interfaces 分层，再按 A～F 对应领域组织；Controller 只处理认证入口/DTO/参数，Application 组织用例，领域服务保持状态规则，Repository 负责带作用域持久化。数据库迁移放所属服务 src/main/resources/db/migration，JUnit 测试放 src/test/java。

REST DTO 不直接暴露 JPA Entity；禁止 Open Session in View，读路径投影 DTO，列表避免 N+1，分页 count 与数据使用相同过滤。领域写事务放服务层；本地事务不能跨 Feign 请求假装原子，跨服务注册用持久幂等任务。库存与订单仅由 commerce 同库事务更新，其他 BFF 不复制实现。

Gateway、BFF 和内部服务均需验证各自边界；内部服务不能因位于内网就省略用户权限。所有自由文本限长，SQL 参数绑定，sort 白名单；生产错误不暴露 SQL/堆栈。连接池、服务调用、锁等待设置有界超时；死锁仅允许以原幂等键有限重试。GET 限制最大 pageSize，导出限制总量。

Flyway 每服务管理自己的数据库，已部署迁移不可修改；开发环境种子只包含虚构账号资料，密码从环境提供且不写 Git。应用账号最小权限，迁移账号与运行账号分离。密钥和环境值用 .env.example 占位设计，实际 .env、上传资源、导出文件、构建产物不进入仓库。

## 功能总表

| 模块 | 功能编号 | 功能 | 状态 |
|---|---|---|---|
| A 账户与会话 | A01 | 消费者注册 | ⚪ 待实现 |
| A 账户与会话 | A02 | 登录、当前身份与退出 | ⚪ 待实现 |
| A 账户与会话 | A03 | 个人收货地址 | ⚪ 待实现 |
| B 商品查询 | B01 | 商品列表及搜索 | ⚪ 待实现 |
| B 商品查询 | B02 | 详情与 SKU 选择 | ⚪ 待实现 |
| C 购物车 | C01 | 加购及购物车读取 | ⚪ 待实现 |
| C 购物车 | C02 | 修改数量、勾选与移除 | ⚪ 待实现 |
| D 下单与订单生命周期 | D01 | 结算预览 | ⚪ 待实现 |
| D 下单与订单生命周期 | D02 | 提交订单与结果恢复 | ⚪ 待实现 |
| D 下单与订单生命周期 | D03 | 取消待确认订单 | ⚪ 待实现 |
| E 历史订单 | E01 | 列表和搜索 | ⚪ 待实现 |
| E 历史订单 | E02 | 订单详情和状态刷新 | ⚪ 待实现 |
| F 基础能力与验收 | F01 | 异常处理与安全导航 | ⚪ 待实现 |
| F 基础能力与验收 | F02 | 端到端业务与隔离测试 | ⚪ 待实现 |

## 模块 A：账户与会话

### A01 消费者注册

**功能实现状态：⚪ 待实现**

**目标与接口：** `POST /auth/register`。

**具体实现思路：** identity 按 USER 域规范化用户名并以 (audience,normalized_username) 唯一约束防重；BCrypt 加盐散列，事务写账户；频率按 IP 与用户名双维度限制。

**联调约束：** 注册页包含用户名、密码、确认密码；本地校验长度和一致性，提交期间锁按钮；成功跳转登录，不自动猜测登录成功。

**示例输入与输出：** 输入 {"username":"buyer_demo","password":"ExampleOnly_123"} → 201 {"accountId":"101","status":"ACTIVE"}；重复返回 409 ACCOUNT_EXISTS。

**验收条件：** 并发同名注册只创建一个账户；响应与日志均无密码。

### A02 登录、当前身份与退出

**功能实现状态：⚪ 待实现**

**目标与接口：** `POST /auth/login；GET /auth/me；POST /auth/logout；GET /auth/csrf`。

**具体实现思路：** user-api 代理 identity，强制 audience=USER；验证凭据并建立会话，失败统一 INVALID_CREDENTIALS；注销销毁服务端 session，cookie 失效。匿名 CSRF 获取、Origin 验证和限流按共同约定。

**联调约束：** authStore 只保存 me 与 permissions；刷新先恢复 me，再执行路由守卫；401 清状态并回登录，保留站内 returnTo；退出清购物车内存。

**示例输入与输出：** 输入 {"username":"buyer_demo","password":"ExampleOnly_123"} → {"accountId":"101","role":"USER"} + Set-Cookie；退出 → {"loggedOut":true}。

**验收条件：** 伪造角色不可提升权限；退出后旧 cookie 无效；returnTo 禁止外站 URL。

### A03 个人收货地址

**功能实现状态：⚪ 待实现**

**目标与接口：** `GET/POST /addresses；PATCH/DELETE /addresses/{id}`。

**具体实现思路：** 地址表归 user-api；所有操作按 user_id 筛选，电话作为字符串校验、列表脱敏；设默认地址在用户级锁下取消旧默认并设新默认。下单传服务端读取的地址快照，历史订单不跟随地址修改。

**联调约束：** 地址页维护收件人、电话、省市区和详细地址；结算页选择地址；删除二次确认，字段错误定位表单；最多 20 条。

**示例输入与输出：** POST {"receiver":"演示用户","phone":"13800000000","region":"广东省东莞市","detail":"演示地址","isDefault":true} → {"id":"201"}。

**验收条件：** 其他消费者地址返回 404；修改地址不改变历史订单。

## 模块 B：商品查询

### B01 商品列表及搜索

**功能实现状态：⚪ 待实现**

**目标与接口：** `GET /products?keyword=&categoryId=&tenantId=&page=1&pageSize=20&sort=createdAtDesc`。

**具体实现思路：** user-api 调 commerce 公开读模型；只查询可售商家和商品，按名称参数化 LIKE，分类/店铺组合索引；sort 白名单 priceAsc/priceDesc/createdAtDesc；响应不含敏感字段。

**联调约束：** 商城首页含搜索、分类、店铺筛选和分页；筛选状态写 URL；输入 300ms 防抖并取消旧请求，空结果、错误、重试分别显示。

**示例输入与输出：** GET /products?keyword=水杯 → {"items":[{"id":"301","tenantId":"401","name":"水杯","minPrice":"39.90"}],"page":1,"pageSize":20,"total":1}。

**验收条件：** 下架商品搜索不到；快速切关键词不能被旧响应覆盖。

### B02 详情与 SKU 选择

**功能实现状态：⚪ 待实现**

**目标与接口：** `GET /products/{id}`。

**具体实现思路：** commerce 查询商品可见性及 SKU 白名单 DTO；输出 skuId、attributes、price、available、version；不返回成本、reserved 或内部封禁理由；不可见资源统一 404。

**联调约束：** 详情展示图片、店铺、商品说明和规格组合；选 SKU 后更新价格/可售量，缺货禁加购；进入失效商品显示已下架并提供返回。

**示例输入与输出：** GET /products/301 → {"id":"301","skus":[{"id":"501","attributes":{"颜色":"蓝"},"price":"39.90","available":8,"version":1}]}。

**验收条件：** 不能用详情 ID 绕过下架；零库存仍可展示但不可购买。

## 模块 C：购物车

### C01 加购及购物车读取

**功能实现状态：⚪ 待实现**

**目标与接口：** `POST /cart/items；GET /cart`。

**具体实现思路：** user-api 持久化 cart_item，唯一 (user_id,sku_id)；加购验证 SKU 来源，quantityDelta 为 1～999，合并后不超过 999；不预占库存。列表批量向 commerce 查询 SKU 当前状态，避免 N+1。

**联调约束：** cartStore 管数量角标和最近获取结果；按店铺分组，失效项单列；未登录点加购跳登录；显示服务端最新价格和失效原因。

**示例输入与输出：** POST {"skuId":"501","quantityDelta":2} → {"itemId":"601","quantity":2}。

**验收条件：** 两设备购物车一致；加购失败不增加角标；缺货和下架保留失效项提示。

### C02 修改数量、勾选与移除

**功能实现状态：⚪ 待实现**

**目标与接口：** `PATCH /cart/items/{id}；DELETE /cart/items/{id}`。

**具体实现思路：** UPDATE 以 id/user_id/version 为条件，版本递增；数量 1～999，0 必须用删除；DELETE 按归属幂等。勾选不进入持久化业务表，提交时仅发送选中 SKU。

**联调约束：** 数量输入提交绝对值及 version；选择只保存在当前结算视图，按店独立全选；失败恢复上次服务端值；单项/多项删除逐项显示结果。

**示例输入与输出：** PATCH /cart/items/601 {"quantity":3,"version":1} → {"quantity":3,"version":2}。

**验收条件：** 并发编辑得到 VERSION_CONFLICT；他人 itemId 返回 404。

## 模块 D：下单与订单生命周期

### D01 结算预览

**功能实现状态：⚪ 待实现**

**目标与接口：** `POST /orders/preview`。

**具体实现思路：** user-api 校验地址所有权，将地址快照和选项送 commerce；commerce 重新读取商品价格、库存与租户，返回 5 分钟有效的 quoteToken（签名绑定 userId/tenantId/项目/版本/地址摘要/到期时间），不预占库存。

**联调约束：** 只能选同一店铺商品进入结算；显示地址、商品快照、金额、0 运费、24 小时待确认说明；金额变化要求重新确认。

**示例输入与输出：** 输入 {"addressId":"201","items":[{"skuId":"501","quantity":2}]} → {"quoteToken":"opaque-demo","totalAmount":"79.80","freight":"0.00","expiresAt":"2026-09-27T12:00:00Z"}。

**验收条件：** 跨店输入返回 MULTI_TENANT_CHECKOUT；预览不扣库存；无地址不能提交。

### D02 提交订单与结果恢复

**功能实现状态：⚪ 待实现**

**目标与接口：** `POST /orders；GET /orders/submissions/{idempotencyKey}`。

**具体实现思路：** user-api 不接收可信金额和用户 ID；重读地址并调用 commerce 本地交易，验证 quoteToken 与快照一致，详见共同并发算法。成功后条件清理 cart_item：只删除对应提交时版本和数量仍未变化的条目；清理失败不回滚订单，可重试。

**联调约束：** 每次用户确认生成一次幂等键，未知结果时持有原键；按钮锁定；超时查询提交结果，不直接新建订单；成功后展示订单号和待商家确认。

**示例输入与输出：** Idempotency-Key: checkout-demo-01；输入 {"quoteToken":"opaque-demo","addressId":"201","items":[{"skuId":"501","quantity":2}]} → 201 {"orderId":"701","status":"PENDING_CONFIRM","totalAmount":"79.80"}。

**验收条件：** 重复提交只生成一个订单；改价返回 QUOTE_CHANGED；缺货事务整体回滚；未知键查询返回 404 SUBMISSION_NOT_FOUND，可用原键重试。

### D03 取消待确认订单

**功能实现状态：⚪ 待实现**

**目标与接口：** `POST /orders/{id}/cancel`。

**具体实现思路：** commerce 根据 user_id 验归属，锁租户/订单/库存；状态仍待确认则写 CANCELLED、库存释放和审计；若商家已确认返回 ORDER_STATE_CONFLICT，不能覆盖。

**联调约束：** 详情页仅待确认显示取消；弹窗告知释放库存；提交 reason、version 与幂等键；冲突后刷新真实状态。

**示例输入与输出：** 输入 {"reason":"不再需要","version":1} → {"orderId":"701","status":"CANCELLED","version":2}。

**验收条件：** 终态禁止取消；取消与确认竞态不出现双重库存变更。

## 模块 E：历史订单

### E01 列表和搜索

**功能实现状态：⚪ 待实现**

**目标与接口：** `GET /orders?keyword=&status=&start=&end=&page=1&pageSize=20`。

**具体实现思路：** commerce 强制 user_id 条件，订单号精确/前缀及明细商品快照名 LIKE；用 EXISTS 搜明细防止重复订单；按 created_at DESC,id DESC；日期筛选下单时间。

**联调约束：** 订单页支持订单号/商品名称关键词、状态、下单日期；保留 URL 条件；状态中文映射统一；返回列表恢复页码。

**示例输入与输出：** GET /orders?keyword=水杯&status=COMPLETED → {"items":[{"id":"701","status":"COMPLETED","totalAmount":"79.80"}],"page":1,"pageSize":20,"total":1}。

**验收条件：** 商品改名后仍可按购买时名称搜到；订单分页不重不漏。

### E02 订单详情和状态刷新

**功能实现状态：⚪ 待实现**

**目标与接口：** `GET /orders/{id}`。

**具体实现思路：** commerce 只读本人订单、明细和状态事件；按 DTO 脱敏非必要信息；返回 serverTime 和 expiresAt，倒计时仅展示，最终状态按服务端。

**联调约束：** 详情显示下单快照、收款确认时间、截止时间、取消理由；待确认时每 15 秒轻量刷新，离开页面/隐藏标签停止；不放任何支付按钮。

**示例输入与输出：** GET /orders/701 → {"id":"701","status":"COMPLETED","confirmationMode":"MANUAL_DEMO","confirmedAt":"2026-09-27T12:30:00Z","totalAmount":"79.80"}。

**验收条件：** 商家确认后可见已完成；修改商品/地址不改变快照；他人订单统一 404。

## 模块 F：基础能力与验收

### F01 异常处理与安全导航

**功能实现状态：⚪ 待实现**

**目标与接口：** `全端请求封装及路由守卫`。

**具体实现思路：** gateway 路由 USER 范围；校验 CSRF/Origin、限流；user-api 验参并转换下游业务错误，保持 requestId，禁止返回堆栈。

**联调约束：** 统一处理 loading/empty/error/403/404/409；fetch credentials=include；写请求加 CSRF token；组件卸载取消请求；Pinia 不存凭据；登录重定向仅白名单站内路径。

**示例输入与输出：** HTTP 409 → {"code":"OUT_OF_STOCK","message":"库存不足","data":null,"requestId":"req-001"}。

**验收条件：** 直接访问受保护路由需登录；故障能重试且不显示虚假成功。

### F02 端到端业务与隔离测试

**功能实现状态：⚪ 待实现**

**目标与接口：** `契约与浏览器验收，无新增业务接口`。

**具体实现思路：** JUnit + MySQL Testcontainers 验地址/购物车归属；契约测试校验 commerce DTO；联合并发与故障测试覆盖共同文档列举的不变量。

**联调约束：** Playwright 串联注册、搜索、加购、预览、下单、等待确认、历史搜索和取消；用两个消费者验证页面不混态。

**示例输入与输出：** 测试夹具：消费者 U1/U2，商家 T1/T2，SKU501 库存 1；预期最多 1 单预占成功。

**验收条件：** 测试全部通过并保存报告后才允许将关联功能状态改为 ✔。

## 本仓库数据模型

| 表 | 核心字段 | 约束与索引 |
|---|---|---|
| user_address | id BIGINT、user_id BIGINT、receiver VARCHAR(50)、phone VARCHAR(32)、region VARCHAR(120)、detail VARCHAR(300)、is_default BOOLEAN、version INT、created_at/updated_at DATETIME(3) | INDEX(user_id,id)；用户最多 20 条；以 user_address_guard.user_id 行锁串行化默认地址切换 |
| user_address_guard | user_id BIGINT PK | 按用户初始化，仅作地址集合锁；不复制身份凭据 |
| cart_item | id、user_id、tenant_id、product_id、sku_id、quantity INT、version INT、created_at/updated_at | UNIQUE(user_id,sku_id)，INDEX(user_id,tenant_id,id)；CHECK quantity BETWEEN 1 AND 999 |

购物车 tenant_id/product_id 由 commerce 返回，不信任提交值。跨服务 ID 不建立跨库外键；商品删除采用归档，购物车读取发现失效返回明确状态。用户身份、订单、支付确认不在本库建第二份表。地址修改锁 guard 再锁 address，默认地址删除后设剩余最新一条为默认。

## 实施步骤及交付判定

1. 确认共同接口合同与版本锁定，建立工程骨架、认证及统一错误处理；本次不生成该骨架。
2. 按 A→B→C 的业务依赖实现，再补齐 D/E 的查询和记录；F 的安全约束从第一模块贯穿实施，不留到最后才加租户过滤。
3. 每个编号至少有正常路径、非法参数、无权限/非所属、状态冲突、重复提交（若为写操作）验收记录。前端组件测试与后端业务测试各自独立。
4. 使用共同文档的联合验收场景完成三仓库联调。示例响应由 mock 驱动界面时必须注明 mock，不视为后端完成。
5. 完成后只更新真正通过的功能状态为 ✔，附实现提交 SHA、测试报告位置和日期；其余继续 ⚪，不因为“文档已完成”而标程序已实现。

## 非本期范围

真实支付/退款、物流、优惠券、商家员工系统、短信邮箱验证服务、复杂多币种、跨店原子结算、搜索引擎和数据仓库不在本期；未设计的增量需求需新增功能编号及兼容合同后实现。管理员销售导出不是本次必需项，商家 XLSX 导出为本期必需项。
