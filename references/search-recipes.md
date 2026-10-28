# 查询配方（按需读）

只在需要构造精确查询、或网络源返回大量噪声时读本文件。全部条目在 2026-10 的 opencode 环境实测过；标「未验证」的不要照抄。

## 一、本地精确检索

优先级：先窄后宽。每换一次 pattern 算一次搜索调用。

```
rg -n "精确符号" <path>          # 带行号，最常用
rg -l "模式" <path>              # 只要文件列表
rg -F "含 . * 的字面量" <path>    # 字面量匹配，避免正则转义
rg -t ts "模式" <path>           # 按语言类型过滤
rg -g "*.dart" "模式" <path>     # 按 glob 过滤
rg --hidden -g "!.git" "模式"    # 含隐藏文件
rg -U "多行[\s\S]*模式" <path>   # 跨行匹配（配置块、函数体）
```

- 找不到时先降约束：去掉 `-t`/`-g`、放宽大小写、拆成更短的词根。
- 配置类关键词常用变体：`kebab-case`、`snake_case`、`camelCase`、全大写常量。
- 已知是某个字符串被引用时，用反向检索（谁 import / 谁调用）比顺向搜更有效。

## 二、GitHub 取内容

```
https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>
https://api.github.com/repos/<owner>/<repo>/contents/<path>
```

- 读单个文件优先用 `raw`：返回纯文本，无 HTML 噪声，长度可控。
- `contents` 接口返回 base64，适合取目录清单，不适合读长文件。
- 需要固定版本时把 `<branch>` 换成 tag 或 commit sha。

## 三、GitHub 仓库检索（发现型）

```
https://api.github.com/search/repositories?q=<keyword>&sort=stars&per_page=25
```

- 认证前提：无 token 可调，但限流严格——实测未认证搜索配额约 10 次/分钟（响应头 `x-ratelimit-remaining` 落到 9）；**串行调用**，并发或连打会 `ECONNRESET`（不是 403），失败后串行重试 1 次即可。
- 匹配范围包含 README，**噪声大**：`q` 里加 `in:name,description` 收窄，结果仍须人工筛。
- 响应是单行 JSON，直接读会很占上下文：先落到文件再按需 grep，或只取前几条。
- **已验证**：`search/code`（代码内检索）无 token 直接 401 `Requires authentication`，别默认能用；代码内检索改走本地 `rg`。
- 普通 REST（`/repos/<owner>/<repo>`、`/repos/.../contents/...`）不受搜索配额限制（实测剩额 57），可放心用。

## 四、包元数据（需已知名称）

```
https://registry.npmjs.org/<pkg>                        # 全部版本与 dist-tags
https://registry.npmjs.org/<pkg>/latest                 # 最新版元数据
https://registry.npmjs.org/-/v1/search?text=<kw>&size=10 # 发现型（已验证可用，返回 objects[].package）
https://pypi.org/pypi/<pkg>/json
https://crates.io/api/v1/crates/<pkg>
```

- 前三类只适合**已知包名**的核实（是否存在、最新版本、发布时间）。
- **体积**：`/<pkg>/latest` 约 2–3 KB，可直接读；`pypi/.../json` 与 `crates.io/...` 带全部历史版本，实测 `requests` 193 KB、`serde` 441 KB ⇒ **不要直接读，先落文件只取 `info.version` / `crate.max_version`**。
- **User-Agent**：crates.io 用 `curl` 的默认 UA 会 **403**，必须显式传自定义 UA（如 `opencode/1.0`）+ `Accept: application/json`。
- **已知失效源**：`api.npms.io/v2/package/<pkg>` 实测（2026-10）仍返回 `analyzedAt: 2023-01-13`、`react@18.2.0` —— 数据停更三年，**别用它核对版本**，直接用 `registry.npmjs.org`。
- 「有没有现成的库」这类发现型问题，元数据接口基本无效：先走官方文档站或仓库检索，再回来核实具体包。
- **批量核实打折**：一次核实 **>5 个包**必然撞「候选 ≤5 / 搜索 ≤12」两道墙（实测 7 个包：5 个验完即 `STOPPED: ③`，剩 2 个进 ASK）。>5 项先按 5 个一批拆分，或直接给分批 ASK，别硬跑。

## 五、网页与存档

- 抓正文优先 `webfetch` 的 markdown 格式，省 token。
- 页面 404/改版时改用存档：`https://web.archive.org/web/2024/<原 URL>`。
- 搜索接口不可用时，退回「官方文档站 + 站点内检索页 + 仓库 README」三件套，别在通用搜索上反复重试。
- **本机实测**：`websearch` 可能直接返回 HTTP 402（额度），此时**不要重试第二次**——直接转 `webfetch` 官网/registry 或本地 `rg`，并在 CAVEATS 里写明降级原因。
