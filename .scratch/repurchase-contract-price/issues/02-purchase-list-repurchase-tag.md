# 02: 待下单清单合约信息标记（price_info 快照存在性展示）

**What to build:** 前端待下单清单（`/purchase/purchaseOrderManagement?showType=1&status=2` 路径，`waitingList.vue`）每个 SKU 行解析后端透传的 `priceInfo`（`ProductOrdersDto.priceInfo`，人工协议价元素 + repurchase 元素同数组共存）：存在 `repurchase:true` 且 `expired !== true` 的元素 → 在 AP 标签下方展示「复购合约」标签（`el-tag` success + dark）；存在 `repurchase:true` 且 `expired === true` → 展示红色（danger）「复购合约已失效」标签；无 repurchase 元素 → 不展示。tooltip 有效/失效均展示阶梯价（分转元）与同步时间——失效不删元素、保留快照供追溯（spec 决策 2 第 4 条），失效态以 tag 文案+颜色表达，tooltip 不另行替换文案。

范围说明：本工单覆盖两处存在性标记（数据源均为 `third_ali_product_info.price_info` 快照，由下单预览触发的 `RepurchaseContractSyncComponent` 同步）：
1. 待下单清单（`showType=1&status=2`，前端解析 `priceInfo`）；
2. 采购订单列表（tabPane 默认视图，后端解析：`OrderQueryServiceImpl#buildContractPrice` 补查 `priceInfo` 列 → `TenantOrdersGoodsVo.hasRepurchase / repurchaseExpired`）。

两处共用解析语义；后端解析落在 `ehub-common` 的 `RepurchasePriceInfoUtils`（静态工具，不依赖 tenant 实体，后续其它列表/模块可复用，含 `matchTierPrice` 档位匹配供价格展示工单使用）。按采购数量匹配档位、合约价替换展示、flow 切换（spec 第 9 条其余内容）属后续工单。

数据链路：
- 待下单清单：`OrderListingServiceImpl#listProductSkuOrders` → `getCurrSupplierThirdProductId` 已把 `thirdAliProductInfo.getPriceInfo()` 透传到 `ProductOrdersDto.priceInfo`；
- 采购订单列表：`OrderQueryServiceImpl#buildContractPrice` → `fillApAndRepurchase`（本次新增）。

**Blocked by:** None (01 的同步组件已落地，priceInfo 已透传).

**Status:** ready-for-human（前端无自动化测试基建，人工验收）

- [x] `ehub-common` 新增 `RepurchasePriceInfoUtils`：`parse`（返回不可变合约快照）/ `hasActive` / `hasExpired` / `matchTierPrice`（分起批量最高档，同量多档取最低价兜底）/ `isSameContract`（忽略 updTime）；脏数据返回无合约不抛错
- [x] `TenantOrdersGoodsVo` 新增 `hasRepurchase` / `repurchaseExpired` 字段
- [x] `OrderQueryServiceImpl#buildContractPrice` 补查 `priceInfo` 列，分组条件扩展为「有协议价 或 有 priceInfo」，行匹配逻辑（thirdProductId 优先，回退 goodsId+supplierId）与 hasAp 一致，抽 `fillApAndRepurchase` 复用
- [x] `waitingList.vue` 单价区（AP 标签下方）新增复购合约标签：有效 success / 过期 danger（文案「复购合约」/「复购合约已失效」），均 `effect="dark" size="mini"`
- [x] `waitingList.vue` tooltip：有效/失效合约均展示 skuId 阶梯价（`price` 分转元，`begin-end` 件区间）+ `updTime`（失效不删元素、快照供追溯；失效态由 tag 文案「复购合约已失效」+ danger 色表达）
- [x] `waitingList.vue` 解析容错：`priceInfo` 为空 / 非 JSON 数组 / JSON 解析异常 → 不展示标签（不抛错）
- [x] `tabPane.vue` 商品名 tag 串（AP tag 后）新增复购合约标签：有效 success / 过期 danger（后端聚合布尔，有效优先；文案「复购合约」/「复购合约已失效」）
- [x] 协议价管理页（showType=4）合约只读：`editProductInfo.vue#getData` 解析时把 repurchase 元素从 `priceInfoList` 剥离挂到 `row.repurchase`（含 JSON 解析容错）；`editProductInfoTable.vue` SKU 列（goodsId 行内）追加只读合约 tag（生效中 primary / 已失效 danger，tooltip 阶梯价/同步时间，无合约不展示）；编辑/批量保存提交数组天然不含合约元素，后端不做写入保底（合约由同步任务自愈）
- [ ] 人工验收：待下单清单与采购订单列表——有有效合约的 SKU 行展示 primary 标签；有 expired 元素的行展示 danger 标签；纯人工协议价 / 无 priceInfo 行为不变
- [ ] 人工验收：协议价管理页——合约行展示只读列；编辑保存后合约列不变（若被覆盖，下次下单预览自愈）
