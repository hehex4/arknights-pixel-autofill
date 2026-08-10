# 项目约定(arknights-pixel-autofill)

面向在本仓库工作的所有 AI 助手(Codex、Claude Code 等)。本文件是本项目约定的唯一事实源。

## 不要把设计类文档推上 GitHub

**设计文档、规格(spec)、实施计划、调研笔记一律只保留在本地工作区,不提交、不推送。**

具体包括但不限于:

- `docs/superpowers/`(brainstorming 产出的 spec、writing-plans 产出的实施计划)
- `docs/research/`(对比调研、技术选型笔记)
- 任何 `*-design.md`、`*-plan.md`、`*-spec.md`

这些路径已写入 `.gitignore`。**提交前请检查暂存区**,确保没有夹带上述文件:

```bash
git status --short
```

例外:`README.md` 是面向用户的公开文档,正常提交推送。

**原因**:这些是内部工作过程记录,不属于对外发布内容,不应出现在公开仓库里。

## 其他约定

- 提交作者 `Zhenyu Wu <wzywuzhenyu@gmail.com>`;AI 参与的提交在信息末尾加 `Co-Authored-By:` 署名行。
- 提交信息用中文,`feat:` / `fix:` / `docs:` / `chore:` 前缀。
- **禁止引入 numpy 与 opencv**:`arknights_pixel_autofill.spec` 显式 `excludes=['numpy']` 以控制打包体积,新算法请用纯 Python + Pillow 实现。
- 40 色 `PALETTE` 的颜色值与顺序不得改动(index+1 即游戏内色号);白色格固定为 index 3。
- 本仓库借鉴过两个外部项目的**思路**,不得复制其代码:
  - [ark-pd](https://gitee.com/guluzz/ark-pd) —— 量化算法思路,PolyForm Noncommercial 1.0.0。
  - [ArkPaint](https://github.com/Eraser2333/ArkPaint) —— ADB 接入思路,**该仓库无 LICENSE 文件**,默认保留所有权利。
  - ADB 线协议本身属 AOSP 公开规范,按规范独立实现无碍。
