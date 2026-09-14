# 01: 收款账户表单「持有人姓名」改为「公司名称」

**What to build:** 打开供应商收款账户新增/编辑弹窗，账户名称字段的标签由「持有人姓名」显示为「公司名称」，输入占位符由「请输入账户持有人姓名」显示为「请输入公司名称」；中英文语言包补齐对应文案。对银行转账/微信/支付宝全部收款方式统一生效，不改变任何校验与保存行为。范围仅限录入表单，列表页「账户持有人姓名」列头不在本工单内。

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent → 已完成（2026-09-14 收敛：并入 ehub develop commit `5acc66c99`，功能已验证）

- [x] 新增/编辑弹窗中该字段标签显示为「公司名称」，占位符显示为「请输入公司名称」
- [x] 中文、英文两种界面语言下文案均正确显示（复用既有 key：`companyName` / `pleaseEnter`，无需新增翻译）
- [x] 银行转账、微信、支付宝三种收款方式下表单均正常打开与保存（纯文案改动，回归确认）
- [x] 字段绑定的数据字段与保存接口不变

## Comments

- 2026-09-14 已实现于 `ehub-web/src/views/supplier/components/suppliersUserAdd.vue`：label/placeholder 复用 i18n 既有 key `companyName`、`pleaseEnter`（中英文已存在），未新增语言 key。功能已手工验证。
