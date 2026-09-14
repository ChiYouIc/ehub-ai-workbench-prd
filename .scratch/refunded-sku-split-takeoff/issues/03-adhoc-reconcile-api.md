# 03: 指定订单的临时对账接口（已关闭/已完成订单修复）

**What to build:** 新增指定订单的临时 REST 接口：对已完成(7)/已关闭-作废(8) 订单核对仓库侧是否存在未下架残留并补发下架，二者处理差异化——已完成(7) 与在库待发货/运输中**完全同逻辑不特判**（Q9）；已关闭-作废(8) **不区分退款直接关闭仓库侧所有订单**（Q10）。**不做批处理**——已完成/关闭订单量太大，线下发现问题后逐单调用修复；同时兼作 `splitOrTakeOffOrder` 的验证入口。运营视角：遇到卡单时给一个订单号即可自助修复。

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent → 已完成（2026-09-14 收敛：Q9/Q10 口径已按修订实现，验收以线上观察为准）

## 实现记录（2026-09-11，2026-09-14 收敛）

- `splitOrTakeOffOrder` 状态门槛放开：额外接受已完成(7)/已作废(8)；已作废(8) 走新增 `abolishedOrderTaskOff` 独立路径（提前返回：遍历全部正常仓库订单逐一 `returnOrder`，覆盖已拆单残留，不改任何订单状态）；已完成(7) 与待发货/运输中完全同逻辑不特判（全退 `closeOrder` 幂等重入）
- `sendShipNotice` 状态门同步放行已作废(8) 进入 `splitOrTakeOffOrder`
- 临时接口 `POST /temp/order/refundReconcile`（TempController）：参数 `orderId`；前置校验——订单存在、状态为已完成(7)/作废(8)，已完成单还须含 refund_status=12 商品项，不满足安全拒绝并返回原因；执行复用 `splitOrTakeOffOrder` 主链路（幂等：仓库无残留仅日志跳过）
- 测试：`ClosedOrderSplitTakeOffTest`——已完成/已作废订单重入不抛状态异常、处理前后订单状态不变；未配置真实 `PURCHASE_ORDER_ID` 时自动跳过
- ~~已关闭订单处理时仅下架仓库侧残留、不改状态~~ → Q9/Q10 修订已按新口径实现（作废走独立路径 `abolishedOrderTaskOff`）

## 决策溯源

Q3（`splitOrTakeOffOrder` 调整支持已关闭/已完成订单）、Q4（指定订单、不批处理）、Q9（已完成7 与待发货/运输中完全同逻辑不特判）、Q10（作废8 不区分退款直接关闭仓库侧所有订单、跳过 `closeOrder`）、Q13（接口复用 `splitOrTakeOffOrder`，内部新分支自动生效不受影响）。

## Acceptance criteria

- [x] `splitOrTakeOffOrder` 的采购订单状态门槛放开：在待发货/运输中之外额外接受已完成(7)/已关闭-作废(8)，供本接口调用；其它调用方行为不变
- [x] 已完成(7)：与在库待发货/运输中**完全同逻辑不特判**——全退下架 + `closeOrder`（重复置 7 幂等、客户订单置待评价），部分退照常拆单，入库检查照做（Q9）
- [x] 已关闭-作废(8)：不区分退款，直接关闭仓库侧对应的所有订单（`abolishedOrderTaskOff` 遍历全部正常仓库订单逐一 `returnOrder`，覆盖已拆单残留），**不调 `closeOrder`**、不改采购/客户订单状态，不做入库检查（Q10）
- [x] 临时 REST 接口（`TempController` 先例）入参为采购订单号，逐单触发；复用 `splitOrTakeOffOrder` 主链路，内部新分支自动生效（Q13）
- [x] 接口前置校验：已完成(7) 单须含 refund_status=12 商品项（否则安全拒绝）；已关闭-作废(8) 单不要求含退款项（Q10）
- [x] 仓库侧无残留（`getSplitOrderInfo` 为空）时安全返回"无残留"，不做多余调用
- [x] 部分退策略复用主链路逻辑（混合包裹仍走拆单+异步下架；仓库订单状态过滤维持待发货/待确认，其它状态打印日志跳过——Q1）
- [x] 关键失败落 errorRemark 且日志可追踪
- [x] 验收方式：`@SpringBootTest` 集成测试（`ClosedOrderSplitTakeOffTest`，依赖真实数据环境，移交开发者测试环境执行）
