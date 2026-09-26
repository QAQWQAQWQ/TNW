# TNW 国家标签审计与崩溃归因报告

**审计日期**：2026-09-26
**审计基准**：git HEAD = `e1247b71`（"9.18"，工作区干净，URF/NRR/SPU 均已不存在）
**游戏版本**：Operation Postern v1.19.3.0.c01a
**模组声明版本**：`supported_version="1.18.*"` ← 与游戏版本不匹配

---

## 一、结论：tag 槽位上限假说不成立

用户怀疑"tag 槽位到达上限导致闪退"。经全量核查，该假说被多项独立证据否证：

| 检查项 | 实测结果 | 是否支持"上限"假说 |
|--------|----------|------------------|
| 已注册 tag 数 | **423** | 否（原版 TNO master 实测 **527**、创意工坊版 **496**，TNW 反而更少） |
| 重复定义的 tag | **0** | 否 |
| 零引用 tag | **0**（最低引用 5 次） | 否 |
| 崩溃时是否进入开局日期 | **是**，多数崩溃已载入 `2002.01.01` | 否（上限问题会在加载期就崩） |
| 日志中 `Invalid Country Tag` | **0 条** | 否 |
| 工作区完全回退（无 URF）后是否仍崩 | **仍崩**，今日 14 次 | 否（决定性） |

**最关键证据**：今日（09-26）工作区已回退到 HEAD 干净状态，URF、NRR、SPU 三个 tag 全部不存在，注册表回到 423 个，但游戏**依然崩溃 14 次**，崩溃偏移分散在约 13 个不同位置（`0x277F`×6、`0x06CE`×2、`0xDEB7`、`0x8ED1`、`0x9F3C`、`0x7EDF`、`0xF1AE`、`0x4AD8` 等）。若崩溃由 tag 数量引起，移除 tag 后不应继续崩溃。

**补充反证**：崩溃偏移 `0x600D` 最早出现在 **2026-09-22 17:38**，那次运行加载的是 **TNO 原版模组**（`Mods: TOT TWO The New Order...`），当时 TNW 里根本不存在 URF/NRR/SPU，但同样崩在 `0x600D`。说明该崩溃点与新增 tag 无因果关系。

> 磁盘空间已排除：C 盘剩余 9.77 GB、D 盘剩余 4.67 GB，均非耗尽状态。

---

## 二、真正的崩溃线索（按可信度排序）

### 线索 1：GUI 崩溃补丁在回退中丢失（已确认，高可信）

回退报告 `TNW_Restoration_Report.md` 记载，曾手工添加 `country_filter` GUI 补丁以修复
`Undefined GUI_TYPE country_filter - This will most likely crash the game` 崩溃。

**当前实测**：

| 项目 | 期望值 | 实测值 | 状态 |
|------|--------|--------|------|
| `frontendgamesetupview.gui` 中 `country_filter` | 3 处 | **0 处** | ❌ 补丁丢失 |
| `frontendgamesetupview.gui` 中 `more_countries` | 1 处 | **0 处** | ❌ 补丁丢失 |
| `frontendgamesetupview.gfx` 中 `GFX_country_filter_entry` | 1 处 | **0 处** | ❌ 补丁丢失 |

**关联崩溃**：`hoi4_20260926_132008`（偏移 `0x9F3C`）日志中出现 **13 条 `Undefined GUI_TYPE`** 错误。

> 注：`gfx/interface/country_filter_entry.dds` 在模组内**不存在**，但原版游戏
> `D:\Steam\steamapps\common\Hearts of Iron IV\gfx\interface\country_filter_entry.dds` **存在**，
> 因此补丁可以回落到原版贴图，无需另建素材。

### 线索 2：模组声明版本与游戏版本不匹配（中可信）

`descriptor.mod` 声明 `supported_version="1.18.*"`，实际游戏为 `v1.19.3`。
跨大版本时 GUI 定义、effect 语法与接口字段均可能变更，是 `Undefined GUI_TYPE` 类崩溃的常见根因。

### 线索 3：加载期脚本报错（低可信，属噪音）

`common/national_focus/tno_italy_scorza_shared.txt:10510` 附近反复报
`Unknown effect-type: GER`（单次运行 193–242 条）。

已核实该文件括号配平（2669/2669），且 `add_to_trade_modifier_PREV` 确实定义于
`common/scripted_effects/TNO_trade_scripted_effects.txt:701`。报错源于脚本中
`GER = { add_to_trade_modifier_PREV = yes }` 这种把 tag 当 effect 容器的写法在新版本解析失败。
**该类错误在 09-22 之前就已存在，且未阻止游戏进入开局日期，属警告级而非致命。**

### 已排除的非致命项

以下报错看似可疑，但经验证**不致命**，与崩溃无因果关系：

- `URF` 旗帜缺失（`File not found`）：09-24 19:20 那次崩溃时旗帜**存在**、零旗帜报错，却崩在完全相同的偏移 `0x600D`。同期 `GNG` 旗帜为 24bpp 警告，文件存在，不致命。
- `Portrait_GER_Kaiser_Speer.png` 缺失：仅 2 处引用，属贴图缺失警告。
- `RUS_stupid_ai_econ` 未定义：`add_ideas` 无效理念警告，引擎跳过处理。

---

## 三、可删 / 可注释的 tag 清单

### 3.1 已注册的 423 个 tag：**一个都不能删**

