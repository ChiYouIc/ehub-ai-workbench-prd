# 02: 支付宝收款账户对公校验 + 币种限人民币

**What to build:** 录入供应商收款账户时，收款方式选择「支付宝」后形成完整前置拦截：账户名称必须匹配公司名称格式（以「公司」结尾，覆盖「有限公司」「有限责任公司」「股份有限公司」等后缀），否则保存被阻断并提示「支付宝付款只允许对公付款」；币种下拉仅提供人民币（CNY）一个选项，从其他收款方式切换到支付宝时已选的非人民币币种自动清空待重选。银行转账、微信收款方式的录入行为完全不变（仍允许个人姓名与任意币种）。新增与编辑保存均执行该校验。

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent → 已完成（2026-09-14 收敛：并入 ehub develop commit `5acc66c99`/`336478141`/`233c18822`，功能已验证，实现说明已同步 TAPD）

- [x] 支付宝 + 以「公司」结尾名称 + 人民币：保存成功
- [x] 支付宝 + 个人姓名（如「张三」）：保存被阻断，账户名称字段定位到错误并提示（实现文案为「仅支持企业支付宝账户」/ `alipayOnlyCorporate`，较 spec 原案「支付宝付款只允许对公付款」更精炼，已经用户确认采用）
- [x] 支付宝收款方式下币种下拉仅剩人民币（CNY）可选（computed `visibleCurrencies` 过滤）
- [x] 已选其他币种（如 USD）后切换收款方式到支付宝：币种被自动清空，需重选后方可保存（watch `form.proceedsWay`）
- [x] 银行转账 / 微信 + 个人姓名 + 非 CNY 币种：保存成功（回归，新校验不误伤）
- [x] 编辑已有支付宝收款账户保存时，同样执行名称正则与币种校验（含存量「支付宝+非 CNY」脏数据的 rules 兜底拦截）
- [x] 校验规则集中定义在表单 rules 层，与既有必填校验同层，不散落多处

## Comments

- 2026-09-14 已实现于 `ehub-web/src/views/supplier/components/suppliersUserAdd.vue`：`isAlipay` computed + `rules.accountHolderName`（公司结尾正则）+ `rules.currencyCode`（CNY 兜底）+ `visibleCurrencies` 过滤 + proceedsWay 切换清空币种。报错文案 key：`alipayOnlyCorporate`、`alipayOnlyCny`。
- 2026-09-14 补充实现期小改动（`suppliersUser.vue`，列表页联动）：① 收款方式显示映射修正为 3=微信、5=支付宝（原 2/3 映射有误，顺带修复）；② `editFrom` 回填 `row.proceedsWay`，编辑弹窗内支付宝校验与币种过滤才能正确生效。
- 2026-09-14 语言包最终落点为 `static/lang/zh.json`、`static/lang/en.json`（应用实际加载路径；初版误写 `static/langzh.json`/`langen.json` 已还原）。韩文由 `fallbackLocale: 'zh'` 兜底。
- 2026-09-14 功能已由用户手工验证（新增/编辑/切换收款方式/币种过滤场景）；实现说明已同步 TAPD story 1136062570001009401 评论（评论 ID 1136062570001012403）。
