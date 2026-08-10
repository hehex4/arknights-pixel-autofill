# 结论:arknights-pixel-autofill 与 ark-pd 代码研究

> 调研日期:2026-08-10。对象:本仓库(`arknights-pixel-autofill` v1.2.0)与 Gitee 仓库 [guluzz/ark-pd](https://gitee.com/guluzz/ark-pd)(奇象巡展拼豆工具 v1.0.0)。两者使用**完全相同的明日方舟 40 色标准色卡**,同为 24×24 像素画工具,但目标、平台与算法路线截然不同。

---

## 1. 目录结构

### 1.1 本仓库(arknights-pixel-autofill)

```text
arknights-pixel-autofill/
├─ arknights_pixel_autofill.py   # 主程序,单文件 2425 行(GUI + 算法 + 自动化)
├─ arknights_pixel_autofill.spec # PyInstaller 打包配置(排除 numpy,UPX 压缩)
├─ arknights_pixel.ico           # 程序图标
├─ requirements.txt              # 运行依赖:Pillow、pywin32
├─ requirements-dev.txt          # 开发依赖:+ PyInstaller
├─ README.md
└─ docs/
   ├─ images/                    # README 引用的截图
   └─ research/                  # 调研文档(本文件所在,本次新建)
```

### 1.2 ark-pd(奇象巡展拼豆工具)

```text
ark-pd/
├─ index.html                # Vite 入口页
├─ package.json              # React + TypeScript + Vite + lucide-react
├─ vite.config.ts
├─ tsconfig*.json
├─ LICENSE                   # PolyForm Noncommercial 1.0.0(非商业许可,见 §5)
├─ README.md
├─ design/                   # 设计概念稿 PNG
├─ output/playwright/        # 页面截图(开发验证产物)
└─ src/
   ├─ main.tsx               # React 入口(10 行)
   ├─ App.tsx                # 状态编排与主工作流(218 行)
   ├─ components.tsx         # 画布/导航/色卡/裁剪/弹窗组件(387 行)
   ├─ image.ts               # 取样、量化、10 种算法预设、PNG 导出(291 行)
   ├─ palette.ts             # 40 色色卡 + CIELAB 转换(65 行)
   └─ styles.css             # PC 与移动端样式(519 行)
```

---

## 2. 本仓库代码理解

单文件 Python 桌面工具,面向 Windows,把图片量化为游戏 40 色 24×24 像素画,然后**模拟鼠标输入自动填进游戏画布**。内部按职责分为六层:

### 2.1 启动层
- Tk 创建前设置 `SetProcessDpiAwarenessContext(PER_MONITOR_AWARE_V2)`,带 shcore/`SetProcessDPIAware` 双重降级,保证跨显示器 DPI 场景坐标正确。
- `SetCurrentProcessExplicitAppUserModelID` 让打包后的 EXE 在任务栏有独立身份。
- `require_admin_before_startup()`:非管理员时用原生 `MessageBoxW` 提示后直接退出,不创建 Tk 主窗口。

### 2.2 输入注入层(`GameMouse`)
最有技术含量的部分之一,解决"Unity 游戏聚焦后忽略普通鼠标事件"的问题:
- 不用 PyAutoGUI 的旧式 `mouse_event`,直接 `SendInput`。
- `move_to()` 采用**三路冗余**:先发相对增量(喂给游戏的 Raw Input)→ `SetCursorPos` 拉回系统光标(抵消 Windows 指针加速)→ 再发 `ABSOLUTE | VIRTUALDESK` 绝对事件(兼容副屏与负坐标)。
- `click()`/`scroll()` 之后再通过 `PostMessage` 向游戏 HWND 补发 `WM_MOUSEMOVE`/`WM_LBUTTONDOWN` 等窗口消息,作为部分客户端锁定系统光标时的备用路径。
- Failsafe:每次 `_send` 前用 `GetAsyncKeyState(VK_F8)` 检查紧急停止,抛 `EmergencyStop` 异常穿透整个绘制循环。

### 2.3 图像算法层
- 40 色 `PALETTE`(RGB 元组),预计算 `OKLAB_PALETTE`。
- 颜色匹配两种:CompuPhase 加权 RGB 距离(经典)与 OKLab 欧氏距离(感知)。
- 缩放取样 4 种(Lanczos/BOX/Bicubic/Nearest)× 适配 3 种(crop/contain/stretch)。
- 可选 Floyd–Steinberg 抖动(逐像素误差扩散)。
- "减少过渡色":先正常量化统计频率,保留出现最多的 16 色,再用这 16 色作为候选集对**原始未抖动图**重新做最近邻映射。
- `import_24_bitmap()`:24×24 PNG 直导,逐像素映射、零重采样。
- 本质是 **per-pixel 最近邻量化**:Pillow 的 resample kernel 负责把信息压到 24×24,之后每个格子独立选色。

### 2.4 窗口定位层
- `find_game_window()`:`EnumWindows` 全枚举后两步筛选——先用标题关键字命中任意窗口确定游戏 PID(官服的中文标题可能挂在隐藏的辅助窗口上,真正的渲染窗口标题是 "Arknights"),再在同 PID 内按 `UnityWndClass` > 可见性 > 客户区面积排序取最优;同时排除自身进程与工具自己的 Tk 窗口。

### 2.5 自动绘制层(`AutoPainter`,后台线程)
- 坐标系:所有 CONFIG 坐标基于 1280×720 基准客户区;非 16:9 时按**居中 16:9 视口**换算。
- **网格自动校准**是可靠性核心:`ImageGrab` 截取客户区灰度图 → `_detect_grid_axis()` 用二阶差分(`|2·p − p₋₂ − p₊₂| + |p₋₂ − p₊₂|`)沿轴打分 → 在预期范围内穷举 (start, pitch) 拟合 25 条等距线,用与比例坐标预期的偏离度做惩罚(避免锁到相位错位的第二条线)→ 逐线在 ±4px 内取峰值微调 → 用中位数得分与残差做置信度检验,不合格则抛异常回退到比例坐标。
- 调色板操作:两阶段——顶部可见 6 行(色号 1–24)填完后滚动到底(24 个独立滚轮刻度,每刻度前重新锚定鼠标位置),再填 5–10 行(色号 25–40);底部行 y 坐标与顶部不同(滚动端点内容整体下移 ~14px),故配置里分两组。
- 填充策略:按颜色分组(减少切换色板次数),可跳过白色格(index 3),每格点击间隔可调(0.04–0.12s)。

### 2.6 GUI 层(Tkinter)
- 无边框自定义标题栏:保留真实 top-level HWND,创建后用 `SetWindowLong` 去掉 `WS_CAPTION`、保留 `WS_THICKFRAME`,兼顾任务栏语义与原生边缘缩放/最大化。
- 响应式:窗口 `<Configure>` 防抖 35ms → 预览画布始终取 24 的整数倍尺寸(保证格子命中测试无误差),窗口高度不足时切换 compact 密度(全局改 ttk style)。
- 手动编辑:左键涂色/拖动连笔(一笔为一个 undo 单元,记录 (col, row, old_index) 列表)、右键吸色、恢复转换结果;色号/行列坐标显示,12 格加粗分界。
- 更新检查:启动 1.4s 后后台线程访问 GitHub Releases API,静默失败,仅版本更高时显示按钮,下载前用户确认。

---

## 3. ark-pd 代码理解

纯前端 Web 应用(React + TypeScript + Vite,无后端、无上传),把图片转成 24×24 **拼豆(perler beads)图纸**供线下制作,附带查看/编辑/导出能力。约 1500 行。

### 3.1 色卡(`palette.ts`)
- 40 个 hex 色,**与本仓库 PALETTE 逐一相同**(如 `#222222`=(34,34,34)、`#D32F36`=(211,47,54)),顺序也一致——同源于游戏调色板。
- 颜色空间用 **CIELAB(D65)**:sRGB → 线性 → XYZ → Lab,距离为 Lab 欧氏距离。与本仓库的 OKLab 思路相同(感知均匀空间),实现不同。

### 3.2 量化算法(`image.ts`)——与本仓库路线的根本差异
本仓库是 "resample 到 24×24 后逐像素最近邻";ark-pd 是 **"每格超采样 → 投票 + 均值混合成本函数 → 邻域平滑迭代"**,一套小型组合优化:

1. **超采样**:把裁剪结果画到 288×288 canvas(每格 12×12 = 144 个样本),alpha 按白底合成。
2. **每格统计**(`pickCellColor`):
   - 空间加权平均色(中心样本权重更高),转 Lab;
   - 144 个样本各自找最近色卡色并**投票**(普通权重 + 中心权重两套票箱);最近邻查找用 **RGB 5-bit 桶缓存**(`(r>>3)<<10 | (g>>3)<<5 | (b>>3)` 为 key),整图共享,把 40 色距离计算从 ~331k 次降到桶数级别;
   - 统计暗像素覆盖率(luminance < 0.28),用于轮廓保护。
3. **成本函数**(`buildCosts`):对每格 × 每色算 cost = 均值 Lab 距离 × `meanWeight` + 票数短缺惩罚 × `voteWeight` + 中心票短缺惩罚 × `centerVoteWeight` ± 暗色 bias(暗覆盖率高时奖励深色、惩罚浅色,反之惩罚深色)。`contrast`/`chroma` 直接在 Lab 空间缩放 L 和 a/b 通道。
4. **边缘感知区域优化**(`edgeAwareRegionOptimization`):先每格取 cost 最小色,再做 `passes` 轮迭代——每格重新在 40 色里选"自身 cost + 平滑项"最小者,平滑项为 `smoothing × exp(−labDistance(邻格源色)/13)`(仅当邻格结果色不同时计入)。**源图相似的相邻格倾向同色(消杂点),真实边缘因源色差大而免罚(保边)**。本质是 Potts 模型 / MRF 能量最小化的 ICM 迭代。
5. **10 种预设** = 同一管线的 9 个参数组合(meanWeight / voteWeight / centerVoteWeight / outlineCoverage / darkBias / smoothing / passes / contrast / chroma),一次性全部生成供用户对比挑选。例如"经典取样"≈纯投票(meanWeight 0.18, voteWeight 2.8, passes 0),"色块平滑"= 高 smoothing 6 passes,"中心采样"= centerVoteWeight 2.9。

### 3.3 UI(`App.tsx` + `components.tsx`)
- `BeadBoard`:576 个 `<button>` 的 CSS Grid;编辑模式用 Pointer Events + `setPointerCapture` 实现拖动连涂(按坐标反算格子索引,去重防重复涂同格)。
- undo/redo:**全矩阵快照栈**(一次手势 = 一个快照,通过 `gestureStartRef` 在 pointerdown 时留底,pointerup 时若有变化才入栈),外加"进入编辑模式时的 baseline"支持一键恢复;与本仓库"逐格增量记录"的 undo 是两种取舍(快照实现简单,24×24 才 576 个 int,成本可忽略)。
- `CropDialog`:拖动 + 缩放选 1:1 区域,输出 720×720 canvas(用 translate/scale 变换绘制,不拉伸原图)。
- `Navigator`:缩略图 + 视口框拖动 pan,滑杆 100–300% zoom(主画布用 CSS `scale()` + 位移变量实现,刻意不用滚轮/双指手势)。
- 下载:canvas 绘制 1200×1200 PNG(`cellSize + 0.5` 防抗锯齿缝隙),网格线可选写入。
- 无路由、无状态库、无测试;组件全部手写,依赖仅 lucide-react 图标。

### 3.4 小问题(如果要参考其代码需留意)
- `VariantDialog` 每次渲染都对 10 个方案各跑一次 `matrixToDataUrl`(建 canvas + toDataURL),未 memo——选中态每变一次就重新生成 10 张图,小图尚可但属浪费。
- `CropDialog` 首帧 `frameRef.current` 未挂载时 `frameSize` 回退 420px 估值,布局与实际可能短暂不一致(确认裁剪时用实际 clientWidth,结果正确)。
- 图片对象 URL 的 revoke 只覆盖 `pendingImage`;裁剪结果转成 dataURL 长期驻留 state,大图会占内存(24×24 场景可接受)。

---

## 4. 对比总结

| 维度 | 本仓库 | ark-pd |
| --- | --- | --- |
| 目标 | 自动把像素画填进游戏(PC 客户端) | 生成拼豆图纸供线下手工制作 |
| 平台 | Windows 桌面(Python/Tkinter/Win32,管理员权限) | 浏览器(React/TS/Vite,纯本地) |
| 代码形态 | 单文件 2425 行,六层职责混排 | 6 个源文件 ~1500 行,组件化 |
| 感知色彩空间 | OKLab(可选,默认 CompuPhase RGB) | CIELAB(唯一路径) |
| 量化思路 | resample 核压缩 → 逐像素最近邻(+可选抖动/减色) | 每格 144 点超采样 → 投票+均值成本 → MRF 邻域平滑 |
| 算法可选项 | 4 缩放 × 2 匹配 × 抖动 × 减色,手动组合切换 | 10 个参数化预设一次全生成,并排预览挑选 |
| 手动编辑 undo | 逐格增量(一笔一组) | 全矩阵快照栈 + redo + baseline 恢复 |
| 独有难点 | SendInput 注入、窗口定位、网格视觉校准、DPI 适配 | 无(不碰系统,复杂度全在算法与交互) |
| 依赖 | Pillow、pywin32 | react、react-dom、lucide-react |
| 许可 | 未声明 LICENSE 文件 | PolyForm Noncommercial 1.0.0 |

**一句话结论**:两个项目共享同一块 40 色画布语义,但本仓库的护城河在 Windows 自动化工程(输入注入 + 视觉校准),ark-pd 的价值在量化算法设计(投票 + 区域优化 + 参数化预设)。二者恰好互补。

### 4.1 值得借鉴到本仓库的点(按性价比排序)

1. **超采样投票量化**:对人物立绘,"每格多点采样 + 投票"比单一 resample kernel 更抗混色,相当于一个更聪明的 BOX。Python 下注意性能:纯循环 24×24×144 样本 ×40 色会慢,应照搬 ark-pd 的 **RGB 5-bit 桶缓存**(本仓库已排除 numpy 以控制打包体积,缓存方案不需要 numpy)。
2. **边缘感知平滑 pass**:在现有量化结果上加 2–4 轮 ICM 迭代(邻格源色相近则倾向同色),能消掉孤立杂点、保住轮廓,与现有"减少过渡色"互补且更细腻。24×24×40 色 × 几轮,纯 Python 也毫无压力。
3. **多方案并排预览**:一次生成若干"预设组合"(现有 4×2 组合 + 新算法)缩略图供点选,比让用户逐个切换 Combobox 再目测好用得多。Tkinter 下可做成一个 Toplevel 网格对话框(参考现有 CropDialog 的结构)。
4. **暗色 bias / 轮廓保护**:统计每格暗像素覆盖率,高覆盖时偏向深色——对线稿类素材的轮廓完整性帮助明显,实现只有几行。
5. 小件:同色高亮(点一格高亮全部同色格,对照游戏内手动补格很实用)、redo 支持(现只有 undo)、全矩阵快照式 undo(可简化现有实现)。

### 4.2 借鉴方式的边界(重要)

ark-pd 采用 **PolyForm Noncommercial 1.0.0**:源码可看、非商业可改可分发,但**直接复制其代码进本仓库会把 NC 条款传导过来**,且本仓库目前没有 LICENSE 文件、还接受夸克网盘等渠道分发,许可状态本就该理清。建议:
- 只借鉴**算法思想**(投票、MRF 平滑、预设参数化都是通用技术,无版权问题),用 Python 独立重新实现;
- 不要逐行移植 `image.ts`;
- 顺手给本仓库补一个明确的 LICENSE(如 MIT 或同类)。

### 4.3 反向输出(本仓库可给 ark-pd 的)

若日后想给 ark-pd 提 issue/PR(非商业前提下):OKLab 距离(比 CIELAB 蓝紫区表现更稳)、Floyd–Steinberg 抖动选项、24×24 PNG 直导。

---

*本文档由代码通读得出:本仓库 `arknights_pixel_autofill.py` 全文,ark-pd `src/` 全部源文件、构建配置与 LICENSE。ark-pd 克隆副本位于会话 scratchpad,未纳入本仓库。*
