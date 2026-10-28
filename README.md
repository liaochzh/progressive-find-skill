# progressive-find

一个用于 **目标定位** 的 agent skill：当你知道要找什么，但不知道它在哪（文件、代码、配置、文档、网页、工具），按「分层搜索 + 预算上限 + 四态交付」把它找出来，而不是把工具调用堆到预算耗尽。

> 面向 opencode / Claude 一类支持 `SKILL.md` 的 agent 运行时。纯提示词实现，无依赖、无脚本、不联网也能用。

## 它解决什么问题

| 常见失败 | 本 skill 的做法 |
|---|---|
| 一上来就全网搜，或反复换关键词硬试 | 先判定 `kind`（本地符号 / 报错文本 / 公开库选型 / 网页 / 设计意图），据此**选定起始层** L1–L5 |
| 搜到相似名字就当成命中 | 命中必须读 ≥30 行上下文或完整小节，按「类型 + 内容 + 可访问性」三项验证后才给 `high` |
| 预算无限膨胀，最后交一个「没找到」 | 预算只在一处定义（每层 ≤3、总 ≤12 次搜索；取证读不计入；进入验证的候选 ≤5），触顶即 `STOPPED` 并列出剩余候选 |
| 结论无法复核 | 四态输出：`FOUND` / `NOT FOUND` / `STOPPED`（+ 第四态「不可核销」由回归用例约束），每条命中必带 `EVIDENCE: <path:行号 | URL@版本>` |

## 何时触发

用户说「找一下 / 帮我找 / 搜索 / 帮我定位 / Where can I find」，或问「这个在哪定义 / 哪个文件控制 / 这报错从哪来 / 有没有现成的库 / 这段代码的出处是哪」，且目标位置或来源未知、需要多轮搜索验证时。

**不适用于**：代码库结构探索（用 explore 一类子代理）、Bug 根因定位（用 debug 一类子代理）、报错的修复与堆栈分析（用 `code-error-fixer` 一类 skill）。skill 的 `description` 里已写明这些排除关系。

## 安装

```bash
# 方式一：整个目录放到 skills 根目录
git clone https://github.com/liaochzh/progressive-find-skill.git
cp -R progressive-find-skill ~/.config/opencode/skills/progressive-find

# 方式二：软链（便于跟随仓库更新）
ln -s "$(pwd)/progressive-find-skill" ~/.config/opencode/skills/progressive-find
```

不同运行时的 skills 目录不同（例如 Claude Code 的 `~/.claude/skills/`），把本目录放进去即可，`SKILL.md` 的 frontmatter（`name` / `description`）就是触发依据。

## 结构

```
SKILL.md                    # 主体：核心原则 → 步骤 0/0.5/1/2/2.5/3/4/5 → 示例 → 交付前自检
references/search-recipes.md # 按需读：本地 rg 变体、GitHub raw/API、包元数据、网页与存档
```

`SKILL.md` 约 8 KB（自设口径 ≤8192 字节 / ≤175 行），只在需要时加载；`references/` 引用只一层深。

## 设计要点

- **预算单一真源**：数字只在「步骤 2」定义，其他章节只引用，避免多处数字漂移。
- **两本账**：搜索调用与取证读分开计，读证据不受搜索预算惩罚。
- **停止判据 ①②③**：连续 3 层无命中 / 相互冲突候选 ≥3 / 搜索满 12 次（软上限，继续须调用方授权）。
- **四行头**：多候选也先给 `FOUND` / `EVIDENCE` / `CONFIDENCE` / `CAVEATS`，再列表——光有表格不算终态。

## 关于 `references/search-recipes.md` 里的实测结论

其中若干条是作者本地环境（2026-10）的实测结果，例如：

- npm registry 发现型接口 `/-/v1/search` 可用；
- GitHub `search/*` 未认证约 10 次/分钟，并发会 `ECONNRESET`；`search/code` 无 token 直接 401；
- `crates.io` 用 `curl` 默认 UA 会被 403，需自定义 UA；
- `api.npms.io` 数据停更在 2023，不要用来核对版本；
- `pypi` / `crates` 的全量 JSON 体积可达数百 KB，先落文件再取字段。

这些结论有日期，接口行为可能变化——照抄前请自行确认。

## License

[MIT](LICENSE) © 2026 liaochzh
