# Options 项目交接说明（For Codex）

> 本文档整理自与 Claude Code 的开发会话（2026-07-06 ~ 2026-07-15），目的是把只存在对话记忆里的决策、踩坑记录和未提交文件补全成书面材料，供后续在 Codex 里维护此项目时使用。写作时间：2026-07-15。

---

## 0. 项目速览

- **目标**：把用户原有的"每周期权到期日流分析"从手动截图流程，改造成可直连 Barchart.com 抓数据、自动生成 HTML 报告的半自动化流程。
- **交易范围**：用户周五盘前扫描，当周到期期权，实际日内只交易 2 个标的。
- **当前阶段**：2026-07-10（周五）起进入 1 个月实盘测试期，用于验证框架修订后能否把胜率从 <40% 提升。
- **交易记录**：用户使用自己的私有外部系统记录实际交易结果（具体地址不公开，未在本仓库留存，需要时向用户本人索取），**不在本项目范围内**，本仓库之前生成的 CSV 交易记录模板已被放弃使用（详见第 2、7 节）。
- **仓库**：`https://github.com/kaguravicky-web/options.git`（公开仓库，本地路径 `C:\Users\kaguravicky\Downloads\options-flow-framework`，分支 `main`）。
- **关联但独立的更大项目**：`D:\交易为生\` 是用户的"盘前报告生产线"主项目（有自己的 `CONTEXT.md`、`scripts/`、`templates/`、`data/` 等），本 options 项目目前只是把生成的 HTML 报告复制一份到 `D:\交易为生\output\{日期}-options-flow.html`，**没有**接入该项目的 `scripts/`/`templates/` 自动化体系，是两条平行的产出线。

---

## 1. 当前完整工作流

1. **选标的**：用户口头给出 ticker 列表（周五盘前扫描全部关注标的，日内实际只交易 2 个）。当前会话中曾用 `NVDA`、`TSLA` 作为随意挑选的演示标的（用户原话"你随便用"），**不是真实持仓**。
2. **判断资产类型**：个股用 `/stocks/quotes/{TICKER}/...` 路径，ETF 用 `/etfs-funds/quotes/{TICKER}/...`（前缀不同，需要提前知道 ticker 是个股还是 ETF）。
3. **逐个标的抓 3 个 Barchart 页面**（详见第 3 节 URL 表），每页流程固定为：
   - `navigate` 到目标 URL（**必须带 `?expiration=YYYY-MM-DD-w` 锁定周五到期日**，见第 3、6、7 节的重大踩坑）
   - `computer` 的 `wait` 动作等待 3 秒（等页面 JS 渲染完成，未等待会拿到空数据或旧数据）
   - `get_page_text` 提取纯文本，从固定的英文句式里正则/人工解析出数值（见第 3 节）
4. **人工/对话中做判断**：根据第 4 节的规则，结合 Gamma Flip Point、Max Pain 偏离、P/C Ratio、IV Rank 得出"本周偏向"文字结论（**当前没有代码把这一步自动化，是 Claude 在对话里手写判断**）。
5. **生成 HTML 报告**：Claude 手写一份完整的自包含 HTML（内嵌 CSS，无外部依赖），当前**没有独立的模板引擎或脚本**，每次都是重新手写 HTML 字符串（见第 2 节）。
6. **存档**：报告存两份——
   - `D:\交易为生\output\{日期}-options-flow.html`（跟主项目 `premarket`/`postmarket` 目录的命名规则对齐：`{日期}-xxx.html`）
   - `Downloads\options-flow-framework\reports\{日期}-options-flow.html`（git 仓库内）
7. **git 提交并推送**：在 `options-flow-framework` 目录下 `git add` + `git commit` + `git push`，远程为 `origin`（`https://github.com/kaguravicky-web/options.git`），分支 `main`。

---

## 2. 实际使用的提示词/命令/脚本，以及未提交文件

