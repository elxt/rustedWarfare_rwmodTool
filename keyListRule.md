# keyList.txt 解析规则

本文说明 `rwmod_tool.py` 如何解析键名列表（外部 `keyList.txt` 与内嵌副本使用完全相同的规则）。

## 一、整体格式

- 文件按行读取，编码为 UTF-8（允许 BOM）。
- 每行先 `strip()` 去除首尾空白。
- 空行忽略。
- 说明行忽略（以以下前缀开头的行）：
  - `TYPE含`
  - `[animationKey]同`
  - `N为任意内容`
  - `可以存在于任何节`
  - `X为任意英文内容`

## 二、节头（section header）

若某行**整行**由一个或多个 `[...]` 组成（正则 `^(?:\[[^\]]+\])+$`），则它是一个节头：

- 例如：`[core]`、`[turret_NAME]`、`[arm_#][leg_#]`。
- 用 `\[([^\]]+)\]` 提取出所有方括号内的 token；该节头后、下一个节头前的每一行都属于这些 token 的**键模式列表**。
- 允许多个 token 共用一个键列表（如 `[arm_#][leg_#]` 表示 `arm_*` 与 `leg_*` 共用）。

### token 与节名的匹配规则

设待校验节名为 `S`，节头 token 为 `T`：

| token 形式 | 匹配条件 | 示例 |
|---|---|---|
| 普通名（无 `_`） | `S == T` | `core` 匹配节 `core` |
| `T = X_NAME` 或 `X_NANE` | `S` 以 `X_` 开头 | `turret_NAME` 匹配 `turret_1`、`turret_R` |
| `T = X_#` | `S` 以 `X_` 开头 | `arm_#` 匹配 `arm_1` |
| `hiddenAction_NAME` | 额外匹配 `action_*` | `action_升级`、`hiddenAction_研发` |

> 注意：`template_NAME`、`comment_NAME` 属于节头，但对应节在校验时被整体豁免（见第五节）。

## 三、键模式（key pattern）

节头之后的每一行是一个键模式。匹配某个具体键 `K` 时：

| 模式写法 | 含义 | 编译结果 |
|---|---|---|
| 普通键名 | 字面匹配 | 转义后的字面量 |
| `#` | 任意数字 | `\d*` |
| `[Language]` | 任意后缀 | `.+` |
| `[actionKey]` | 任意动作键（取自 `[hiddenAction_NAME]` 组的键） | 候选交替 |
| `[canBuildKey]` | 任意建造键（取自 `[canBuild_NAME]` 组的键） | 候选交替 |
| `[animationKey]` | 任意内容 | `.+` |
| `[time]` | 任意内容 | `.+` |
| `animation_TYPE_[animationKey]` | 特判 | `animation_(idle\|moving\|attack)_.+` |
| `mutatorN_xxx` | N 为任意内容 | `mutator.*_xxx` |
| `name/pos/isLocked:` | 组合写法 | 展开为 `…_name`、`…_pos`、`…_isLocked` |

- `[actionKey]` / `[canBuildKey]` 的候选集合来自对应的 `[hiddenAction_NAME]` / `[canBuild_NAME]` 节头分组下收集到的键。
- 匹配时先按节头找到适用的分组；只要键匹配该分组内**任一**模式即视为合法。

## 四、放行规则（不受列表限制）

以下键/节直接视为合法：

- 任何以 `@` 开头的键（`@define`、`@global`、`@copyFromSection`、`@copyFrom_skipThisSection`、`@memory` 等）。
  - 其中 `@define X` / `@global X`：`X` 为任意英文内容（可带下划线），**其 value 不受任何限制**；
    工具只按原样读取/解析宏，绝不把该类 value 当作路径或其它格式去处理。
- `template_*`、`comment_*` 节（其键复制自其它节类型，无法按名判断）。
- 没有任何节头能匹配到的节（例如自定义节）：因无法判定，一律放行。

## 五、匹配流程

对每个 `(节名 S, 键名 K)`：

1. 若 `K` 以 `@` 开头 → 合法。
2. 若 `S` 以 `template_` 或 `comment_` 开头 → 合法。
3. 遍历节头，收集所有 token 能匹配 `S` 的分组：
   - 若一个都没匹配到 → 合法（无法判定）。
   - 否则，只要 `K` 匹配任一收集到的键模式 → 合法；否则非法。

## 六、内嵌副本（精简）

`rwmod_tool.py` 内的 `KEYLIST_INTERNAL_B64` 是 `keyList.txt` 的精简内嵌副本：

1. 先去掉空行与说明行（`trim_keylist_text`）；
2. 再 `zlib` 压缩、`base64` 编码。

使用时按相反顺序解码后再套用以上同一套解析规则。精简只删除说明文字，不影响任何节头与键模式。

## 七、校验开关

键校验是可选项，仅在显式指定时执行：

- `--keylist-internal`：使用内嵌副本。
- `--keylist-external`：读取脚本目录（其次输入文件目录）下的 `keyList.txt`。
- 二者互斥；都不给则不做键校验。
- `--strip-invalid-keys`：配合上面任一开关，把非法键从输出中删除（否则只报告）。
