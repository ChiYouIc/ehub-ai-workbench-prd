# AI 对话 — UI 设计文档

> 模块：ai-chat ｜ 关联：交互草图 `ai-chat-ia.md`（r3）｜ **工程 UI 基线 `designs/ui-baseline.md`（r1）** ｜ 状态：**r3（2026-08-25，新增 P6 思考与工具调用 v2 预览画板）**
>
> 本文档是基线的**模块实例化**：色彩 token、状态色映射规则、布局范式、治理规则均继承 `designs/ui-baseline.md`，此处只落模块映射表与模块专属细节。高保真原型 `ai-chat-pages.pen`（同目录）六画板 P1~P6（P6 为 v2 预览）。

## 1. 基线继承声明

- 继承 `designs/ui-baseline.md`（r1）全部条款，无 token 级偏离。
- 色彩 token 直接引用基线 §2，不重复抄录。
- **布局范式登记**：对话页为**单栏消息流 + 吸底输入区**布局，不适用基线 §4.1「列表页三段式」——这不是对基线的偏离，而是该范式的适用范围本就不含对话流页面（三段式适用于实体列表页）；本模块按 ia.md §3 的三态画板实例化，登记于此以备走查对照。

## 2. 状态色彩映射（基线 §3 规则 → 本模块映射表）

依据 `ai-chat-design.md` §5 状态机与 `ai-chat-ia.md` §5 状态展示表，按基线 §3 规则落成本模块映射：

| 领域状态 | 组件 | 色彩语义 | 说明 |
|---|---|---|---|
| streaming（流式进行中） | 用户气泡 / 输入区 / 操作底行 | primary / 禁用 / success | 用户气泡 primary 底白字（右侧）；输入区整体 opacity 0.5 禁用 + 停止按钮；操作底行右侧 success 绿点 + 「生成中」 |
| done（end 事件） | 操作底行右侧 | secondary | 「生成中」标记消失；token 汇总灰色次要文本 + 时间戳 |
| error（error 事件） | 内联错误卡 | danger | tag_danger_bg 填充 + danger 边框 + danger 文案（含错误码）+ 「重试」按钮（danger 描边） |
| interrupted（中断） | 操作底行右侧 | secondary / info | info 灰点 + 「已中断」secondary + 时间戳，**无 token 统计**；**无错误卡**（与 error 分呈现） |
| empty（空态） | 欢迎区 / 输入区 | primary 文本 | 居中欢迎区（图标 + 主副文案 primary/secondary）；输入框下方提示文案 secondary |
| @提及 chip | 输入框内 chip | primary_light 底 + el_primary 文字 | 选中成员后插入，圆角 4 |
| /指令选中行 | 弹层列表项 | el_active_bg / el_primary | 选中行高亮 + check 图标 |
| 入参错误（PARAM_ERROR） | el-message / toast | danger | 「请输入内容后再发送」等随服务端信息呈现 |
| NOT_FOUND（会话不存在） | el-message / toast | danger | 续聊深链失效场景，统一文案「会话不存在」 |
| thinking（v2 预览） | 思考块 | secondary | fill_page 底 + brain 图标 secondary + 标题/摘要 secondary 12px |
| tool_call 已完成（v2） | 工具块 | primary / secondary | wrench 图标 primary + 工具名 text_primary + primary_light 状态标签 + 入参/出参代码框 |
| tool_call 运行中（v2） | 工具块 | primary / text_primary | loader 图标 primary + 工具名 + 「调用中…」text_primary |
| tool_call 错误（v2） | 工具块 | danger | wrench 图标 primary + 工具名 + tag_danger_bg 状态「调用失败」danger 文字 |

模块附注：本模块**无持续状态 tag**（无启用/停用类领域状态），基线 §3 规则 2 的「瞬时进行态 primary」在流式体验中的落位 = 流式期间输入控件禁用态 + 用户气泡 primary 底 + 操作底行「生成中」success 标记。

## 3. 布局实例化（基线 §4 → 本模块）

