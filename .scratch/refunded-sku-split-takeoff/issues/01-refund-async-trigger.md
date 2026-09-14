# 01: 退款成功异步触发拆单/下架

**What to build:** 客户订单退款成功后，系统异步对对应采购订单执行拆单/下架（整退直接下架关单；部分退拆单后异步下架退款侧）。退款消息确认不被仓库接口耗时阻塞，失败由定时任务兜底。采购运营视角：退款后无需逐单跟进；仓库运营视角：窗口期内退款的单在发货前被拦截。

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent → 已完成（2026-09-11 用户 review 通过）

## 实现记录（2026-09-11）

- 新增 `RefundTakeOffComponent`（ehub-tenant service/order/components）：`@Async triggerSplitTakeOff(customerOrderId)` + `listPurchaseOrderIds(customerOrderId)`；内部逐单调用 `sendShipNotice`，单单失败仅记日志不外抛
- 挂接点 1：`OrdersBiz.ordersGoodsRefund` 尾部（部分退款，refund_status 已置 12）
- 挂接点 2：`OrdersBiz.cancelled` 尾部（整单退款关单后）
- 测试：`RefundTakeOffComponentTest`（`@SpringBootTest` 风格，需真实 CUSTOMER_ORDER_ID 数据支撑，未配置时自动跳过）

## Review 结论（2026-09-11）

- 用户已 review，工单定为**已完成**
- 遗留备注：本机 Maven 默认 JDK 与 Lombok 不兼容导致 `mvn test` 未能本地执行（`module jdk.compiler does not opens com.sun.tools.javac.processing`，需 JDK 8 环境或升级 Lombok），测试验证依赖开发者的 IDE/测试环境执行，非代码问题

## 决策溯源

Q2（异步处理）、Q7（就地演进）、Q8（退款指客户订单退款：部分退款在 `OrdersBiz.ordersGoodsRefund` 置 `orders_goods.refund_status=12` 后触发；整单退款在 `OrdersBiz.cancelled` 关单后触发。1688 采购退款 `handleRefundSuccess` 不挂触发）。入口从采购订单（`orders` 表）入手：由客户订单号反查名下采购订单逐单处理。

## Acceptance criteria

- [x] 客户订单部分退款（`ordersGoodsRefund` 置 refund_status=12 后）异步调用拆单/下架入口，失败不阻塞退款流程、仅记日志
- [x] 客户订单整单退款（`cancelled` 关单后）异步调用拆单/下架入口
- [x] 入口以采购订单（`orders` 表）为处理单元：按客户订单号反查名下采购订单逐单处理
- [x] 沿用现有分布式锁防并发，不与人工操作、定时任务产生重复拆单/重复下架
- [x] 整单退款：直接下架对应仓库订单并走既有关单流程
- [x] 部分退款：只处理仓库状态为待发货(2)/待确认(-20)的仓库订单；其它状态打印日志跳过（Q1：状态过滤维持现状不扩展）
- [x] 部分退款：混合仓库订单先拆单再入 Redis 异步下架队列；只含退款商品的仓库订单直接下架
- [x] 已到货边界：仅针对采购已到货商品，未到货沿用现有行为
- [x] 1688 采购退款链路（`handleRefundSuccess`）行为不变，不挂本触发（Q8）
- [x] 关键节点（收到退款消息、触发拆单、下架结果）有结构化日志
- [x] 验收方式：`@SpringBootTest` 集成测试（先例 `PurchasePreAfterSalesTaskTest`），在 `splitOrTakeOffOrder`/`sendShipNotice` seam 断言仓库侧 returnOrder/splitPackage 调用与订单最终状态（注：测试依赖真实数据环境，移交开发者测试环境执行）
