# shop-user — 用户商城

多租户电商演示项目，当前为 **仅开发文档阶段**。功能实现状态全部为 **⚪ 待实现**。

## 根目录文档

- [前端开发文档](FRONTEND_DEVELOPMENT.md)：模块 A～F，功能编号、交互、接口示例、验收和状态。
- [后端开发文档](BACKEND_DEVELOPMENT.md)：模块 A～F，业务算法、数据模型、权限、事务、接口示例和状态。
- [共同架构与接口约定](COMMON_ARCHITECTURE.md)：技术版本、服务归属、多租户隔离、订单状态机、联合验收。

## 目录约定

| 路径 | 用途 |
|---|---|
| frontend/ | 后续 Vue 3 + Pinia 前端；当前仅 .gitkeep 占位 |
| backend/ | 后续 Java 21 + Spring Cloud Alibaba 后端；当前仅 .gitkeep 占位 |
| *.md（根目录） | 开发文档，不能移入程序子目录 |

无应用程序、数据库迁移或可运行构建文件，当前不能启动商城；文档中的接口是开发合同。

## 三仓库协作

- [shop-user](https://github.com/CareyChi/shop-user)：商城、用户 API、网关。
- [shop-seller](https://github.com/CareyChi/shop-seller)：商家后台、商家 API、唯一交易服务。
- [shop-admin](https://github.com/CareyChi/shop-admin)：管理员后台、平台 API、统一身份服务。

合同版本 **1.0.0**，三仓库文档基线 **1.0.0**。共同架构文档的变更需同步三仓库；商家为租户，用户可跨店浏览、按店结算，收款由商家手动演示确认，不接入支付。