### 3.1 对话页五态（ia.md §3 → P1/P2/P3/P4/P5）

- **P1 空态**：居中欢迎区（💬 图标 44px primary + 主文案 22px 600 + 副文案 14px secondary）+ 吸底输入区；输入框固定高度 120，圆角 8，border_color 描边，内分上下两区（文本区 + 操作底行）；底行左侧 = 上传按钮（paperclip 14px + 文字 12px，border 描边）+ 模型选择下拉（sparkles primary + 模型名 + chevron-down）；底行右侧 = 字数统计 `0 / 8000`（secondary 12px）+ 发送按钮（64×32，primary 填充白字，空白时 opacity 0.5）；输入框下方居中提示「输入 @ 可提及成员 · 输入 / 可唤起指令」（secondary 12px）。
- **P2 进行中**：消息流（纵向滚动、自动滚底、增量渲染 + 流式光标 ◌）+ 吸底输入区（整体 opacity 0.5 禁用，「回复生成中…」占位 + 停止按钮 64×32 blank 底 border_color 描边）。
- **P3 错误/中断**：错误 = 助手位内联错误卡（tag_danger_bg + danger 边框 + circle-alert 图标 + 错误文案含错误码 + 「重试」按钮 danger 描边 rotate-ccw 图标，基线 §3 规则 4/5 的非 toast 例外）；中断 = 内容保留 + 操作底行 info 灰点 + 「已中断」secondary + 时间戳（无 token 统计）。
- **P4 @提及弹层**：输入 `@` 触发，absolute 定位输入框上方左对齐，宽 420px，blank 填充 + border_color 描边 + 圆角 8 + 阴影（offset 0,4 blur 16 #0000001A）；弹层头（「提及成员」+ 过滤词）+ 分组成员列表（头像 24×24 圆形 + 姓名 + 职务，选中行 active_bg + check 图标）+ 弹层脚（快捷键提示）。
- **P5 /指令弹层**：输入 `/` 触发，结构与 P4 同（同尺寸、同阴影、同头脚）；指令列表 = 图标底 24×24 primary_light 圆角 6 + 指令名 + 描述，选中行 active_bg + check 图标。
- **P6 思考与工具调用（⚠️ v2 预览）**：助手消息在信息头与正文之间新增**过程区**（纵向 gap 8 堆叠），内含思考块和/或工具调用块。v1 百炼 Agent 不产生此区域。

### 3.2 消息范式（模块专属）

- **用户气泡**：右对齐，primary 填充白字，圆角 [12,4,4,12]（左上大右侧小），padding [10,14]，lineHeight 1.6，fontSize 14。
- **助手消息**（无气泡外壳，平铺三段）：
  1. **信息头**：AI 头像（26×26，primary 填充，圆角 6，内 sparkles 14px 白字）+ 「AI 助手」（13px 600 text_primary）+ 模型标签（primary_light 底，el_primary 文字 11px，圆角 4，padding [2,8]）
  2. **正文**：textGrowth fixed-width，fill_container，lineHeight 1.6，fontSize 14，text_primary；流式尾部光标 ◌
  3. **操作底行**：左右 space_between —— 左侧消息操作（复制 copy / 重新生成 refresh-cw / 赞 thumbs-up / 踩 thumbs-down，28×28 圆角 4，secondary 色 lucide 图标 14px）；右侧统计（状态标记 + 时间戳 12px secondary + token 统计 12px secondary）
- 历史回看复用同一消息范式（chat-conversation P4，见该模块 ui-spec §3.2）。

#### 3.2.1 过程块范式（v2 预览，P6）

过程区位于助手信息头与正文之间，vertical layout gap 8，fill_container 宽。

- **思考块**：fill_page 填充，圆角 6，padding [8,12] ——
  - 思考头：horizontal gap 6，alignItems center —— brain 图标（lucide 14px secondary）+ 标题（12px 600 secondary，如「已深度思考（用时 4 秒）」）+ chevron-down（12px secondary）
  - 思考摘要：12px secondary，lineHeight 1.5，fill_container（默认显示）
  - 展开后：完整思考内容（v2 确认交互细节）
