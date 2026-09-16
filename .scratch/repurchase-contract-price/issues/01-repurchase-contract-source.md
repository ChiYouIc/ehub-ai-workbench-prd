# 01: 复购合约数据源：1688 API 查询服务 + price_info 合约元素 + 定时巡检

**What to build:** 系统成为复购合约的权威数据源：通过 1688 开放平台 API（`com.alibaba.trade:repurchase.contract.get-1`）拉取商品规格级的生效合约，以行级 `repurchase:true` 元素形式合并进 `third_ali_product_info.price_info`（不建新表/新列，人工协议价元素同数组共存），并定时保持与 1688 一致；对外提供一个查询服务，按（1688 商品 offerId、specId、采购数量、收货地址编码）返回合约报价对象（是否命中、命中档位价、运费/包邮性、开票、交期、有效期、合约状态）。后续价格展示、下单 flow、付款校验均消费该服务，本工单不含这些消费方。

架构决策（已与提出方确认）：不动人工协议价体系——`third_ali_product_info`（AP）不加新列、不建新表，合约快照以带 `repurchase:true` 标识的 JSON 元素合并进 `price_info` 数组（元素含 skuId、阶梯价 priceRanges（price 单位分）、updTime、expired 标记），通过行级 offerId + specId 自然关联；合约失效（到期/终止）把元素内 `expired` 置 true 而非删除。

现状提示：alipur 模块已手写该 API 的 param/result 类（沿用仓库手写 param 类的惯例），但 result 模型缺少合约状态、有效期起止等字段，实现时须对照 1688 官方 API 文档核实并补齐，否则到期/终止兜底无数据依据。网关调用、token/子账号处理、同步任务、单测风格均跟随仓库现有先例（Mockito 直测服务类）。

**Blocked by:** None (can start immediately).

**Status:** ready-for-agent

- [ ] 1688 网关 client 新增复购合约查询方法，入参复用已有 param 类；对照官方文档核实并补齐 result 缺失字段（合约状态、有效期起止等）
- [ ] 查询服务契约：输入（offerId、specId、采购数量、收货地址编码），输出合约报价对象：是否命中、合约 ID、命中档位（序号/起批量/单价/运费）、是否包邮（地址维度）、开票方式、交期、有效期起止、合约状态
- [ ] 档位匹配规则：采购数量落入「满足起批量的最高档」取该档单价；低于最低档视为不达档
- [ ] 地址覆盖判断：包邮区内 / 区外 / 合约未约定包邮地区 三种结果
- [ ] 有效期与状态判定：未生效 / 生效中 / 到期 / 已终止
- [ ] 合约快照存储：以 `repurchase:true` 元素合并进 `third_ali_product_info.price_info`（元素结构：skuId、priceRanges 阶梯价（分）、updTime、expired），人工协议价元素原样保留；无合约时把元素内 `expired` 置 true 标记失效（不删除元素，保留 skuId 与阶梯价快照供追溯；合约恢复时原位替换并翻回 `expired=false`）；脏数据（非 JSON 数组）降级保留现状；已有 alipur 查询 API（`POST /restful/ali/trade/repurchase/contract`）与 tenant 同步组件（`RepurchaseContractSyncComponent`，下单预览触发）可复用
- [ ] 定时巡检任务：增量同步合约的新建、续签、变更、到期、终止；到期/终止把元素内 `expired` 置 true 而非删除记录
- [ ] 同步任务失败时产生告警
- [ ] 缓存策略：查询优先命中快照缓存，实时查询结果覆盖快照，保证消费方拿到的是最新合约状态
- [ ] Seam A 单元测试（JUnit4 + Mockito，mock 1688 网关 client）：档位匹配边界（恰好达档 / 差一件不达档 / 数量跨多档取最高满足档 / 无合约）、地址覆盖三态、有效期与状态四态、price_info 合并/标记失效/幂等（合约内容无变化不写库不刷 updTime；已失效仍无合约不重复写库）、脏数据降级
- [ ] 术语「复购合约」「合约价」「人工协议价（AP）」补入 PRD 仓库 GLOSSARY.md，与现有「合约采购」「重采」区分
