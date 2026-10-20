<!-- CODEGRAPH_START -->

## CodeGraph

本仓库已由 CodeGraph 建立索引（存在 `.codegraph/` 目录）。需要**理解或定位代码**时，优先使用 CodeGraph，而不是先 grep/find 或逐文件 Read：

- **MCP 工具**：`codegraph_explore`（本项目 Trae 中的 MCP 服务器名为 `mcp_codegraph`，暴露单一工具 `codegraph_explore`）一次调用即可返回相关符号的原文、带行号源码与调用路径——包括 grep 无法追踪的动态分发跳转（回调、事件、多态分发）。在查询中直接给出文件或符号名即可读取其当前源码。若工具被列为延迟加载，通过工具搜索按名加载。
- **Shell 兜底**：`codegraph explore "<符号名或问题>"` 输出与 MCP 工具完全一致；`codegraph node <符号>` 查看单个符号的源码与调用链。

### 使用原则

- **直接回答，不要委派探索**：一次 `codegraph_explore` 通常足以回答整个问题；需要更多信息时，再调用一次并给出更具体的符号名。CodeGraph 本身就是预建索引，再派发文件读取子代理或执行 grep + read 循环，等于重复它已完成的工作。
- **信任索引结果**：索引来自完整 AST 解析。不要用 grep 重复验证——更慢、更不准且浪费上下文。
- **Read/Grep 只在索引未覆盖时使用**：仅用于确认 CodeGraph 未返回的具体细节，或它不索引的内容（配置文件、文档等）。
- **索引滞后**：若响应开头出现 "⚠️ Some files referenced below were edited since the last index sync…"，仅对横幅中列出的文件直接 Read；不在横幅中的文件索引为最新，以 CodeGraph 为准。

### MCP 配置位置

- 项目级：`.trae/mcp.json`
- 用户级（Trae Solo）：`~/.trae/mcp.json`
- 用户级（Trae CN）：`~/.trae-cn/mcp.json`

若仓库没有 `.codegraph/` 目录，跳过 CodeGraph——是否建立索引由用户决定。

<!-- CODEGRAPH_END -->
