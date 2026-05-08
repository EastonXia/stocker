# stocker — A 股分析 Claude Skills 仓库

## 项目定位

保存一组在 **Claude Code / Cursor CLI** 中运行的 A 股分析 skills,通过 WebFetch / WebSearch 自行抓取公开行情数据,产出深度复盘报告。

**当前范围**:仅 A 股。港股、美股、跨市场联动**不做**(若需扩展请先与用户确认)。

**面向场景**:每天 A 股收盘后,在 Claude Code / Cursor CLI 里跑一次复盘 / 由 launchd 定时跑一次。

## 当前 Skill 列表

| Skill | 用途 | 关键文件 |
|---|---|---|
| `a-stock-closing-review` | A 股每日收盘复盘 | `.cursor/skills/a-stock-closing-review/SKILL.md` + `data-collection.md` |

> Cursor CLI 从 `.cursor/skills/` 自动发现 skill;Claude Code 兼容方式见下方说明。

触发词(用户在 Claude Code / Cursor CLI 里说就会调起):
- "今日 A 股复盘""今天大盘怎么样""A 股收盘总结""今日板块轮动""盘后分析"

## 数据采集铁律(避开最大坑)

所有 skill 抓取股市数据时,**必须遵守 `.cursor/skills/a-stock-closing-review/data-collection.md`** 的流程,核心三条:

### 1. Phase 0 优先

走东财 push2 系列 JSON API(返回纯 JSON,不依赖 JS 渲染、不依赖搜索引擎索引),**不要**先用 WebSearch + 文章页 —— JS 动态行情页 WebFetch 会抓到空白占位、AI 摘要会把昨日数据冒充今日。

### 2. 跨日陷阱(最危险的失败模式)

`0.1 / 0.2 / 0.3` push2 实时接口跨日跑会拿到 **次日盘中数据**,且接口不报错、数据结构正常,**模型极易把错数据当对数据用**。

**每次跑前必须先判断"目标交易日 vs 当前北京时间"**:
- 当天跑(目标日 15:00 ~ 24:00):用实时接口
- 跨日 / 周末 / 节假日跑:**必须切 0.6 push2his 历史日线接口**(见 `data-collection.md`)

### 3. 数据缺失处理

严禁凭记忆补数据,缺数据明确标注"未获取";档位 C(P0 缺 ≥ 3 项)时**不归档**,告知用户稍后重试。

## 报告归档

- **路径**:`/Users/eastonshay/xym/life/stocker/reports/YYYY-MM-DD-A股收盘复盘.md`
- **文件名日期**:用 **复盘对象的交易日**,不是当前日期(周末跑周五数据 → 文件名是周五日期)
- **同名覆盖**:同一交易日多次跑直接覆盖,不留多版本
- **写入工具**:用 `Write`,不要 `cat`/`echo` via Bash

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

## 红线(违反必返工)

1. **不编造行情数据** —— 缺数据就标"未获取",决不凭记忆补
2. **跨日复盘前必判断"目标日 vs 当前日"** —— 漏判会拿错数据且模型察觉不到
3. **未来走势必带依据 + 概率语言 + 证伪条件** —— 三缺一即返工
4. **档位 C 的报告不准归档** —— 不污染历史 reports

## 配套文档

- `.cursor/skills/a-stock-closing-review/SKILL.md` —— 复盘 skill 主文档(10 段输出结构、分析框架、自检清单)
- `.cursor/skills/a-stock-closing-review/data-collection.md` —— 数据采集执行手册(Phase 0/1/2、已知坑、降级规则、迭代记录)
- 项目记忆:`~/.claude/projects/-Users-eastonshay-xym-life-stocker/memory/`(自动加载,记录用户偏好和项目演进决策)

## 双栈兼容说明

- **Cursor CLI**:从 `.cursor/skills/` 自动发现 skill,读取 `AGENTS.md` 作为项目指令。
- **Claude Code**:仍读取 `CLAUDE.md`(与 `AGENTS.md` 内容同步)。原 skill 路径 `a-stock-closing-review/` 已迁移至 `.cursor/skills/a-stock-closing-review/`,Claude Code 通过 SKILL.md frontmatter 触发词机制依然可以调起。
- 修改 skill 时只改 `.cursor/skills/` 下的版本即可,两端共用同一份。