- **工具调用块**：fill_page 填充 + border_light 描边 1px，圆角 6，padding [8,12] ——
  - 工具头：horizontal gap 6，alignItems center —— wrench 图标（lucide 14px primary）+ 工具名（12px 600 text_primary）+ 状态标签 + chevron-right（12px secondary）
  - **状态标签 - 已完成**：primary_light 底，el_primary 文字 11px，圆角 4，padding [1,8]（如「已完成 · 1.2s」）
  - **状态标签 - 运行中**：无背景，text_primary 文字 11px（如「调用中…」）；图标换为 loader（lucide 14px primary，建议旋转动画）
  - **状态标签 - 错误**：tag_danger_bg 底，danger 文字 11px，圆角 4，padding [1,8]（如「调用失败」）
  - **工具 IO 区**（展开后，vertical gap 6，padding [6,0,0,0]）——
    - 入参行：horizontal gap 6，alignItems center —— 「入参」标签（11px secondary）+ 代码框（fill_blank 底 + border_light 描边 1px + 圆角 4 + padding [3,8]，内容 11px text_regular，等宽字体展示 JSON）
    - 出参行：同入参行结构，标签为「出参」
    - 错误行（仅错误态）：同结构，标签为「错误」，代码框内为错误信息

### 3.3 输入区范式（模块专属）

- **输入框容器**：fill_container 宽，固定高度 120px，fill_blank，圆角 8，border_color 描边 1px，vertical layout，padding 12，justifyContent space_between
- **底行**：fill_container 宽，space_between，alignItems center ——
  - 左侧：上传按钮（paperclip 14px + 「上传」12px，border 描边，gap 4，padding [4,8]，圆角 4）+ 模型选择（sparkles 14px primary + 模型名 12px + chevron-down 12px secondary，border 描边，gap 4，padding [4,8]，圆角 4），两者 gap 8
  - 右侧：字数统计 `n / 8000`（secondary 12px）+ 发送/停止按钮（64×32，圆角 4），gap 12
- **发送按钮**：primary 填充 + 白字 14px；空白时 opacity 0.5 禁用
- **停止按钮**：blank 底 + border_color 描边 + regular 文字 14px「停止」
- **禁用态**：整体 opacity 0.5
- **提示文案**（输入框下方）：「输入 @ 可提及成员 · 输入 / 可唤起指令」，secondary 12px，居中

### 3.4 提及/指令弹层范式（模块专属）

- **容器**：宽 420px，fill_blank，border_color 描边 1px，圆角 8，阴影 outer offset [0,4] blur 16 color #0000001A，vertical layout
- **弹层头**：padding [10,12]，space_between —— 左侧标题 13px 600 text_primary + 右侧过滤词 12px text_secondary
- **分割线**：border_light，fill_container 宽，高 1px（头尾各一）
- **列表区**：vertical gap 2，padding 6 ——
  - 分组标题：padding [6,12,2,12]，text_secondary 11px
  - 列表项：fill_container，圆角 6，gap 10，padding [8,12]，alignItems center；选中态 el_active_bg + 右侧 check 图标 14px el_primary
  - @成员项：圆形头像 24×24 primary 填充（首字白字 12px）+ 姓名 13px（选中 600）+ 职务 secondary 11px
  - /指令项：图标底 24×24 primary_light 圆角 6（内 lucide 图标 14px primary）+ 指令名 13px（选中 600）+ 描述 secondary 11px
- **弹层脚**：padding [8,12]，快捷键提示「↑↓ 选择 · Enter 确认 · Esc 关闭」secondary 11px

### 3.5 反馈（基线 §4.4 → 本模块 toast 文案）

- 入参空白：「请输入内容后再发送」（danger）
- 续聊深链失效：「会话不存在」（danger，NOT_FOUND 统一句式）
- 流式错误走内联卡不走 toast（见 §3.1 P3），toast 仅承接请求级/入参级错误

