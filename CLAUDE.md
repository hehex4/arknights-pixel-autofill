# 项目约定

本项目的约定统一维护在 [AGENTS.md](AGENTS.md),请以该文件为准。

要点速览(详见 AGENTS.md):

- **设计文档 / spec / 实施计划 / 调研笔记只留本地,不提交不推送**(`docs/superpowers/`、`docs/research/` 已在 .gitignore)。README.md 除外。
- 禁止引入 numpy 与 opencv(打包体积约束)。
- 40 色 PALETTE 的值与顺序不得改动;白色格为 index 3。
- 外部项目(ark-pd / ArkPaint)只借鉴思路,不复制代码。
