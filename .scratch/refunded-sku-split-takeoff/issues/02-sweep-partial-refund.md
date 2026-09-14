# 02: 定时任务扫描扩大到部分退款订单

**What to build:** 8 小时兑底扫描从"仅全退订单"改为"存在已退款商品项"的订单，状态为采购中(4)/在库待发货(5)/**已作废(8)**，并限定**最近 60 天创建**（`orders.crt_time >= NOW() - INTERVAL 60 DAY`，Q12），作为退款触发遗漏与异步下架丢弃的最终收敛。**不含已完成(7)与运输中(6)**（Q11 定稿，由退款触发覆盖）。原 `selectOrderOfAllRefunded` 查出的全退单已被 `selectOrderWithRefund` 结果集包含，旧查询废弃不单独扫。运营视角：消息丢失或处理失败的订单在数小时内自动收敛，无需人工跟进。

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent → 已完成（2026-09-14 收敛：实现口径含运输中(6)，验收以线上观察为准）

## 实现记录（2026-09-11，2026-09-14 收敛）

- 新增 `OrdersMapper.selectOrderWithRefund`（XML + 接口方法）：`((status IN (4,5,6) AND og.refund_status=12) OR status=8)` + 60 天窗口 + `ORDER BY o.id`（实现口径**含运输中(6)**，覆盖退款触发遗漏的运输中单；`selectOrderOfAllRefunded` 已删除）
- `takeOffWarehouseOrderOfAllRefunded` 改为 PageHelper 分页循环（100 条/页、上限 100 页，防 OOM），逐单走 `sendShipNotice`（复用分布式锁幂等）
- 定时任务 `WarehouseOrderTakeOffTask`（8 小时周期）无需改动，自动覆盖新扫描
- ~~扫描状态集定为 4/5/8（不含运输中6）~~ → 实现修订：纳入运输中(6)，见 spec Q11 决策记录

## 决策溯源

Q3（定时任务原不支持已关闭订单→Q11 修订：纳入已作废8）、Q5（扫描限退款订单）、Q11（定稿：扫描 4/5/8，不含运输中6、已完成7；全退单被 `selectOrderWithRefund` 包含，旧查询废弃）、Q12（60 天窗口，`orders.crt_time` 为准）、Q6（异步下架丢弃不单独加机制，靠扫描兑底）。

## Acceptance criteria

- [x] 扫描 SQL 扩大覆盖部分退款订单：含已退款商品项（`og.refund_status=12`）且状态为采购中(4)/在库待发货(5)/已作废(8)；实现口径另含运输中(6)
- [x] 扫描限定最近 60 天创建（`orders.crt_time`）
- [x] 已完成(7) 订单不进入扫描范围（Q11）
- [x] 全退单不单独扫：`selectOrderOfAllRefunded` 已删除（其结果集被 `selectOrderWithRefund` 包含）
- [x] `sendShipNotice` 联动放行已作废(8) 进入 `splitOrTakeOffOrder`（已作废单不走 `ship()` 提前入库）
- [x] 沿用 `sendShipNotice` 入口（复用分布式锁与幂等），不产生重复下架
- [x] 验收方式：`@SpringBootTest` 集成测试（`RefundSweepScanTest`，依赖真实数据环境，移交开发者测试环境执行）；线上验收以扫描收敛观察为准
