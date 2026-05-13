# stocker — A 股 / 黄金 / 港股 / 美股分析 Claude Skills 仓库

## 项目定位

保存一组在 **Claude Code / Cursor CLI** 中运行的行情分析 skills(A 股 / 黄金 / 港股 / 美股),通过 WebFetch / WebSearch 自行抓取公开行情数据,产出深度复盘报告。

**当前范围**:A 股 + 黄金(国内视角:上海金 Au99.99 / Au(T+D))+ 港股(内地视角:恒指 / 恒科 / 国企 + 南向资金 + AH 溢价 + 中概 ADR 联动)+ 美股(内地视角:三大指数 + 纳 100 + 罗素 + 费半 + Magnificent 7 + 中概 ADR + DXY / 美债 / VIX 利率汇率情绪驱动 + 财报)。其他市场(原油、外汇本身、加密货币等)**不做**(若需扩展请先与用户确认)。

**面向场景**:
- A 股:工作日收盘后由 launchd 17:00 自动跑复盘
- 黄金:SGE 日盘 16:00 后手动跑复盘(暂未配定时)
- 港股:港股收盘 16:00 + CAS 后手动跑复盘,**最佳窗口 17:00-18:00**(等南向资金日终披露,暂未配定时)
- 美股:**次日北京时间早上 7:00-9:00 手动跑昨夜美股**(夏令时 7:00 后,冬令时 8:00 后),暂未配定时

## 当前 Skill 列表

| Skill | 用途 | 关键文件 |
|---|---|---|
| `a-stock-closing-review` | A 股每日收盘复盘 | `.cursor/skills/a-stock-closing-review/SKILL.md` + `data-collection.md` |
| `gold-daily-review` | 黄金每日复盘(国内视角:上海金 Au99.99 / Au(T+D)) | `.cursor/skills/gold-daily-review/SKILL.md` + `data-collection.md` |
| `hk-stock-review` | 港股每日收盘复盘(内地视角:恒指 / 恒科 / 国企 + 南向 + AH 溢价 + ADR 联动) | `.cursor/skills/hk-stock-review/SKILL.md` + `data-collection.md` |
| `us-stock-review` | 美股每日收盘复盘(内地视角:三大指数 + M7 + 中概 + DXY / 美债 / VIX + 财报) | `.cursor/skills/us-stock-review/SKILL.md` + `data-collection.md` |

> Cursor CLI 从 `.cursor/skills/` 自动发现 skill;Claude Code 兼容方式见下方说明。

触发词(用户在 Claude Code / Cursor CLI 里说就会调起):
- A 股:"今日 A 股复盘""今天大盘怎么样""A 股收盘总结""今日板块轮动""盘后分析"
- 黄金:"今日黄金""黄金行情""上海金复盘""沪金分析""黄金盘后""今天金价""黄金日盘复盘"
- 港股:"今日港股""港股收盘""恒生指数""恒生科技""港股盘后""港股复盘""南向资金""AH 溢价""中概股 ADR"
- 美股:"昨夜美股""美股复盘""美股收盘""纳指""标普 500""道指""七巨头""中概股""美元指数""10Y 美债""VIX""财报复盘"

## 数据采集铁律(避开最大坑)

所有 skill 抓取行情数据时,**必须遵守各 skill 自带的 `data-collection.md` 流程**(A 股见 `.cursor/skills/a-stock-closing-review/data-collection.md`,黄金见 `.cursor/skills/gold-daily-review/data-collection.md`,港股见 `.cursor/skills/hk-stock-review/data-collection.md`,美股见 `.cursor/skills/us-stock-review/data-collection.md`),核心三条共通:

### 1. Phase 0 优先

走东财 push2 系列 JSON API(返回纯 JSON,不依赖 JS 渲染、不依赖搜索引擎索引),**不要**先用 WebSearch + 文章页 —— JS 动态行情页 WebFetch 会抓到空白占位、AI 摘要会把昨日数据冒充今日。

### 2. 跨日陷阱(最危险的失败模式)

push2 实时接口跨日 / 跨时段跑会拿到 **次日盘中或夜盘数据**,且接口不报错、数据结构正常,**模型极易把错数据当对数据用**。

