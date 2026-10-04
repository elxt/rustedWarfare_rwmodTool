本工具仅供学习交流使用，切勿随意传播。
本工具由人工智能生成并经过人工修改与实测校对。

# rwmod_tool.py 使用说明

一个用于 **Rusted Warfare（铁锈战争）** `.rwmod` 的读取 / 解包 / 内联解析 / 键校验 / 重新打包工具。
纯 Python 3（3.8+）编写，无任何第三方依赖。

它会：

1. 读取 `.rwmod`（本质是 zip 压缩包），包括文件头被破坏、标准解压库拒读的包；
2. 还原被“封包 / 伪造文件夹”处理的成员名；
3. 按游戏引擎的方式解析并规范化 `.ini` / `.template` / `mod-info.txt`；
4. 修正 ini 中指向被伪装文件的多余结尾 `/`；
5. （可选）把 `${...}` 宏、`copyFrom` 继承展开成最终生效内容；
6. （可选）对节下的键做合法性校验；
7. 重新打包成一个标准 `.rwmod`（默认只产出 rwmod 文件，不输出解包文件夹）。

> 全程在压缩包内部按条目处理，不会先整体解压到临时目录。

参考资料：游戏本体 Rusted Warfare 1.15；GitHub 项目 `n9tank/rwTool`；VSCode 插件 `RWini_Plugin`。

---

## 一、环境要求

- Windows / Linux / macOS 均可；
- Python 3.8 或更高（命令为 `python` 或 `python3`）；
- 无第三方库。

## 二、快速开始

只需要下载rwmod_tool.py

```bash
# 最简用法：解析并规范化 ini，输出标准 rwmod
python rwmod_tool.py 3.rwmod
```

产物：与输入同目录的 `3_modified.rwmod`（标准 rwmod，可直接放进游戏 `mods` 目录）。
默认不输出解包文件夹；需要时用 `-o` 指定。

---

## 三、参数说明

```text
python rwmod_tool.py <输入文件> [选项]

位置参数：
  input                 输入文件

选项：
  -o, --out-dir DIR     可选：额外把解包后的文件写到该目录（默认不输出文件夹）
  --out-rwmod FILE      输出 rwmod 路径
                        （默认：与输入同目录的 <文件名>_modified.rwmod）
  --no-ini-modify       不解析改写 ini/template/mod-info.txt，原样复制
  --resolve             展开 ${...} 宏与 copyFrom / @copyFromSection / 部件级 copyFrom
  --resolve-template    同时展开并内联 all-units.template（隐含 --resolve）
  --keylist-internal    用脚本内嵌的键列表做键名合法性检查
  --keylist-external    用 keyList.txt（脚本目录 → 输入目录）做键名校验
                        （两者互斥；都不给则不做键校验）
  --strip-invalid-keys  配合键列表开关：删除非法键（否则只报告）
  --repair              强制使用修复读取器（文件头损坏时自动启用）
  --no-progress         关闭进度条
  --no-dirs             不在输出 zip 中写入任何目录条目
  --list                只列出条目，不写任何文件
  --selftest            运行内置自测（使用内嵌 ini，不读取你的文件）
  -h, --help            显示帮助
```

> 大文件/大 mod 上做键校验会明显变慢；进度条输出到 stderr，非终端时自动关闭。

---

## 四、常用示例

```bash
# 1. 仅规范化 ini，输出标准 rwmod（最常用）
python rwmod_tool.py 1.rwmod

# 2. 展开宏与继承，保留模板机制
python rwmod_tool.py 1.rwmod --resolve

# 3. 完全自包含（连模板一起内联）
python rwmod_tool.py 1.rwmod --resolve-template

# 4. 自定义输出路径
python rwmod_tool.py 3.rwmod --out-rwmod out.rwmod --resolve --resolve-template

# 5. 需要时额外输出解包文件夹
python rwmod_tool.py 3.rwmod -o out_dir

# 6. 只看包内有哪些文件
python rwmod_tool.py 3.rwmod --list

# 7. 不做任何解析，原样解包再打包
python rwmod_tool.py 3.rwmod --no-ini-modify

# 8. 文件头被破坏的包（自动修复；也可显式强制）
python rwmod_tool.py 5.rwmod
python rwmod_tool.py 5.rwmod --repair
```

