# Repository Guidelines

## 项目结构与模块组织
`source/` 是 Cocos Creator 扩展的 TypeScript 源码目录。核心入口包括 `source/main.ts`（扩展生命周期与面板消息）、`source/mcp-server.ts`（本地 HTTP MCP 服务）、`source/tools/`（场景、节点、预制体、资源等工具实现）以及 `source/panels/`（基于 Vue 的编辑器面板）。`source/test/` 当前主要放手工测试辅助脚本，不是完整自动化测试体系。`static/` 存放面板 HTML/CSS 资源，`i18n/` 存放多语言文本，`dist/` 为编译产物，不要手动修改。

## 构建、测试与开发命令
开始前先执行 `npm install`，安装 `typescript` 和依赖。

- `npm run build`：使用 `tsc` 将 `source/` 编译到 `dist/`。
- `npm run watch`：以监听模式持续编译，适合开发扩展时使用。
- `node TestScript.js`：运行仓库内的独立辅助脚本。

本项目需要在 Cocos Creator 3.8.6+ 中加载和验证；不要把它当作独立 Web 服务来调试。

## 编码风格与命名约定
使用 TypeScript，保持 4 空格缩进和分号风格，与现有代码一致。工具实现按领域放在 `source/tools/*-tools.ts` 中。类名使用 `PascalCase`，方法和变量使用 `camelCase`，导出的 MCP 工具名使用 `snake_case`，例如 `get_current_scene`、`create_prefab`。新增能力时优先复用现有 `Editor.Message` 调用模式，不要随意引入新抽象。

## 测试指南
仓库目前没有成熟的自动化测试框架。新增编辑器能力时，可在 `source/test/` 下补充针对性的手工测试文件，例如 `prefab-tools-test.ts`。每次改动至少验证以下内容：

- TypeScript 编译通过。
- 扩展能在 Cocos Creator 中正常加载。
- 受影响的 MCP 接口或面板操作能在真实项目中工作。

## 提交与 Pull Request 规范
现有 Git 历史以简短祈使句为主，例如 `Update README.md`。提交信息应简洁、具体、聚焦单一主题，避免混入无关改动。PR 需要包含变更摘要、影响范围、手工验证步骤；如果修改了面板或 UI，请附截图；如有关联问题单，请一并链接。

## 安全与配置提示
MCP 服务默认应绑定到 `127.0.0.1`，除非有明确需求，不要改成对外暴露。配置会写入宿主 Cocos 项目的 `settings/` 目录，涉及配置变更时，优先在临时项目中验证，避免污染正式工程。