### 没有独立脚本文件
**这是最重要的一点**：整个抓取+生成+发布流程目前**完全没有落地成可重复运行的脚本**（没有 `.py`/`.js`/`.ps1` 文件）。所有步骤都是 Claude 在对话中通过 MCP 工具调用逐步手动执行的，包括：
- 浏览器导航/等待/读取文本（见第 8 节）
- HTML 报告是 Claude 每次手写的 HTML 字符串（不是模板+数据填充）
- git 命令通过 Bash 工具逐条手动执行

**这是 Codex 接手后最值得优先做的事**：把第 1 节的工作流写成真正的脚本（例如 Playwright/Puppeteer 脚本抓取 Barchart + 一个 HTML 模板 + Python/Node 填数据），摆脱"每次都要手动对话驱动"的现状。

### 实际执行过的 git 命令（供参考）
```bash
# 仓库初始化（在 options-flow-framework 目录下）
git init -q
git add README.md 交易记录表模板.csv
git -c user.name="kaguravicky" -c user.email="kaguravicky@gmail.com" commit -q -m "..."
git remote add origin https://github.com/kaguravicky-web/options.git
git branch -M main
git push -u origin main
```
注意：commit 时用 `-c user.name=... -c user.email=...` **临时覆盖**身份，没有改全局 git config。`git push` 当时**没有弹出任何登录提示**，说明这台 Windows 机器的 Git 凭据管理器里已经缓存了该 GitHub 账号的授权——**这个免密推送能力是本机环境特有的，换一台机器或换 Codex 的运行环境大概率需要重新认证**（token 或 OAuth）。

### 未提交到 git 的本地文件
| 路径 | 说明 | 与仓库内文件的关系 |
|---|---|---|
| `C:\Users\kaguravicky\Downloads\期权流分析框架_模板.md` | 用户最初上传的框架源文件，Claude 后续在这份文件上做修订 | 内容几乎等同于仓库 `README.md`，但**存在 1 处漂移**：README.md 多了一段"每日报告"指向 `reports/` 目录的说明，这行没有回填到 Downloads 里的原文件 |
| `C:\Users\kaguravicky\Downloads\期权流交易记录表.csv` | 最初给用户设计的交易记录表 | 内容与仓库内 `交易记录表模板.csv` **完全一致**（已核对无 diff），用户已表示不再使用（用自己的系统记录交易），此文件可视为废弃 |
| `D:\交易为生\output\2026-07-06-options-flow.html` | 演示报告的"生产线"副本 | 内容与仓库 `reports/2026-07-06-options-flow.html` **完全一致**（同一份文件复制过去的） |

---

## 3. Barchart 页面 URL、字段提取方法、限流/登录/动态表格处理

### 三个核心页面

| 数据 | URL 模式 | 是否需要 `?expiration=` | 提取到的文本片段示例 |
|---|---|---|---|
| Max Pain | `/stocks/quotes/{TICKER}/max-pain-chart`（ETF 用 `/etfs-funds/quotes/{TICKER}/max-pain-chart`） | **必须**，否则默认显示最近到期日（可能是当天 0DTE） | `"Max Pain: 745.00"`，标题里会带到期日如 `"SPY - Max Pain - 07/06/26 (0 DTE)"` |
| Gamma Exposure | `/stocks/quotes/{TICKER}/gamma-exposure`（ETF 同理） | **必须**，否则默认聚合未来 4 个到期日 | `"{TICKER} gamma flip point is 744.85 where..."`、`"{TICKER} put wall is 750.00. {TICKER} call wall is 750.00."` |
| Put/Call Ratio | `/stocks/quotes/{TICKER}/put-call-ratios`（ETF 同理） | 不需要（按当日成交/累计持仓统计，不区分到期日） | `Put Volume Total` / `Call Volume Total` / `Put/Call Volume Ratio` / `Put Open Interest Total` / `Call Open Interest Total` / `Put/Call Open Interest Ratio`，以及 `Implied Volatility:` / `Historic Volatility:` / `IV Rank:` / `IV Percentile:` |