---

## 五、展开开关的区别

| 命令 | 效果 |
|---|---|
| 不加开关 | 只做注释/空行清理、重复键/节合并、路径引用修正；保留 `copyFrom`、`${...}`、模板机制 |
| `--resolve` | 展开 `${...}`、`copyFrom`、`@copyFromSection`、部件级 `copyFrom`；`all-units.template` 保持原样，交由游戏运行时套用 |
| `--resolve --resolve-template` | 在上项基础上，再把 `all-units.template` 展开内联进每个单位，得到完全自包含的文件 |

**展开后的清理：**

- 已成功解析的 `copyFrom` / `@copyFromSection` / 部件级 `copyFrom` 会被删除；
- 某文件不再含任何 `${...}` 时，其中的 `@global` / `@define` 定义键会被删除；
- 若仍有无法解析的 `${...}`，则保留其 `@global` / `@define` 供游戏运行时继续解析，
  找不到目标的悬空继承指令也会保留；
- `--resolve-template` 时，`all-units.template` 已内联，这些模板文件会从输出中删除。

**未知节清理（规范化时始终执行）：**

- 删除引擎不识别且无人引用的节（如 `[mx]`）；
- 被 `@copyFromSection`、部件 `copyFrom` 或 `${节.键}` 引用的节一定保留；
- 引擎认可的节类型不删：`core`、`graphics`、`attack`、`movement`、`ai`、`turret_`、
  `projectile_`、`action_`、`effect_`、`leg_`、`arm_`、`attachment_`、`animation_`、
  `hiddenAction_`、`canBuild_`、`placementRule_`、`resource_`、`global_resource_`、
  `decal_`、`comment_`、`template_`；
- `mod-info.txt` 是特例：只允许 `[mod]`、`[maps]`、`[music]`，其它节删除；
  反过来这三个节也不能出现在其它文件里。

**键名合法性检查（可选，默认不做）：**

- 仅在指定 `--keylist-internal` 或 `--keylist-external` 时执行，二者互斥；
- 按“节类型 → 合法键”校验，报告不在列表里的键（含文件/节/键名）；
- 默认只报告不删除；`--strip-invalid-keys` 才删除；
- 放行：`@` 开头的键、`template_*` / `comment_*` 节、列表中没有键组的节；
- 完整解析规则见 [`keyListRule.md`](keyListRule.md)。

---

## 六、输出说明

- **输出 rwmod**（默认唯一产物）：标准 zip；文件名 UTF-8（自动设置 UTF-8 标志位）；
  文件用 deflate、目录用 store；不再有“带数据的尾斜杠条目”。
- **目录条目**：只写输入包中本来就存在的目录条目，**不合成父目录**（合成的父目录会改变引擎
  遍历 mod 的顺序，从而影响 `@global` 变量按加载顺序解析）。`--no-dirs` 则一个目录条目都不写。
- 文本文件按引擎顺序重写为 `[节]\n键:值`；二进制文件（png、ogg、ttf 等）原字节复制。
- **解包文件夹**：仅当传入 `-o/--out-dir` 时输出，目录结构与包内一致。
- **进度条**：stderr 显示 `读取 / 解析 / 展开 / 写出 / 打包`；非终端自动关闭，亦可 `--no-progress`。

**运行结束时打印的统计行（stdout）：**

| 输出行 | 含义 |
|---|---|
| `files written` | 写进输出 rwmod 的**文件条目数**（不含目录条目；`--resolve-template` 删除的模板不计入） |
| `zip header repair` | **仅在使用修复读取器时出现**：输入 zip 头被破坏（`version / flags / crc / size` 非法），改走 EOCD/中央目录 + 本地头偏移 + raw deflate 的修复路径 |
| `disguised names fix` | 被“封包”伪装的成员数：条目名尾部多余的伪造 `/` 被去掉的个数 |
| `ini files parsed` | 被识别为配置并解析重写的文件数（括号内为**节数**、**键数**）；`--no-ini-modify` 时为 0 |
| `path refs fixed` | ini 中指向被伪装文件的引用被去掉多余尾 `/` 的条数（支持 `ROOT:` / `SHADOW:`、逗号多路径）；仅非 0 时打印 |
| `output rwmod` | 生成的 rwmod 文件路径 |