全量扫描 6123 个脚本文件（71.9 MB）统计每个 tag 的被引用次数：

- **零引用 tag：0 个**
- **引用次数最低值：5 次**（QUE、UYG、WSM、PUR、ESM）
- 低于 10 次引用的仅 38 个，均仍有实际脚本调用

结论：不存在"无关紧要、可安全删除"的已注册 tag。删除任何一个都会使引用它的脚本
（焦点树、决议、事件、ideas、scripted_effects）出现空引用，可能引发**比当前更严重的**崩溃。

### 3.2 真正的异常方向相反：**有 35 个 tag 被引用却未注册**

这些才是日志里 `is not in the tag list`（单次运行 76 条）的来源。
它们的正确处理方式是**补注册**，而不是删除——删除引用它们的上百处脚本代价极高。

#### A 类：有 history 文件但未注册（16 个）

| tag | 引用次数 | 87 个作用域块 | 说明 |
|-----|---------|-------------|------|
| HGR | 351 | 87 | Heydrich's Germany |
| SGR | 196 | — | Speer's Germany |
| FRI | 187 | — | Free Indonesia |
| GGR | 173 | — | Göring's Germany |
| BGR | 167 | — | Bormann's Germany |
| IRK | 117 | — | Irkutsk（**同时是 3 个地块的 core**：566-Irkutsk、567-Tulun、890-Kirensk） |
| NII | 103 | — | Negara Islam Indonesia |
| SRB | 102 | — | Soerabaja |
| BKB | 98 | — | Bukit Barisan |
| MKS | 97 | — | Makassar |
| SLS | 93 | — | Sulawesi Selatan |
| PMT | 81 | — | Permesta |
| ABD | 74 | — | Angkatan Bersendjata Demokrasi Rakjat |
| DMP | 73 | — | Dewan Militer PRIM |
| DMS | 73 | — | Dewan Militer Siliwangi |
| JJK | 37 | — | Jogjakarta |

> 这 16 个文件在 `history/countries/` 下存在，但 `common/country_tags/00_countries.txt`
> 中没有对应注册行，导致日志报 `Unknown history file in country database`（单次 32 条）。

#### B 类：无 history 文件，被当作作用域与 core 使用（19 个）

| tag | 作用域块数 | 用途 |
|-----|-----------|------|
| SAM | 17 | 萨马拉（250/251/755/850/851 的 core） |
| VYT | 14 | 维亚特卡（399/400/752/865 的 core） |
| WRS | 14 | 西俄罗斯（214/262/860/861/862/869/870 的 core） |
| NOV | 12 | 新西伯利亚（40/570 的 core） |
| KEM | 11 | 克麦罗沃（569 的 core） |
| OMS | 11 | 鄂木斯克（571 的 core） |
| PRM | 10 | 彼尔姆（753/864 的 core） |
| AMR | 9 | 阿穆尔（886 的 core） |
| BRY | 9 | 布里亚特（565/759 的 core） |
| DRL | 8 | 乌拉尔（846 的 core） |
| ONG | 8 | 奥涅加（858/859 的 core） |
| SBA | 8 | 西伯利亚黑军（568/758 的 core） |
| URL | 8 | 乌拉尔（847/848 的 core） |
| SVR | 7 | **斯维尔德洛夫斯克**（653/580 的 core） |
| TYM | 7 | **秋明**（403/572/754 的 core） |
| CHT | 6 | 赤塔（563 的 core） |
| MGN | 6 | 马格尼托哥尔斯克（582 的 core） |
| ORE | 5 | 奥伦堡（652/849/852 的 core） |
| ZLT | 3 | **兹拉托乌斯特**（573/871 的 core） |

> **加粗的 TYM、SVR、ZLT 正是 URF 七个地块的 `add_core_of` 核心标签。**
> 它们未注册是模组的**原有状态**（09-25 08:05 无 URF 的运行中同样报这 5 个），
> 并非本次新增 tag 引入，游戏也能正常进入开局日期。

---

## 四、建议的处理顺序

1. **先恢复 GUI 补丁**（线索 1）：这是唯一已确认的崩溃成因，且补丁内容明确、可精确重建。
2. **确认版本兼容性**（线索 2）：若仍崩，需判断是否将模组适配到 1.19，或降级游戏到 1.18。
3. **A 类 16 个 tag 补注册**：在 `common/country_tags/00_countries.txt` 中按其所属地区补上注册行，消除 32 条 `Unknown history file` 报错。
4. **B 类 19 个 tag 保持现状**：它们是 core 占位符，未注册不影响运行；若要消除日志噪音，需评估脚本影响面后再动。
5. **不要删除任何已注册 tag**。

---

## 五、审计方法与数据完整性说明

- 引用统计范围：`common/`、`events/`、`history/`、`interface/`、`localisation/` 下全部 `.txt`，
  排除 `gfx/` 与 `common/country_tags/`（避免自引用计数）。
- 编码处理：全部按 latin-1（28591）透传读取，避免破坏原有编码与 CRLF/LF 换行。
- 崩溃样本：`Documents\Paradox Interactive\Hearts of Iron IV\crashes` 下 39 个崩溃记录，
  其中近三日 26 个逐条解析异常偏移、加载模组、是否进入开局日期与错误关键词。
- 引用次数统计的完整结果见 `D:\TNW develop\tag_refcount.txt`（前 200 项，按引用数升序）。