`?expiration=` 的值格式是 `YYYY-MM-DD-w`（如周五到期是 2026-07-10，就是 `?expiration=2026-07-10-w`），这个格式是从 Volatility & Greeks 页面的到期日下拉菜单实际跳转 URL 里观察到的（`?expiration=2026-07-06-w`），未在 Barchart 官方文档上验证过，只是实测有效。

### 提取方法
用 Claude in Chrome 插件（`get_page_text`）读取渲染后页面的纯文本（不是截图，不是 HTML 源码），上面这些字段都以英文自然语句形式出现在正文里，靠字符串匹配/人工阅读定位数值。

### 登录要求
- 测试全程用户的 Chrome 是**已登录**一个免费 "My Barchart" 账号的状态（页面上有 "My Account" 入口），未测试过**未登录/无痕模式**下 `get_page_text` 能否读到同样的数据——**这是一个未验证的假设，Codex 如果换一个干净的浏览器环境需要重新验证是否需要登录**。

### 限流（免费账号每日 20 次页面浏览）
- 页面顶部固定出现横幅："Enjoying your free My Barchart? You've used X of 20 views today."
- **每次 `navigate`（包括 404 页面）都会计数 +1**，实测猜错 URL 导致 404 也照样扣额度，浪费了 3 次额度在猜 Max Pain 页面路径上。
- 一个标的完整抓 3 个页面 = 3 次额度，即约 **6-7 个标的/天**是免费账号的硬上限。
- 曾尝试用 `find` 工具在已加载页面里定位链接 `href`（不产生新的页面浏览），再直接 `navigate` 到正确 URL，以避免继续猜错 URL 浪费额度——这个方法本身**成功**（找到了 ETF 路径 `/etfs-funds/quotes/SPY/max-pain-chart` 的正确写法）。
- 曾尝试点击页面内链接（`computer` 的 `left_click` 配合 `ref`）而不是 `navigate`，猜测这样走客户端路由是否不计入配额——**实测点击没有生效（页面没有跳转，配额也没变化），这个假设没有被验证，只是一次失败的尝试，不代表结论成立**。

### 动态/复杂表格读不到
- `Options Prices` / `Volatility & Greeks` 页面上逐行权价的期权链明细表（strike-by-strike 的 OI、Greeks）**无法**通过 `get_page_text` 读到——页面只返回了页头信息（IV/HV/到期日列表），表格本身的行数据是空的，大概率是虚拟滚动/复杂网格渲染，纯文本提取抓不到。**如果未来需要逐行权价明细，需要换方法**（截图 + 视觉解析，或者找 Barchart 有没有可访问的 JSON API 端点）。

### 反爬/CAPTCHA
全程没有遇到验证码或明显的反爬拦截，唯一的"墙"就是上面说的每日 20 次浏览量限制。

### maxpain.com 已失效
`https://www.maxpain.com` 用 `WebFetch` 请求会 307 重定向到 `afternic.com` 的域名待售页面，说明这个域名目前是停放状态、没有真实数据，已从数据源里移除。

---

## 4. 报告中每个指标/结论的计算规则

### Gamma Flip Point（主判据）
不是自己算的，是 Barchart 页面直接给出的计算结果。判断规则：

| 条件 | 标记 | 含义 |
|---|---|---|
| 当前价 > Gamma Flip | 正Gamma 🔴 | 做市商净多 Gamma，对冲行为"追涨杀跌反着来"→ 压制波动 |
| 当前价 < Gamma Flip | 负Gamma 🟢 | 做市商净空 Gamma，对冲行为顺势加码 → **双向**放大波动（不是只助涨） |
| 当前价 ≈ Flip（±1%内） | 中性/过渡区 | 环境不稳定 |

旧版规则（已废弃）：用 `Call OI >> Put OI` 简化推断正负 Gamma，2026-07-06 实测发现这个代理指标跟真实 Flip Point 方向可能相反（NVDA/TSLA 当天都是这种情况），现已降级为仅供参考的情绪指标，不再是主判据。