其它可能出现的行：

| 输出行 | 含义 |
|---|---|
| `unknown sections : N removed` | 删除的“引擎不识别且无人引用”的节数 |
| `invalid keys : N (kept/removed) [keyList: ...]` | 键名校验结果（配合键列表开关） |
| `unresolved macros : N file(s)` | `--resolve` 后仍有 `${...}` 未展开的文件数 |
| `templates removed : N file(s)` | `--resolve-template` 时从输出删除的 `all-units.template` 数量 |
| `output folder` | 使用 `-o` 时额外写出的解包目录路径 |

---

## 七、背景知识

### 1. rwmod 与“伪造文件夹（封包）”

`.rwmod` 就是一个 zip。正常情况下：

- 文件条目名不带结尾斜杠，例如 `A/B/C.ini`；
- 空目录条目名带结尾斜杠，例如 `A/B/C/`（0 字节）。

有些打包工具（“封包”）会把真实文件伪装成文件夹：在每个文件条目名后追加 `/`。
游戏仍能读到它们（因为查找文件时会回退尝试 `名字/`）。
本工具会把这种伪造的结尾斜杠去掉并输出标准 rwmod。

### 2. 引擎的 INI 读取规则

本工具模拟游戏本体（Java）的 ini 读取方式：

| 输入 | 处理 |
|---|---|
| 行分隔符 | `\r`、`\n`、`\r\n` 均视为换行（Java `readLine` 语义） |
| BOM / 零宽填充 | 忽略（`\ufeff`） |
| 空行 | 忽略 |
| 行首 `#` | 注释，忽略 |
| 行首 `"` | 整行注释，忽略 |
| `"""` 成对出现 | 块注释；用在值行时=跨行值，把后续物理行拼接到同一个值（物理换行去掉） |
| `[节名]` | 节头；判定时先合并 `"""` 并去除行尾空白/控制填充，`]` 需为逻辑行最后一个有效字符 |
| `[comment_...]` | 整节忽略 |
| `键 : 值` 或 `键 = 值` | 以第一个 `:` 或 `=` 分割，键/值 `trim()` |
| 重复的节 | 合并到第一次出现的位置 |
| 重复的键 | 后者覆盖值，保留第一次出现的位置 |
| 第一个 `[节]` 之前的普通键 | 本工具忽略（自测覆盖该行为） |
| 其它无法识别的行 | 忽略 |

> 混淆包常用 `\r` 换行 + 空白/控制字符 + `"""` 块把内容打散；按上表规则解析后即可还原成正常 ini。

### 3. 继承与宏（`--resolve`）

游戏配置之间可以互相继承、引用；本工具可选地把它们展开：

- **`[core] copyFrom: 路径`**：跨文件继承（相对当前文件目录；`ROOT:` 相对 mod 根；`CORE:` 相对内置单位）。
- **`[core] copyFrom: A, B`**：多个来源，后列出的优先级更高。
- **`@copyFromSection: 节A, 节B`**：段级继承。
- **部件级 `copyFrom`**：写在 `turret_* / leg_* / projectile_* / action_*` 等节里，
  值是同级后缀，例如 `[leg_2] copyFrom: 1` → 复制 `leg_1`。
- **`${宏}`**：
  - `${节.键}` 取指定节的值；
  - `${section.键}` / `${当前节名.键}` 取当前节的值；
  - `${名}` 先找当前节的 `@define 名`，再找 `@global 名`；
  - 含 `+ - * / % ^ ( )` 时按数值计算，支持 `int/cos/sin/sqrt`；
  - **含逗号的值不是算式**（如坐标 `1,2,3,4`），原样保留。