**每次跑前必须先判断"目标交易日 + 时段 vs 当前北京时间"**:
- A 股:当天 15:00 ~ 24:00 用实时接口;跨日 / 周末 / 节假日切 push2his 历史日线接口
- 黄金:**比 A 股更激进** —— 黄金有夜盘 20:00 ~ 次日 02:30,**20:00 之后实时接口立刻拿到夜盘最新价,不再是日盘收盘**;只有当日 15:30 ~ 19:59 的窗口能用实时接口,其他时段(含次日 02:30 后、周末、节假日)必须切历史日线接口
- 港股:当天 16:10(CAS 结束)~ 23:59 用实时接口;**额外注意港股交易日历与 A 股不同步**(圣诞、复活节、佛诞、香港回归等港股自有节假日,A 股春节比港股多休 4-6 日);跨日 / 港股自有假期 / A 股或港股任一休市 → 切 push2his 历史日线接口;**南向资金 datacenter 接口在 17:00 前调用通常返上一交易日数据**,需校验返回的 `TRADE_DATE` 字段
- 美股:**最复杂的时区映射** —— 北京时间次日早上跑时,**目标日 = 昨夜美股交易日**(不是当前北京日期);夏令时美股收盘 = 北京时间次日 4:00,冬令时 = 次日 5:00;**默认直接走 push2his 历史接口**(指定 `beg=end=YYYYMMDD`)无脑规避跨日陷阱;美股自有节假日(MLK Day / 总统日 / 阵亡将士日 / 独立日 / 劳动节 / 感恩节 / 圣诞节等)与中国完全错开;部分日子是半日市

具体接口与时段对照表见各 skill 的 `data-collection.md`。

### 3. 数据缺失处理

严禁凭记忆补数据,缺数据明确标注"未获取";档位 C(P0 缺 ≥ 3 项)时**不归档**,告知用户稍后重试。

## 报告归档

| 类型 | 归档路径 | 文件名格式 |
|------|---------|-----------|
| A 股 | `reports/a-stock/` | `YYYY-MM-DD-A股收盘复盘.md` |
| 黄金 | `reports/gold/` | `YYYY-MM-DD-黄金日盘复盘.md` |
| 港股 | `reports/hk-stock/` | `YYYY-MM-DD-港股收盘复盘.md` |
| 美股 | `reports/us-stock/` | `YYYY-MM-DD-美股收盘复盘.md` |

- **文件名日期**:用 **复盘对象的交易日**,不是当前日期(周末跑周五数据 → 文件名是周五日期;港股要用港股交易日历;**美股要用昨夜美股交易日**,即"今天-1 日"或周末/节假日跑时回退到上一个美股交易日)
- **同名覆盖**:同一交易日多次跑直接覆盖,不留多版本
- **写入工具**:用 `Write`,不要 `cat`/`echo` via Bash
- **目录不存在时**:Write 会自动创建(等价于 `mkdir -p`)

## 预测约束(输出未来走势时强制)

凡涉及"未来 1-2 日趋势"或"明日动向"的判断,**必须同时满足三条**:

1. **给依据** —— 消息面 / 资金面 / 技术面至少一条具体证据
2. **概率化语言** —— "倾向""大概率""存在 XX 可能";禁用"肯定""必然""一定会"
3. **证伪条件** —— 一句话"若 XX 出现,该判断不成立"

违反任一条,该预测作废,必须改写。这是把分析与算命区分开的关键。

## 定时任务(launchd)

- **plist**:`~/Library/LaunchAgents/com.easton.stocker.daily-review.plist`
- **触发**:工作日(周一到周五)17:00 北京时间
- **命令**:`claude -p "执行 a-stock-closing-review,做今天的 A 股收盘复盘" --dangerously-skip-permissions`
- **日志**:`stocker/launchd-stdout.log`(成功输出)/ `stocker/launchd-stderr.log`(报错)
- **claude 绝对路径**(plist 里固化的):`/Users/eastonshay/.nvm/versions/node/v20.19.4/bin/claude`

**管理命令**:

```bash
# 加载(启用定时)
launchctl load ~/Library/LaunchAgents/com.easton.stocker.daily-review.plist

# 卸载(停止定时,plist 保留)
launchctl unload ~/Library/LaunchAgents/com.easton.stocker.daily-review.plist

# 立即手动跑一次(不等 17:00,debug 用)
launchctl start com.easton.stocker.daily-review

# 看是否已加载
launchctl list | grep stocker
```

修改 plist 后必须先 `unload` 再 `load`,改动才生效。

> 黄金 / 港股 / 美股三个 skill 目前都不配定时,手动触发为主。如后续要加:
> - 黄金:建议新建独立 plist(`com.easton.stocker.gold-review.plist`),触发 SGE 日盘 16:30 后、夜盘开盘 20:00 前(避开夜盘陷阱)
> - 港股:建议触发 17:30 后(等南向资金日终披露完整),且需要先校验当日是港股交易日 + 南向通道开放
> - 美股:建议触发 **次日早上** 7:30(夏令时)/ 8:30(冬令时),需要 plist 处理夏冬令时切换(每年 2 次手动调时间)

## 红线(违反必返工)

1. **不编造行情数据** —— 缺数据就标"未获取",决不凭记忆补
2. **跨日复盘前必判断"目标日 + 时段 vs 当前日"** —— 漏判会拿错数据且模型察觉不到(黄金 20:00 后尤甚,港股 17:00 前南向数据延迟)
3. **港股要单独判断港股交易日历** —— A 股交易日 ≠ 港股交易日,且南向通道在任一边休市时关闭
4. **美股目标日 = 昨夜美股交易日,不是当前北京日期** —— 北京时间周一/周日早上跑都对应美股周五;美股自有节假日要顺延
5. **未来走势必带依据 + 概率语言 + 证伪条件** —— 三缺一即返工
6. **档位 C 的报告不准归档** —— 不污染历史 reports

## 配套文档

**A 股复盘**:
- `.cursor/skills/a-stock-closing-review/SKILL.md` —— skill 主文档(10 段输出结构、分析框架、自检清单)
- `.cursor/skills/a-stock-closing-review/data-collection.md` —— 数据采集执行手册(Phase 0/1/2、已知坑、降级规则、迭代记录)

**黄金复盘**:
- `.cursor/skills/gold-daily-review/SKILL.md` —— skill 主文档(10 段输出结构、6 因子驱动诊断、自检清单)
- `.cursor/skills/gold-daily-review/data-collection.md` —— 数据采集执行手册(夜盘陷阱起手判断、Phase 0/1/2、secid 探测协议、迭代记录)

**港股复盘**:
- `.cursor/skills/hk-stock-review/SKILL.md` —— skill 主文档(10 段输出结构、内地视角分析框架、AH 溢价 + ADR 联动专章、自检清单)
- `.cursor/skills/hk-stock-review/data-collection.md` —— 数据采集执行手册(港股交易日历起手判断、Phase 0/1/2、港股域 secid 前缀对照表、行业板块榜缺失代偿路径、迭代记录)

**美股复盘**:
- `.cursor/skills/us-stock-review/SKILL.md` —— skill 主文档(10 段输出结构、SPDR 11 ETF + M7 双视角、利率汇率 VIX 三联动驱动诊断、中概 ADR 与港股联动、盘前盘后 + 财报、自检清单)
- `.cursor/skills/us-stock-review/data-collection.md` —— 数据采集执行手册(目标日时区映射、Phase 0/1/2、美股域 secid 前缀对照表(`100/103/105/106/107/171`)、GICS 行业板块榜 + VIX 现货 + 盘前盘后 + 财报日历四类无主源 fallback 路径、迭代记录)

**通用**:
- 项目记忆:`~/.claude/projects/-Users-eastonshay-xym-life-stocker/memory/`(自动加载,记录用户偏好和项目演进决策)

## 双栈兼容说明

- **Cursor CLI**:从 `.cursor/skills/` 自动发现 skill,读取 `AGENTS.md` 作为项目指令。
- **Claude Code**:仍读取 `CLAUDE.md`(与 `AGENTS.md` 内容同步)。skill 路径 `.cursor/skills/<skill-name>/`,Claude Code 通过 SKILL.md frontmatter 触发词机制依然可以调起。
- 修改 skill 时只改 `.cursor/skills/` 下的版本即可,两端共用同一份。