### Call Wall / Put Wall
Barchart 直接给出："基于所有合约里 Gamma 读数最高的行权价"。用作阻力/支撑的参考强度，**不是绝对不可突破的硬阻力**——大成交量/大新闻可以直接穿越。

### Max Pain 偏离
`偏离% = (当前价 - Max Pain) / Max Pain`，正值表示价格在 Max Pain 之上（到期前有向下拉回的理论引力，DTE 越小引力越强）。

### P/C Ratio
两个变体：Volume Ratio（当日成交量比，反映当天资金动向）、OI Ratio（累计持仓比，反映仓位结构）。Barchart 页面原文的经验法则："低于 0.7 通常偏多头情绪，高于 1.0 通常偏空头情绪"，本框架把它当作**情绪参考**，不单独触发交易。

### IV Rank
Barchart 直接给出（相对过去一年的隐含波动率百分位）。
- `>50%`：期权贵，慎买裸方向性期权，考虑价差策略
- `<30%`：期权便宜，更适合直接买方向性期权
- `30%-50%`：中性

### "本周偏向"结论
⚠️ **目前没有公式化的加权评分算法**，是 Claude 在对话里综合 Gamma 环境 + Max Pain 偏离 + 检查表结果后**人工写出的定性判断**（多/空/中性 + 一句话理由）。如果 Codex 要把这一步自动化，需要重新设计一个明确的评分/决策规则，当前框架文件里只有各项独立指标的判断规则，没有汇总成单一分数的逻辑。

### 警报等级
| 警报 | 触发条件 | 等级 |
|---|---|---|
| Gamma 环境反转 | 价格反向穿越 Gamma Flip Point | 🔴 |
| 跌破 Put Wall | — | 🔴 |
| Put OI 异常放大 | — | 🟡 |
| 价格远高于 Max Pain | 越接近到期日权重越高 | 🟡 |
| P/C Ratio 从低位骤升 | — | 🟡 |

自动防守标记的伪代码规则（写在框架文件里，**没有实际代码实现**，只是文字规则）：
```
IF 持仓价格距 Put Wall ≤ 5% THEN 输出警告
IF 价格反向穿越 Gamma Flip Point THEN 输出结构性止损信号
```

---

## 5. "九星连珠得分"——正式口径

**用户已于 2026-07-15 确认："九星连珠"是夸张性的历史名称，正式规则就是 5 项检查，按五分制计分，不扩展为 9 项。**

- 实际检查项固定为 **5 条**：
  1. 当前价格与 Gamma Flip Point 的相对位置是否与计划方向一致
  2. 股价逼近/突破 Call Wall
  3. 有技术 setup（EMA9/21 / HMA）
  4. 有基本面/叙事催化剂
  5. CTA 尚未提前入场
- 判定标准是"满足 4/5 → 高度关注，满足 5/5 → 重仓候机"，用的是**五分制**。
- 交易记录表模板（CSV）里的字段名是"九星连珠得分"，示例值写的是 `3/5`——同样是**五分制**，不是九分制。
- "九星连珠"仅作为名称保留，不代表检查项数量。所有界面、报告、数据字段和统计逻辑均应使用 `/5`，不得解释为九分制。

---

## 6. 示例报告（NVDA、TSLA，2026-07-06）数据来源说明

⚠️ **这份演示报告存在已知数据缺陷，不能当作真实分析参考，只能当作 HTML 格式范例。**

- **采集日期**：2026-07-06（周一），盘中实时抓取（不是收盘后）。
- **标的选择**：用户原话"你随便用"，Claude 随意挑选 NVDA、TSLA 作为演示，**不是用户真实持仓/关注标的**。
- **到期日缺陷**：抓取这份数据时，**还没有发现需要加 `?expiration=` 参数锁定到期日的问题**（这个问题是在生成完这份报告之后才发现并写进框架文件的）。也就是说：
  - Max Pain 页面默认显示的是**最近到期日**（页面标题里写的是 `"NVDA - Max Pain - 07/06/26 (0 DTE)"`，即当天到期，不是框架设定的"本周五"）
  - Gamma Exposure 页面默认是"未来 4 个到期日"的**聚合值**，同样不是单独的周五数据
  - 因此报告里 NVDA Max Pain=195.00、Gamma Flip=191.62，TSLA Max Pain=400.00、Gamma Flip=389.08 这些数字，**大概率对应的是周一 0DTE（或多档聚合），不是周五到期的真实数据**