- **`@define 名: 值`**：文件内局部宏；**`@global 名: 值`**：本文件内可见的全局宏。
- **`all-units.template`**：引擎把它作为默认值套用到同目录（及上级目录）的单位上；
  标记 `dont_load` 不会被套用。

优先级（高 → 低）：本文件 > 后列出的 copyFrom > 先列出的 copyFrom > all-units.template。

> 两个“文件/节自身”的控制标记不随继承复制：
> - `[core] dont_load`：从带 `dont_load:true` 的模板复制出来的单位仍会正常加载；
> - `@copyFrom_skipThisSection`：只对被复制方自身的节生效。

### 4. 损坏的 zip 文件头与修复读取器

有些 rwmod 的 zip 头被改写（`version made by / version needed / flags / crc / 压缩与解压大小`
被随机化），标准库会直接报错（Python `zipfile` 会因 version 非法而拒绝），但游戏仍能读。

本工具内置修复读取器：

- 从 EOCD / Zip64 EOCD 取到中央目录；
- 用中央目录中仍正确的“本地头偏移”定位每个条目；
- 忽略被改坏的 `version / flags / crc / size` 字段，直接对数据流做 **raw deflate 解压**，
  以解压流的自然结束点确定真实长度。

修复是自动的（正常读取失败时启用），也可用 `--repair` 强制启用。修复读取器只读输入，不修改原文件。

### 5. 配置文件的识别

为避免把二进制文件误当 ini，工具按以下规则判定哪些成员是配置：

1. 名字以 `.ini` / `.template` 结尾，或为 `mod-info.txt`；
2. `[core] copyFrom` 指向的文件（用与引擎一致的路径解析递归跟踪）；
3. 兜底：无扩展名、能按 UTF-8/GBK/Big5 解码、且能解析出引擎可识别节的成员。

### 6. ini 内路径引用的修正

这类包把文件存成 `名字/`（伪文件夹），ini 也用带结尾 `/` 的路径引用它。工具在读入时已去掉成员名的
结尾 `/`，因此也会去掉 ini 中指向真实文件的引用结尾 `/`：

- 支持 `SHADOW:` / `ROOT:` 前缀、逗号分隔的多路径、相对当前 ini 目录/父目录/根目录；
- 内置 `SHARED:` / `CORE:` 不动；
- 仅当“去掉 `/` 后按引用规则能解析到包内真实文件”时才去掉；仍指向目录/非文件的值保持原样；
- `@` 开头的键（`@define` / `@global` 等）的值不受影响；
  若某个 `@global` / `@define` 的值确实是路径，请用 `--resolve`：宏展开后值会落到普通键上，那时会继续按上述规则去掉尾 `/`。

统计里的 `path refs fixed` 即修正条数。

---

## 八、注意事项与已知限制

1. **`CORE:xxx` 引用未展开**：它指向游戏安装包 `assets/units` 里的内置单位，工具只处理 mod 自身文件。
2. **不可解析的宏会原样保留**：某些基础模板自身引用的宏由使用它的单位提供，会保持宏不变。
3. **悬空路径会保留**：继承目标确实找不到时，指令保留（游戏同样无法解析）。
4. **`ROOT:` 的兼容匹配**：先按 mod 根解析；若找不到（封包时把 mod 文件夹名去掉了），
   会自动逐段去掉前导路径再匹配。
5. **文本编码**：优先 UTF-8，失败回退 GBK / Big5。
6. **行尾**：输出统一 `\n`，对游戏无影响。
7. `--resolve` 与 `--no-ini-modify` 同时使用时，`--resolve` 不生效。
8. `--resolve` 不会把 `footprint` / `constructionFootprint` 等“逗号坐标值”当算式求值，逗号列表原样保留。

---

## 九、自测

```bash
python rwmod_tool.py --selftest
```

测试内容：INI 解析（注释/空行/重复键/重复节/跨行值/混淆节头）、宏展开、`copyFrom` 继承、
`@copyFromSection` 与数值计算、逗号值不被误算、未知节清理、键名合法性校验、路径引用斜杠修正。
输出 `SELFTEST OK` 即正常。