## 4. 组件状态清单（供走查）

| 组件 | 需覆盖状态 |
|---|---|
| 输入框 | 空态（发送禁用）/ 可发送 / 流式中禁用（opacity 0.5，占位「回复生成中…」）/ 错误/中断后恢复 |
| 发送按钮 | 可用（primary 填充）/ 禁用（opacity 0.5） |
| 停止按钮 | 流式中可见（blank 底 + border_color 描边 + regular 文字） |
| 上传按钮 | 可用（border 描边）/ 流式中禁用（随输入区 opacity 0.5） |
| 模型选择下拉 | 可用 / 流式中禁用 |
| 字数统计 | `0 / 8000` ~ `n / 8000`，超限变 danger |
| 提示文案 | 始终显示（secondary 12px） |
| 助手信息头 | 始终显示（头像 + 名称 + 模型标签） |
| 助手正文 | 流式渲染（光标 ◌）/ done（静态）/ interrupted（静态，无光标） |
| 消息操作按钮 | 复制 / 重新生成 / 赞 / 踩（28×28 secondary 图标） |
| 操作底行-统计-生成中 | success 绿点 + 「生成中」success 文字（仅流式） |
| 操作底行-统计-done | token 汇总 secondary + 时间戳 secondary（「生成中」消失） |
| 操作底行-统计-中断 | info 灰点 + 「已中断」secondary + 时间戳（无 token 统计） |
| 内联错误卡 | tag_danger_bg + danger 边框 + 错误文案含错误码 + 「重试」按钮（danger 描边） |
| 用户气泡 | primary 填充白字，圆角 [12,4,4,12] |
| @提及弹层 | 关闭 / 打开（@触发）/ 过滤中 / 选中 |
| @提及 chip | 输入框内 primary_light 底 + el_primary 文字 |
| /指令弹层 | 关闭 / 打开（/触发）/ 过滤中 / 选中 |
| 思考块（v2） | 折叠（摘要+耗时）/ 展开（完整内容） |
| 工具调用块-已完成（v2） | 折叠（工具名+状态）/ 展开（+入参/出参 JSON） |
| 工具调用块-运行中（v2） | 工具名+「调用中…」+ loader 图标 |
| 工具调用块-错误（v2） | 工具名+「调用失败」danger + 展开（+错误信息） |
| 消息流 | 空态（欢迎区）/ 有消息 / 续聊深链回填历史 |
| toast | 入参错误 / NOT_FOUND |

## 5. 治理

继承基线 `designs/ui-baseline.md` §5（继承规则、变更规则、偏离显式登记、文档优先于原型）；本模块当前无偏离项（见 §1 布局适用范围登记）。

### r1 → r2 变更记录（2026-08-25）

- 新增 P4 @提及成员弹层、P5 /快捷指令弹层
- 助手消息从「气泡」改为「信息头 + 正文 + 操作底行」三段平铺范式
- 操作底行新增：复制 / 重新生成 / 赞 / 踩 按钮组
- 操作底行统计区：流式态增加 success「生成中」标记，中断态增加 info「已中断」标记（无 token 统计）
- 输入框底行增加上传按钮、模型选择下拉、字数统计
- 输入框下方增加 @ / 提示文案
- 错误卡增加错误码展示 + 「重试」按钮
- 停止按钮改为 blank 底描边样式
- 用户气泡圆角从 8 改为 [12,4,4,12]

### r2 → r3 变更记录（2026-08-25）

- 新增 P6 思考与工具调用 v2 预览画板（六画板 P1~P6）
- §2 状态色映射新增 thinking / tool_call 已完成 / 运行中 / 错误四行（v2 预览）
- §3.1 页面清单新增 P6 条目
- §3.2.1 新增过程块范式：思考块（brain + 摘要 + 可展开）、工具调用块（wrench + 三种状态标签 + IO 代码框）
- §4 组件状态清单新增思考块、工具调用块（已完成/运行中/错误）四行