- **价格漂移**：由于 3 个页面是依次单独抓取的（不是同一时刻的原子快照），同一标的在不同页面上看到的"当前价"存在几毛钱的漂移（如 NVDA 在 Max Pain 页读到 196.47，在 Gamma Exposure 页读到 195.91，在 P/C Ratio 页读到 195.92），是市场实时跳动导致，不是数据错误。
- **采集时间**：NVDA 三个页面约在 2026-07-06 11:01–11:02 ET 之间完成；TSLA 约在 10:47–10:57 ET 之间完成。

**结论**：如果要把这份报告当作"框架效果"的参照，必须先补一次**带 `?expiration=2026-07-10-w` 参数**的重新抓取，目前还没有做过这次复测。

---

## 7. 已知问题 / 未完成事项 / 踩坑记录 / 原计划的下一步

### 已知问题
1. 演示报告（NVDA/TSLA）数据到期日不准确，见第 6 节，未重新验证过修复后的效果。
2. Options Prices / Volatility & Greeks 页面的逐行权价明细表读不到（见第 3 节）。
3. 未验证 Barchart 免费账号在未登录状态下 `get_page_text` 是否还能读到同样的数据。
4. "点击链接是否不计入每日浏览额度"的假设没有验证成功（尝试失败，非结论）。
5. `Downloads\期权流分析框架_模板.md` 与仓库 `README.md` 之间有 1 行漂移（见第 2 节表格），未回填同步。
6. "九星连珠"已确认是夸张性的历史名称；正式规则固定为 5 项、按 `/5` 计分（见第 5 节）。
7. "本周偏向"结论目前是人工定性判断，没有代码化的评分公式（见第 4 节）。

### 未完成事项 / 原计划的下一步
1. **把整个抓取+生成流程写成真正的脚本**（目前完全靠对话手动驱动，见第 2 节）——这是最优先的技术债。
2. 周五（2026-07-10）开始实盘测试，计划累积 20-30 笔真实交易后做相关性分析，判断框架里哪个字段真的跟胜负挂钩——**这个分析目前完全没有开始，需要先等用户在自己的外部交易记录系统里积累数据，再由 Codex/Claude 读取那份数据做统计**（注意：那是用户自己的私有网站，本项目未曾访问过，具体数据结构未知）。
3. 用修复后的 `?expiration=` 参数重新抓一次 NVDA/TSLA（或用户实际关注的标的）做验证性对比。
4. 考虑是否需要升级 Barchart Premier 账号来解除每日 20 次浏览的限制（如果周五实际扫描标的数经常超过 6-7 个）。

---

## 8. Chrome 插件读取网页时的具体操作和"提示词"

用的是 **Claude Code 内置的 Chrome 浏览器扩展 MCP**（工具名前缀 `mcp__claude-in-chrome__*`，本次会话中途 MCP 服务器曾断开重连，工具名一度显示为 `mcp__Claude_in_Chrome__*`，是同一个扩展，只是命名大小写/下划线格式因重连而变化——**这是 Claude Code 环境特有的能力，如果 Codex 运行在没有这个浏览器扩展桥接的环境里，需要用等价方案（例如 Playwright 脚本）重新实现，不能假设有同名工具可用**）。

### 实际操作序列（每个 Barchart 页面重复这一套）
1. `list_connected_browsers` —— 确认本机 Chrome 扩展已连接，拿到 `deviceId`
2. `select_browser`（传入 `deviceId`）—— 绑定这个浏览器会话
3. `navigate`（传入目标 URL + `tabId`）
4. 用 `browser_batch` 把下面两步合并成一次调用（减少往返）：
   - `computer` 动作 `wait`，`duration: 3`（秒），等待页面 JS 渲染完成——**这个等待是必须的，太快读取会拿到空壳页面**
   - `get_page_text`（传入 `tabId`）—— 返回渲染后的纯文本正文