> 在游戏内验证时，请在一次连续运行里观察；若反复强杀进程，游戏会进入 **safe mode** （`preferences.ini` 的 `numLoadsSinceRunningGameOrNormalExit`），导致跳过自定义单位加载，造成“零报错”的假象。测试前将该值置 0，并确认 stdout 出现 `Loading units from mod`。

---

## 十、文件说明

| 文件 | 说明 |
|---|---|
| `rwmod_tool.py` | 主程序（单文件） |
| `README.md` | 本说明文档 |
| `keyList.txt` | 键名合法性检查用的合法键列表（外部） |
| `keyListRule.md` | `keyList.txt` 的解析规则说明 |
| `*_modified.rwmod` | 默认输出的规范化标准 rwmod |
| `*_unpacked/`（可选） | 仅在传入 `-o` 时生成的解包目录 |
| `chat*.json`（非必要） | 与agent的聊天纪录，也许不完整 |

> `keyList.txt` 已内嵌进工具中，`--keylist-internal` 使用的就是它（去说明行的精简版）。

---

## 十一、反混淆脚本 rwmod_decompile.py

`rwmod_decompile.py` 是配套的“反编译”脚本（依赖同目录的 `rwmod_tool.py`，使用时会将其编译为pyc），把一个 rwmod展开成按内容命名的可读源码目录，便于阅读、研究和手工编辑：

1. 与主工具共用读取器：自动修复损坏的 zip 头、去掉伪文件夹结尾 `/`；
2. 按引擎规则解析所有配置，包括伪装成 `.ogg`、完全没有扩展名的隐藏配置（以及 `copyFrom` 链上的文件）；
3. 内容驱动文件命名：
   - 配置：取生效内容里的 `[core] displayText` → `displayName` → `name`；被继承的基础配置取`<首个引用它的单位>_base`；
   - 图片 / 音效 / 字体：取引用它的配置名 + 键名，如 `AA_Beam_Gunship_image.png`；
   - 无法从内容命名的文件回退为原名的可见转义（零宽 / 私有区 / 控制字符 → `x202e_` 这类标记），保持唯一；
4. 二进制按文件签名补回扩展名（png / ogg / wav / mp3 / jpg / ttf …）；
5. 同步改写配置内所有路径引用（`copyFrom`、`image`、`sound`、`ROOT:` / `SHADOW:` …），使目录内部自洽；
6. 清除节名 / 键 / 值中的私有区、零宽方向、控制填充字符。

输出结构：

```text
<out>/units/       引擎会自动加载的配置（原 .ini）
<out>/bases/       仅通过 copyFrom / template 引用的配置（原 .ogg、无扩展名等）
<out>/templates/   all-units.template（文件名保持不变）
<out>/resources/   图片、音效、字体等
<out>/misc/        其它（地图、未知二进制）
<out>/mod-info.txt
```

常用命令：

```bash
python rwmod_decompile.py 5.rwmod                   # 默认输出 5_decompiled/
python rwmod_decompile.py 3.rwmod --resolve         # 展开宏/继承后再输出
python rwmod_decompile.py 5.rwmod --config-ext keep # 保留配置原扩展名
python rwmod_decompile.py 5.rwmod --list            # 只看 原名 -> 新名 映射
python rwmod_decompile.py --selftest
```

参数：`-o/--out-dir`、`--resolve`、`--resolve-template`、`--repair`、`--no-progress` 与主工具一致；
另加 `--config-ext {ini,keep}`（配置扩展名策略，默认 `ini`）与 `--list`。

注意：

- 该目录面向 阅读 / 编辑 。路径引用已同步改写，直接压回 rwmod 通常可用，但按角色分组可能改变目录作用域行为（如 `all-units.template` 的适用范围）；需要行为一致的标准 rwmod 请用 `rwmod_tool.py`。
- `--config-ext ini`（默认）会把隐藏配置改名为 `.ini`，压回后可能被引擎自动加载；推荐保持原扩展名，用`--config-ext keep`。
- `*_decompiled/` 是生成的反混淆源码目录
- 此反混淆程序不保证输出结果可直接使用

---