### 用于定位真实 URL 的辅助操作（非每次都需要）
当直接猜 URL 导致 404（浪费每日额度）时，改用：
- `find`（自然语言查询，例如实际用过的 query："Max Pain & Vol Skew navigation link"）—— 在已加载页面的可访问性树里模糊匹配元素，返回 `ref_N`
- `read_page`（传入该 `ref_id`）—— 读出这个元素的真实 `href`，据此拼出正确 URL 再 `navigate`，避免继续瞎猜浪费配额

### 没有对 Barchart 本身"发提示词"
Barchart 是普通网站，不是 AI 服务，`get_page_text` 没有 prompt 参数，就是纯文本提取，没有"提示词"这个概念。唯一带 prompt 参数的调用是曾经尝试过的 `WebFetch`（例如对 `maxpain.com` 用过的提取指令："提取这个页面中的Max Pain价格、Put Wall、Call Wall、Put/Call Ratio、各行权价的Open Interest数据"），但那次因为域名已失效（重定向到域名出售页）没有拿到真实数据——**WebFetch 这条路径在本项目里从未真正成功过，实际生效的方法是上面的浏览器扩展 `get_page_text`**。

---

## 9. GitHub 仓库之外的相关记忆、对话、文件、截图、本地脚本

### Claude 的持久记忆文件（Codex 默认看不到，是 Claude Code 专属的记忆系统）
路径：`C:\Users\kaguravicky\.claude\projects\C--Users-kaguravicky-Downloads\memory\`
- `project_options_flow_framework.md` —— 本项目的开发状态、数据源方案、已知问题、GitHub 仓库信息（本文档很大程度上是把这份记忆和相关对话内容摊开写成书面材料）
- `user_options_novice.md` —— 关于用户期权背景、沟通偏好的记忆
- `MEMORY.md` —— 上面两份记忆的索引
- **这些文件是 Claude Code 特有的持久记忆机制，不在 git 仓库里，Codex 大概率无法直接访问**，本 HANDOFF.md 就是为了把其中值得长期保留的内容转成 Codex 能读到的书面文档（对应用户需求第 10 条）。

### 本地未提交文件
见第 2 节表格：`Downloads\期权流分析框架_模板.md`、`Downloads\期权流交易记录表.csv`、`D:\交易为生\output\2026-07-06-options-flow.html`。

### 截图
**本项目全程没有产生/保存任何截图**。数据抓取从一开始尝试截图（用户反馈"太麻烦"而发起本项目），到后来全部改用 `get_page_text` 纯文本提取，过程中没有留存任何截图文件。

### 本地脚本
**不存在**。见第 2 节，全流程没有任何 `.py`/`.js`/`.ps1` 等脚本文件，都是 Claude 对话中直接调用工具完成的。

### 更大的关联项目（超出本仓库范围，仅供参考）
`D:\交易为生\` 目录下还有 `CONTEXT.md`、`scripts/`、`templates/`、`data/`、`deepvue/`、`moomoo/`、`reference/` 等，是用户更大的"盘前报告生产线"项目（详见 Claude 记忆 `project_premarket_report_system.md`，同样不在本仓库里）。本 options 项目**只是把生成的 HTML 报告文件复制一份进 `D:\交易为生\output\` 目录**，两个项目目前是并行、非集成关系。

### 用户自己的外部交易记录系统
用户自建/自用的私有交易记录网站（**具体地址不写入本公开仓库**，需要时向用户本人索取），本项目从未访问或读取过这个系统的数据，具体数据结构、认证方式均未知。

---

*本文档由 Claude Code 于 2026-07-15 整理生成，写在仓库根目录 `HANDOFF.md`，会随下一次 commit 一并推送到 GitHub。*
