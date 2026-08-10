# v1.3.0 MuMu 接入 + 算法重构 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让工具支持连接 MuMu 模拟器自动填色,并用 ark-pd 风格的"超采样投票 + ICM 邻域平滑"管线全面替换旧量化算法(10 个预设入下拉,旧算法控件删除)。

**Architecture:** 新建纯算法模块 `quantize.py`(无 Tk/win32 依赖,可独立测试),主文件删除旧管线并接入新下拉;MuMu 走"顶层标题匹配 → 最大可见子窗口定位 + 拖拽滚动"分支,网格校准/SendInput/F8 全部复用。

**Tech Stack:** Python 3.10+,Pillow,pywin32,tkinter,pytest(新增 dev 依赖),PyInstaller。

## Global Constraints

- 仅支持 Windows;Python 3.10+;运行需管理员权限(现状保持)。
- **禁止引入 numpy**(`arknights_pixel_autofill.spec` 显式 excludes,控制打包体积)。
- **禁止逐行移植 ark-pd 代码**(PolyForm Noncommercial 1.0.0)。只复现方法与行为;所有代码独立手写;预设参数为对照参考行为校准的自定值,最终以 Task 9 实测校准为准。
- UI 文案一律中文;技术名词可保留英文。
- 版本号:`APP_VERSION = "1.3.0"`(Task 8 修改)。
- 提交信息:中文 `feat:`/`docs:`/`test:` 前缀,末尾加 `Co-Authored-By: Claude <noreply@anthropic.com>`。
- 40 色 `PALETTE` 的颜色值与顺序**不得改动**(游戏调色板序即色号)。
- 白色格 index = 3(「跳过白色格」逻辑依赖,不得变)。

## 设计概要(已与用户确认)

1. **算法**:裁切/适配 → LANCZOS 缩放到 288×288(每格 12×12=144 采样)→ 每格统计(中心加权均值 CIELAB + 40 色投票[RGB 5-bit 桶缓存] + 暗部覆盖率)→ 逐格逐色成本(Lab 距离 + 票数短缺惩罚 + 暗色偏置,支持 contrast/chroma 调色)→ ICM 邻域平滑(源色相近的邻格倾向同色)。10 个预设:平衡推荐/轮廓清晰/色块平滑/细节保留/经典取样/柔和过渡/鲜明色彩/高对比度/主色优先/中心采样。
2. **GUI**:删「缩放取样算法」「颜色匹配算法」下拉与「抖动」「减少过渡色」复选框;新增「生成算法」下拉(默认平衡推荐);「缩放方式」保留;每图采样统计缓存,切预设只重跑成本+ICM。位图直导改用 CIELAB 最近邻。
3. **MuMu**:新增「客户端类型」下拉(PC客户端/MuMu模拟器),切换时联动默认关键字(明日方舟↔MuMu);MuMu 模式窗口定位取"命中顶层窗口的最大可见子窗口"客户区;调色板滚动改拖拽手势(拖动+末端停顿杀惯性,重复 3 次到端点);聚焦作用于顶层,截图/坐标/PostMessage 以子窗口为基准。
4. **最后一步(Task 11,执行前需用户再次确认)**:综合优点新增 2 个预设「照片抖动」「精简16色」。
5. **入档不实现**:多方案并排预览;差异填充(截图识别画布现状只补差异格)。

## File Structure

- Create: `quantize.py` — 全部量化算法 + PALETTE + 预设(从主文件迁入 PALETTE/N/flatten_alpha)
- Create: `tests/test_quantize.py` — 算法单元测试(`tests/samples/` 已被 .gitignore 忽略,测试文件本身要入库,见 Task 1 对 .gitignore 的检查)
- Modify: `arknights_pixel_autofill.py` — 删旧管线与控件、接新下拉、MuMu 分支、版本号
- Modify: `requirements-dev.txt` — 加 pytest
- Modify: `README.md` — v1.3.0 说明、MuMu 使用指引、算法致谢(Task 10)
- 不动: `arknights_pixel_autofill.spec`(quantize.py 经 import 自动收集)

---

### Task 1: quantize.py 基础(PALETTE 迁移 + CIELAB + 最近邻)

**Files:**
- Create: `quantize.py`
- Create: `tests/test_quantize.py`
- Modify: `requirements-dev.txt`
- Check: `.gitignore`(确认只忽略 `tests/samples/` 而非整个 `tests/`;若忽略了整个 `tests/`,改为 `tests/samples/`)

**Interfaces:**
- Produces: `N=24`, `SAMPLES=12`, `SUPER=288`, `PALETTE: list[tuple[int,int,int]]`(40 项,顺序同游戏), `PALETTE_LAB`, `rgb_to_lab(rgb)->tuple[float,float,float]`, `lab_distance(a,b)->float`, `nearest_palette_index(rgb)->int`

- [ ] **Step 1: 写失败测试**

`tests/test_quantize.py`:

```python
# -*- coding: utf-8 -*-
import quantize as q


def test_palette_shape():
    assert len(q.PALETTE) == 40
    assert q.PALETTE[3] == (255, 255, 255)   # 白色格 index 3,skip_white 依赖
    assert q.PALETTE[0] == (34, 34, 34)
    assert q.N == 24 and q.SAMPLES == 12 and q.SUPER == 288


def test_lab_endpoints():
    L, a, b = q.rgb_to_lab((255, 255, 255))
    assert abs(L - 100.0) < 0.2 and abs(a) < 0.5 and abs(b) < 0.5
    L0, _, _ = q.rgb_to_lab((0, 0, 0))
    assert abs(L0) < 0.2


def test_nearest_palette_identity():
    for i, color in enumerate(q.PALETTE):
        assert q.nearest_palette_index(color) == i
```

- [ ] **Step 2: 运行确认失败**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: FAIL/ERROR,`ModuleNotFoundError: No module named 'quantize'`

- [ ] **Step 3: 最小实现**

`quantize.py`:

```python
# -*- coding: utf-8 -*-
"""24×24 游戏调色板量化管线。

方法思路参考 ark-pd(https://gitee.com/guluzz/ark-pd,PolyForm NC 1.0.0):
超采样 → 每格统计 → 成本函数 → ICM 邻域平滑。本模块为独立重写,
不含其代码;预设参数为对照参考行为自行校准的取值。
仅依赖 Pillow;刻意不使用 numpy(见 .spec 的 excludes)。
"""

import math

N = 24          # 画布格数
SAMPLES = 12    # 每格每轴采样点数
SUPER = N * SAMPLES  # 超采样图边长 288

# 40 色游戏调色板,顺序与游戏一致(index+1 即界面色号)。
PALETTE = [
    (34, 34, 34),     (180, 180, 180), (234, 231, 223), (255, 255, 255),
    (211, 47, 54),    (156, 10, 0),     (214, 12, 74),   (230, 150, 141),
    (254, 152, 117),  (247, 208, 192),  (252, 239, 234), (251, 246, 232),
    (220, 210, 200),  (226, 206, 171),  (213, 99, 34),   (212, 140, 66),
    (242, 153, 0),    (249, 201, 51),   (252, 228, 153), (179, 180, 122),
    (194, 218, 114),  (108, 110, 0),    (170, 139, 82),  (169, 143, 116),
    (170, 146, 40),   (63, 43, 18),     (116, 73, 31),   (83, 70, 88),
    (42, 36, 70),     (57, 69, 153),    (90, 69, 157),   (186, 163, 215),
    (182, 188, 223),  (169, 172, 190),  (99, 171, 185),  (180, 210, 220),
    (145, 216, 230),  (71, 174, 160),   (182, 211, 200), (39, 56, 100),
]


def _srgb_to_linear(value):
    value = value / 255.0
    return value / 12.92 if value <= 0.04045 else ((value + 0.055) / 1.055) ** 2.4


def rgb_to_lab(rgb):
    """sRGB(0-255) → CIELAB(D65)。"""
    r, g, b = (_srgb_to_linear(c) for c in rgb)
    x = (0.4124 * r + 0.3576 * g + 0.1805 * b) / 0.95047
    y = 0.2126 * r + 0.7152 * g + 0.0722 * b
    z = (0.0193 * r + 0.1192 * g + 0.9505 * b) / 1.08883

    def f(t):
        return t ** (1.0 / 3.0) if t > 0.008856 else 7.787 * t + 16.0 / 116.0

    fx, fy, fz = f(x), f(y), f(z)
    return 116.0 * fy - 16.0, 500.0 * (fx - fy), 200.0 * (fy - fz)


PALETTE_LAB = [rgb_to_lab(color) for color in PALETTE]


def lab_distance(a, b):
    return math.sqrt((a[0] - b[0]) ** 2 + (a[1] - b[1]) ** 2 + (a[2] - b[2]) ** 2)


def nearest_palette_index(rgb):
    lab = rgb_to_lab(rgb)
    best = 0
    best_d = float("inf")
    for i, plab in enumerate(PALETTE_LAB):
        d = (lab[0] - plab[0]) ** 2 + (lab[1] - plab[1]) ** 2 + (lab[2] - plab[2]) ** 2
        if d < best_d:
            best_d = d
            best = i
    return best
```

`requirements-dev.txt` 追加一行:

```text
pytest>=8.0
```

- [ ] **Step 4: 运行确认通过**

Run: `py -m pip install pytest`(若未装)然后 `py -m pytest tests/test_quantize.py -v`
Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add quantize.py tests/test_quantize.py requirements-dev.txt .gitignore
git commit -m "feat: 新建quantize模块与CIELAB色彩匹配基础"
```

---

### Task 2: 图片适配 prepare_source

**Files:**
- Modify: `quantize.py`
- Test: `tests/test_quantize.py`

**Interfaces:**
- Produces: `flatten_alpha(img)->Image`(RGBA 白底合成), `prepare_source(img, mode)->Image`(输出恒为 SUPER×SUPER RGB;mode ∈ "crop"/"contain"/"stretch",语义与旧版 resize_to_24 相同)

- [ ] **Step 1: 写失败测试**(追加到 `tests/test_quantize.py`)

```python
from PIL import Image


def test_prepare_source_sizes():
    img = Image.new("RGB", (100, 50), (255, 0, 0))
    for mode in ("crop", "contain", "stretch"):
        out = q.prepare_source(img, mode)
        assert out.size == (q.SUPER, q.SUPER)
        assert out.mode == "RGB"


def test_contain_pads_white():
    img = Image.new("RGB", (100, 50), (0, 0, 0))
    out = q.prepare_source(img, "contain")
    assert out.getpixel((q.SUPER // 2, 2)) == (255, 255, 255)   # 上方留白
    assert out.getpixel((q.SUPER // 2, q.SUPER // 2)) == (0, 0, 0)


def test_flatten_alpha_composites_white():
    img = Image.new("RGBA", (8, 8), (0, 0, 0, 0))
    assert q.flatten_alpha(img).getpixel((0, 0)) == (255, 255, 255)
```

- [ ] **Step 2: 运行确认失败**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 新增 3 条 FAIL,`AttributeError: ... 'prepare_source'`

- [ ] **Step 3: 实现**(追加到 `quantize.py`,并在文件头 import 区加 `from PIL import Image, ImageOps`)

```python
def flatten_alpha(img):
    img = img.convert("RGBA")
    bg = Image.new("RGBA", img.size, (255, 255, 255, 255))
    bg.alpha_composite(img)
    return bg.convert("RGB")


def prepare_source(img, mode="crop"):
    """把任意图片适配为 SUPER×SUPER 的超采样输入。"""
    img = flatten_alpha(img)
    method = Image.Resampling.LANCZOS
    if mode == "stretch":
        return img.resize((SUPER, SUPER), method)
    if mode == "contain":
        out = Image.new("RGB", (SUPER, SUPER), (255, 255, 255))
        tmp = img.copy()
        tmp.thumbnail((SUPER, SUPER), method)
        out.paste(tmp, ((SUPER - tmp.width) // 2, (SUPER - tmp.height) // 2))
        return out
    return ImageOps.fit(img, (SUPER, SUPER), method=method, centering=(0.5, 0.5))
```

- [ ] **Step 4: 运行确认通过**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add quantize.py tests/test_quantize.py
git commit -m "feat: 新增288超采样图片适配"
```

---

### Task 3: 每格统计 compute_cell_stats

**Files:**
- Modify: `quantize.py`
- Test: `tests/test_quantize.py`

**Interfaces:**
- Produces: `CellStats`(dataclass:`mean_rgb`, `mean_lab`, `votes: list[float]`, `center_votes: list[float]`, `total_weight: float`, `center_total: float`, `dark_coverage: float`), `compute_cell_stats(img)->list[CellStats]`(len N*N,行优先;入参必须是 SUPER×SUPER RGB,否则 ValueError)
- 常量:`DARK_LUMA = 0.30`(暗像素亮度阈值)

- [ ] **Step 1: 写失败测试**(追加)

```python
def _flat(color):
    return Image.new("RGB", (q.SUPER, q.SUPER), color)


def test_cell_stats_flat_color():
    stats = q.compute_cell_stats(_flat((211, 47, 54)))
    assert len(stats) == q.N * q.N
    s = stats[0]
    assert all(abs(m - e) < 1.0 for m, e in zip(s.mean_rgb, (211, 47, 54)))
    top = max(range(len(q.PALETTE)), key=s.votes.__getitem__)
    assert top == 4                      # (211,47,54) 为色号5(index 4)
    assert s.votes[4] > 0.99 * s.total_weight


def test_cell_stats_dark_coverage():
    assert q.compute_cell_stats(_flat((0, 0, 0)))[0].dark_coverage == 1.0
    assert q.compute_cell_stats(_flat((255, 255, 255)))[0].dark_coverage == 0.0


def test_cell_stats_rejects_wrong_size():
    import pytest
    with pytest.raises(ValueError):
        q.compute_cell_stats(Image.new("RGB", (24, 24)))
```

- [ ] **Step 2: 运行确认失败**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 新增 3 条 FAIL,`AttributeError: ... 'compute_cell_stats'`

- [ ] **Step 3: 实现**(追加;文件头加 `from dataclasses import dataclass`)

```python
DARK_LUMA = 0.30   # 亮度低于此值(0..1)记为暗像素

# 每格 12×12 采样点的空间权重(中心更高),模块级预计算一次。
_SPATIAL_W = []
_CENTER_W = []
for _sy in range(SAMPLES):
    for _sx in range(SAMPLES):
        _d = min(1.0, math.hypot(_sx - (SAMPLES - 1) / 2.0, _sy - (SAMPLES - 1) / 2.0) / 7.0)
        _SPATIAL_W.append(1.0 + 0.4 * (1.0 - _d))
        _CENTER_W.append(0.2 + 1.8 * (1.0 - _d) ** 2)


@dataclass
class CellStats:
    mean_rgb: tuple
    mean_lab: tuple
    votes: list
    center_votes: list
    total_weight: float
    center_total: float
    dark_coverage: float


def compute_cell_stats(img):
    """对 SUPER×SUPER 输入逐格统计;每图只需调用一次,可缓存复用。"""
    if img.size != (SUPER, SUPER) or img.mode != "RGB":
        raise ValueError(f"需要 {SUPER}×{SUPER} RGB 输入,当前 {img.size} {img.mode}")
    px = img.load()
    bucket_cache = {}
    stats = []
    for cy in range(N):
        for cx in range(N):
            rs = gs = bs = tw = ctw = 0.0
            dark = 0
            votes = [0.0] * len(PALETTE)
            cvotes = [0.0] * len(PALETTE)
            wi = 0
            for sy in range(SAMPLES):
                yy = cy * SAMPLES + sy
                for sx in range(SAMPLES):
                    r, g, b = px[cx * SAMPLES + sx, yy]
                    w = _SPATIAL_W[wi]
                    cw = _CENTER_W[wi]
                    wi += 1
                    rs += r * w
                    gs += g * w
                    bs += b * w
                    tw += w
                    ctw += cw
                    if (0.2126 * r + 0.7152 * g + 0.0722 * b) / 255.0 < DARK_LUMA:
                        dark += 1
                    # RGB 每通道取高5位分桶,同桶共享最近邻结果,避免逐点找 40 色。
                    key = (r >> 3, g >> 3, b >> 3)
                    idx = bucket_cache.get(key)
                    if idx is None:
                        idx = nearest_palette_index(
                            ((r >> 3 << 3) + 4, (g >> 3 << 3) + 4, (b >> 3 << 3) + 4)
                        )
                        bucket_cache[key] = idx
                    votes[idx] += w
                    cvotes[idx] += cw
            mean_rgb = (rs / tw, gs / tw, bs / tw)
            stats.append(CellStats(
                mean_rgb, rgb_to_lab(mean_rgb), votes, cvotes, tw, ctw,
                dark / (SAMPLES * SAMPLES),
            ))
    return stats
```

- [ ] **Step 4: 运行确认通过**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 9 passed

- [ ] **Step 5: Commit**

```bash
git add quantize.py tests/test_quantize.py
git commit -m "feat: 新增每格超采样统计与色卡投票"
```

---

### Task 4: 成本函数 + 预设定义

**Files:**
- Modify: `quantize.py`
- Test: `tests/test_quantize.py`

**Interfaces:**
- Produces: `Preset`(frozen dataclass:`mean_w, vote_w, center_w, outline_cov, dark_bias, smooth, passes, contrast=1.0, chroma=1.0`), `PRESETS: dict[str, Preset]`(10 项,插入顺序即下拉顺序,首项"平衡推荐"), `build_costs(stats, preset)->list[list[float]]`(N*N × 40), `DARK_L = 40.0`

- [ ] **Step 1: 写失败测试**(追加)

```python
def test_presets_catalog():
    names = list(q.PRESETS)
    assert names[0] == "平衡推荐"
    assert len(names) == 10
    assert "经典取样" in names and "中心采样" in names


def test_costs_flat_color_prefers_exact():
    stats = q.compute_cell_stats(_flat((252, 239, 234)))    # 色号11(index 10)
    for name, preset in q.PRESETS.items():
        costs = q.build_costs(stats, preset)
        assert len(costs) == q.N * q.N and len(costs[0]) == 40
        assert min(range(40), key=costs[0].__getitem__) == 10, name


def test_costs_dark_cell_biased_dark():
    stats = q.compute_cell_stats(_flat((34, 34, 34)))       # 全暗格
    preset = q.PRESETS["轮廓清晰"]
    costs = q.build_costs(stats, preset)
    # 暗格上任何浅色(白)的成本应显著高于最深色
    assert costs[0][3] > costs[0][0]
```

- [ ] **Step 2: 运行确认失败**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 新增 3 条 FAIL,`AttributeError: ... 'PRESETS'`

- [ ] **Step 3: 实现**(追加)

```python
DARK_L = 40.0   # CIELAB L 低于此值的色卡色视为"深色"


@dataclass(frozen=True)
class Preset:
    mean_w: float       # 均值 Lab 距离权重
    vote_w: float       # 票数短缺惩罚权重
    center_w: float     # 中心票短缺惩罚权重
    outline_cov: float  # 暗部覆盖率≥此值时启用暗色偏置
    dark_bias: float    # 暗色偏置强度
    smooth: float       # ICM 平滑强度
    passes: int         # ICM 迭代轮数
    contrast: float = 1.0
    chroma: float = 1.0


# 初始参数对照参考实现(ark-pd)行为设定,Task 9 以两边实图对比结果校准。
PRESETS = {
    "平衡推荐": Preset(1.00, 1.35, 0.25, 0.30, 0.72, 0.42, 4),
    "轮廓清晰": Preset(0.88, 1.42, 0.40, 0.18, 1.18, 0.24, 3, contrast=1.08),
    "色块平滑": Preset(1.08, 0.72, 0.08, 0.34, 0.48, 0.92, 6, contrast=0.96, chroma=0.94),
    "细节保留": Preset(0.82, 1.18, 0.90, 0.26, 0.70, 0.12, 2, contrast=1.03, chroma=1.05),
    "经典取样": Preset(0.18, 2.80, 0.00, 0.30, 0.32, 0.00, 0),
    "柔和过渡": Preset(1.22, 0.78, 0.15, 0.38, 0.30, 0.58, 5, contrast=0.78, chroma=0.82),
    "鲜明色彩": Preset(1.12, 1.08, 0.20, 0.30, 0.60, 0.32, 3, contrast=1.04, chroma=1.38),
    "高对比度": Preset(1.08, 1.12, 0.28, 0.22, 0.95, 0.28, 3, contrast=1.38, chroma=1.08),
    "主色优先": Preset(0.38, 3.20, 0.12, 0.28, 0.48, 0.20, 2),
    "中心采样": Preset(0.50, 0.55, 2.90, 0.27, 0.58, 0.08, 1),
}


def build_costs(stats, preset):
    """逐格 × 逐色成本;越小越优。"""
    costs = []
    for s in stats:
        target = (
            50.0 + (s.mean_lab[0] - 50.0) * preset.contrast,
            s.mean_lab[1] * preset.chroma,
            s.mean_lab[2] * preset.chroma,
        )
        dark_cell = s.dark_coverage >= preset.outline_cov
        light_cell = s.dark_coverage < 0.12
        row = []
        for i, plab in enumerate(PALETTE_LAB):
            cost = lab_distance(target, plab) / 18.0 * preset.mean_w
            cost += max(0.0, 0.58 - s.votes[i] / s.total_weight) * preset.vote_w
            cost += max(0.0, 0.52 - s.center_votes[i] / s.center_total) * preset.center_w
            if dark_cell:
                cost += -preset.dark_bias if plab[0] < DARK_L else preset.dark_bias
            elif light_cell and plab[0] < DARK_L:
                cost += 0.38    # 亮格轻罚深色,防止杂点变黑点
            row.append(cost)
        costs.append(row)
    return costs
```

- [ ] **Step 4: 运行确认通过**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 12 passed

- [ ] **Step 5: Commit**

```bash
git add quantize.py tests/test_quantize.py
git commit -m "feat: 新增10个生成算法预设与成本函数"
```

---

### Task 5: ICM 平滑 + 对外 API + 位图直导

**Files:**
- Modify: `quantize.py`
- Test: `tests/test_quantize.py`

**Interfaces:**
- Produces: `icm_smooth(stats, costs, preset)->list[int]`(len N*N), `quantize_matrix(stats, preset_name)->list[list[int]]`(24 行 × 24 列,值 0..39;主 GUI 唯一入口), `import_24_bitmap(img)->list[list[int]]`(尺寸不符抛 ValueError,文案含"位图尺寸必须为 24×24")

- [ ] **Step 1: 写失败测试**(追加)

```python
def test_quantize_flat_single_color():
    stats = q.compute_cell_stats(_flat((252, 239, 234)))
    for name in q.PRESETS:
        matrix = q.quantize_matrix(stats, name)
        assert {v for row in matrix for v in row} == {10}, name


def test_quantize_two_regions_edge_kept():
    img = Image.new("RGB", (q.SUPER, q.SUPER), (255, 255, 255))
    for yy in range(q.SUPER):
        for xx in range(q.SUPER // 2):
            img.putpixel((xx, yy), (34, 34, 34))
    stats = q.compute_cell_stats(img)
    matrix = q.quantize_matrix(stats, "色块平滑")   # 最强平滑也不得抹掉真实边缘
    assert matrix[12][0] == 0 and matrix[12][11] == 0
    assert matrix[12][12] == 3 and matrix[12][23] == 3


def test_quantize_deterministic_and_valid():
    img = Image.linear_gradient("L").resize((q.SUPER, q.SUPER)).convert("RGB")
    stats = q.compute_cell_stats(img)
    for name in q.PRESETS:
        m1 = q.quantize_matrix(stats, name)
        m2 = q.quantize_matrix(stats, name)
        assert m1 == m2
        assert all(0 <= v < 40 for row in m1 for v in row)


def test_import_24_bitmap():
    import pytest
    bmp = Image.new("RGB", (q.N, q.N), q.PALETTE[7])
    matrix = q.import_24_bitmap(bmp)
    assert len(matrix) == q.N and matrix[0][0] == 7
    with pytest.raises(ValueError):
        q.import_24_bitmap(Image.new("RGB", (23, 24)))
```

- [ ] **Step 2: 运行确认失败**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 新增 4 条 FAIL,`AttributeError: ... 'quantize_matrix'`

- [ ] **Step 3: 实现**(追加)

```python
def _argmin(row):
    return min(range(len(row)), key=row.__getitem__)


def icm_smooth(stats, costs, preset):
    """对初始逐格最优解做邻域平滑迭代:源图相似的相邻格倾向同色,
    真实边缘(源色差大)平滑项趋零而得以保留。"""
    result = [_argmin(row) for row in costs]
    if preset.passes <= 0 or preset.smooth <= 0.0:
        return result

    neighbor_affinity = []
    for y in range(N):
        for x in range(N):
            i = y * N + x
            entries = []
            for j in (i - N if y > 0 else -1, i - 1 if x > 0 else -1,
                      i + 1 if x < N - 1 else -1, i + N if y < N - 1 else -1):
                if j >= 0:
                    weight = preset.smooth * math.exp(
                        -lab_distance(stats[i].mean_lab, stats[j].mean_lab) / 13.0
                    )
                    entries.append((j, weight))
            neighbor_affinity.append(entries)

    for _ in range(preset.passes):
        for i in range(N * N):
            row = costs[i]
            entries = neighbor_affinity[i]
            best_color = result[i]
            best_energy = float("inf")
            for color in range(len(PALETTE)):
                energy = row[color]
                for j, weight in entries:
                    if result[j] != color:
                        energy += weight
                if energy < best_energy:
                    best_energy = energy
                    best_color = color
            result[i] = best_color
    return result


def quantize_matrix(stats, preset_name):
    """主入口:每格统计 + 预设名 → 24×24 色卡索引矩阵。"""
    preset = PRESETS[preset_name]
    flat = icm_smooth(stats, build_costs(stats, preset), preset)
    return [flat[y * N:(y + 1) * N] for y in range(N)]


def import_24_bitmap(img):
    """24×24 位图逐像素直导,零重采样。"""
    if img.size != (N, N):
        raise ValueError(f"位图尺寸必须为 {N}×{N},当前为 {img.width}×{img.height}。")
    src = flatten_alpha(img)
    return [
        [nearest_palette_index(src.getpixel((x, y))) for x in range(N)]
        for y in range(N)
    ]
```

- [ ] **Step 4: 运行确认通过**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 16 passed

- [ ] **Step 5: Commit**

```bash
git add quantize.py tests/test_quantize.py
git commit -m "feat: 新增ICM邻域平滑与量化主入口"
```

---

### Task 6: 主程序接入新算法(删旧管线与旧控件)

**Files:**
- Modify: `arknights_pixel_autofill.py`

**Interfaces:**
- Consumes: `quantize.py` 的 `N, PALETTE, PRESETS, prepare_source, compute_cell_stats, quantize_matrix, import_24_bitmap, flatten_alpha`
- Produces: `App.algorithm_mode: tk.StringVar`(值 ∈ PRESETS 键), `App.algo_box: ttk.Combobox`;`App.reprocess()` 走新管线并缓存 stats

- [ ] **Step 1: 删除旧算法层**

在 `arknights_pixel_autofill.py` 中删除以下定义(整段删除):`N`、`PALETTE`、`color_distance`、`_srgb_to_linear`、`rgb_to_oklab`、`OKLAB_PALETTE`、`nearest_palette_index`、`flatten_alpha`、`RESAMPLE_METHODS`、`resize_to_24`、`quantize_image`、`import_24_bitmap`。

在 `from pathlib import Path` 之后加:

```python
from quantize import (
    N,
    PALETTE,
    PRESETS,
    compute_cell_stats,
    flatten_alpha,
    import_24_bitmap,
    prepare_source,
    quantize_matrix,
)
```

- [ ] **Step 2: 删除旧控件与状态**

`App.__init__` 中删除:`self.resample_mode`、`self.color_match_mode`、`self.dither`、`self.reduce_transitions` 四个变量;新增:

```python
self.algorithm_mode = tk.StringVar(value=next(iter(PRESETS)))
self._cell_stats = None       # 当前图的每格统计缓存
self._stats_key = None        # (epoch, crop_box, fit_mode)
self._image_epoch = 0         # 每次载入新图自增,防缓存串图
```

`_build_ui` 中删除「缩放取样算法」「颜色匹配算法」两组 Label+Combobox 和 `self.dither_check`、`self.transition_check` 两个 Checkbutton;在「缩放方式」组后新增:

```python
ttk.Label(left, text="生成算法", style="Muted.TLabel").pack(anchor="w")
self.algo_box = ttk.Combobox(
    left,
    textvariable=self.algorithm_mode,
    state="readonly",
    values=tuple(PRESETS),
    style="Dark.TCombobox",
)
self.algo_box.pack(fill="x", pady=(2, 3))
self.algo_box.bind("<<ComboboxSelected>>", lambda _event: self.reprocess())
```

删除方法:`_sync_algorithm_controls`、`_perceptual_enabled`、`_toggle_reduce_transitions`。

- [ ] **Step 3: 重写 reprocess 与两个导入口**

`reprocess` 整体替换为:

```python
def reprocess(self):
    if self.source_image is None:
        return
    if self.direct_bitmap_mode:
        self.matrix = import_24_bitmap(self.source_image)
        self.original_matrix = [row[:] for row in self.matrix]
        self.edit_history.clear()
        self._render_preview()
        used = len({value for row in self.matrix for value in row})
        self._update_matrix_info(
            f"位图已逐像素导入:24×24,无尺寸重采样,共使用 {used} 种游戏颜色。"
        )
        return
    source = self.source_image
    if self.crop_box is not None:
        source = source.crop(self.crop_box)
    key = (self._image_epoch, self.crop_box, self.fit_mode.get())
    if self._stats_key != key:
        self._cell_stats = compute_cell_stats(prepare_source(source, self.fit_mode.get()))
        self._stats_key = key
    self.matrix = quantize_matrix(self._cell_stats, self.algorithm_mode.get())
    self.original_matrix = [row[:] for row in self.matrix]
    self.edit_history.clear()
    self._render_preview()
    used = len({v for row in self.matrix for v in row})
    self._update_matrix_info(
        f"转换完成:{self.algorithm_mode.get()},共使用 {used} 种游戏颜色;可继续修改。"
    )
```

`open_image` 中:删除对 `resample_box`/`match_box`/`dither_check`/`transition_check`/`_sync_algorithm_controls` 的引用;在 `self.direct_bitmap_mode = False` 后加 `self._image_epoch += 1`;保留 `self.mode_box.configure(state="readonly")` 并新增 `self.algo_box.configure(state="readonly")`。

`open_24_bitmap` 中:`import_24_bitmap(bitmap, perceptual=...)` 改为 `import_24_bitmap(bitmap)`;删除对已删控件的 state 操作;新增 `self.algo_box.configure(state="disabled")` 与 `self.mode_box.configure(state="disabled")`(原有的 mode_box disable 保留);同样 `self._image_epoch += 1`。

- [ ] **Step 4: 手动冒烟验证**

Run: `py arknights_pixel_autofill.py`(管理员 PowerShell)
清单:载入一张普通图 → 依次切换 10 个「生成算法」预设,预览均即时刷新且互有差异;切「缩放方式」三档正常;裁切后重新出图;导入 24×24 PNG 位图正常且两个下拉变灰;手动涂色/吸色/撤销/恢复正常;左栏不再出现旧的 4 个算法控件。
Expected: 全部通过,无 console 报错

- [ ] **Step 5: 回归单测 + Commit**

Run: `py -m pytest tests/ -v` → 16 passed

```bash
git add arknights_pixel_autofill.py
git commit -m "feat: 生成算法下拉替换旧量化管线"
```

---

### Task 7: MuMu 基础件(拖拽输入 + 子窗口定位 + 客户端类型下拉)

**Files:**
- Modify: `arknights_pixel_autofill.py`

**Interfaces:**
- Produces: `GameMouse.press(x, y)`, `GameMouse.release()`, `GameMouse.drag(x0, y0, x1, y1, steps=12, duration=0.28, settle=0.18)`;`find_mumu_window(keyword)->(top_hwnd, render_hwnd, title) | (None, None, None)`;`App.client_type: tk.StringVar`(值 ∈ "PC客户端"/"MuMu模拟器"), `DEFAULT_KEYWORDS: dict`

- [ ] **Step 1: GameMouse 增加触摸式拖拽**(加在 `scroll` 方法之后)

```python
def press(self, x, y):
    self.move_to(x, y)
    self._send(self.LEFTDOWN)
    self._post_button(win32con.WM_LBUTTONDOWN, win32con.MK_LBUTTON)

def release(self):
    self._send(self.LEFTUP)
    self._post_button(win32con.WM_LBUTTONUP, 0)

def _post_button(self, message, wparam):
    hwnd = self.target_hwnd
    if hwnd and self.last_position and win32gui.IsWindow(hwnd):
        try:
            cx, cy = win32gui.ScreenToClient(hwnd, self.last_position)
            win32gui.PostMessage(hwnd, message, wparam, self._make_lparam(cx, cy))
        except Exception:
            pass

def drag(self, x0, y0, x1, y1, steps=12, duration=0.28, settle=0.18):
    """按下-匀速拖动-末端停顿-抬起。停顿用于消除模拟器触摸惯性滚动。"""
    self.press(x0, y0)
    for i in range(1, steps + 1):
        t = i / steps
        self.move_to(round(x0 + (x1 - x0) * t), round(y0 + (y1 - y0) * t))
        time.sleep(max(0.0, duration / steps - self.pause))
    time.sleep(settle)
    self.release()
```

同时重构 `click()`:其尾部 PostMessage 三连中的 DOWN/UP 两条改用 `self._post_button`(行为不变,消重复)。

- [ ] **Step 2: 新增 find_mumu_window**(加在 `find_game_window` 之后)

```python
def find_mumu_window(keyword):
    """MuMu 模式:命中可见顶层窗口后,取其最大可见子窗口作为渲染区。
    不依赖具体窗口类名,模拟器版本升级换类名不受影响。"""
    own_pid = os.getpid()
    needle = keyword.strip().lower()
    candidates = []

    def enum_cb(hwnd, _):
        try:
            if not win32gui.IsWindowVisible(hwnd):
                return True
            title = win32gui.GetWindowText(hwnd)
            _, pid = win32process.GetWindowThreadProcessId(hwnd)
            if pid == own_pid or not needle or needle not in title.lower():
                return True
            left, top, right, bottom = win32gui.GetClientRect(hwnd)
            area = max(0, right - left) * max(0, bottom - top)
            if area > 0:
                candidates.append((area, hwnd, title))
        except Exception:
            pass
        return True

    win32gui.EnumWindows(enum_cb, None)
    if not candidates:
        return None, None, None
    candidates.sort(key=lambda item: item[0], reverse=True)
    _, top_hwnd, title = candidates[0]

    children = []

    def child_cb(hwnd, _):
        try:
            if win32gui.IsWindowVisible(hwnd):
                left, top, right, bottom = win32gui.GetClientRect(hwnd)
                children.append(((right - left) * (bottom - top), hwnd))
        except Exception:
            pass
        return True

    try:
        win32gui.EnumChildWindows(top_hwnd, child_cb, None)
    except Exception:
        pass
    render_hwnd = top_hwnd
    if children:
        children.sort(key=lambda item: item[0], reverse=True)
        if children[0][0] > 0:
            render_hwnd = children[0][1]
    return top_hwnd, render_hwnd, title
```

- [ ] **Step 3: 客户端类型下拉 + 关键字联动**

模块级(CONFIG 附近)加:

```python
DEFAULT_KEYWORDS = {"PC客户端": "明日方舟", "MuMu模拟器": "MuMu"}
```

`App.__init__` 变量区加 `self.client_type = tk.StringVar(value="PC客户端")`。

`_build_ui` 中「02 自动化设置」标题之后、「游戏窗口标题关键字」之前插入:

```python
ttk.Label(left, text="客户端类型", style="Muted.TLabel").pack(anchor="w", pady=(4, 0))
client_box = ttk.Combobox(
    left,
    textvariable=self.client_type,
    state="readonly",
    values=tuple(DEFAULT_KEYWORDS),
    style="Dark.TCombobox",
)
client_box.pack(fill="x", pady=(2, 6))
client_box.bind("<<ComboboxSelected>>", self._on_client_type_change)
```

`App` 中新增方法:

```python
def _on_client_type_change(self, _event=None):
    selected = self.client_type.get()
    other = "MuMu模拟器" if selected == "PC客户端" else "PC客户端"
    # 只在关键字仍是另一模式的默认值时联动,不覆盖用户手改内容。
    if self.window_keyword.get().strip() == DEFAULT_KEYWORDS[other]:
        self.window_keyword.set(DEFAULT_KEYWORDS[selected])
```

- [ ] **Step 4: 手动验证**

Run: `py arknights_pixel_autofill.py`
清单:「客户端类型」下拉出现在关键字上方;切到 MuMu 后关键字自动变 `MuMu`,切回变 `明日方舟`;手动把关键字改成 `雷电` 后切换类型不再被覆盖。
Expected: 全部通过

- [ ] **Step 5: Commit**

```bash
git add arknights_pixel_autofill.py
git commit -m "feat: 新增客户端类型选择与MuMu窗口定位基础件"
```

---

### Task 8: AutoPainter MuMu 分支 + 版本号

**Files:**
- Modify: `arknights_pixel_autofill.py`

**Interfaces:**
- Consumes: Task 7 的 `find_mumu_window`, `GameMouse.drag`, `App.client_type`
- Produces: `AutoPainter.paint(matrix, keyword, click_delay, skip_white, client_type)`;`AutoPainter._scroll_palette(..., use_drag)` 与 `_scroll_palette_drag(...)`

- [ ] **Step 1: 拖拽滚动实现**(`_scroll_palette` 增加分支;新方法加在其后)

`_scroll_palette` 签名改为 `def _scroll_palette(self, origin_x, origin_y, w, h, to_bottom, use_drag=False):`,方法体开头加:

```python
if use_drag:
    self._scroll_palette_drag(origin_x, origin_y, w, h, to_bottom)
    return
```

新方法:

```python
def _scroll_palette_drag(self, origin_x, origin_y, w, h, to_bottom):
    """模拟器触摸语义:在调色板内拖拽滑动。内容跟随手指,
    去底部=向上拖;重复 3 次保证到达端点,期间误触色块无害
    (随后必然显式点击目标颜色)。"""
    anchor_x = CONFIG["palette_scroll_anchor"][0]
    x_hi, y_hi = self._scaled(anchor_x, 240.0, w, h)
    x_lo, y_lo = self._scaled(anchor_x, 640.0, w, h)
    if to_bottom:
        start, end = (x_lo, y_lo), (x_hi, y_hi)
    else:
        start, end = (x_hi, y_hi), (x_lo, y_lo)
    for _ in range(3):
        if self.stop_event.is_set():
            return
        game_mouse.drag(
            origin_x + start[0], origin_y + start[1],
            origin_x + end[0], origin_y + end[1],
        )
        time.sleep(0.30)
    time.sleep(0.35)
```

- [ ] **Step 2: paint() 接入客户端类型**

签名改为 `def paint(self, matrix, keyword, click_delay, skip_white, client_type="PC客户端"):`,方法体开头加 `is_mumu = client_type == "MuMu模拟器"`。

窗口获取循环改为双分支(保持原有 5 次重试结构):

```python
for _ in range(5):
    if is_mumu:
        top_hwnd, render_hwnd, candidate_title = find_mumu_window(keyword)
        if not top_hwnd:
            time.sleep(0.2)
            continue
        if not focus_window(top_hwnd):
            time.sleep(0.2)
            continue
        time.sleep(0.45)
        top_hwnd, render_hwnd, candidate_title = find_mumu_window(keyword)
        if not top_hwnd:
            time.sleep(0.2)
            continue
        candidate = render_hwnd
    else:
        candidate, candidate_title = find_game_window(keyword)
        if not candidate:
            time.sleep(0.2)
            continue
        if not focus_window(candidate):
            time.sleep(0.2)
            continue
        time.sleep(0.45)
        refreshed, refreshed_title = find_game_window(keyword)
        if refreshed and refreshed != candidate:
            candidate, candidate_title = refreshed, refreshed_title
            focus_window(candidate)
            time.sleep(0.35)
    try:
        ox, oy, w, h = get_client_info(candidate)
    except Exception:
        time.sleep(0.2)
        continue
    hwnd, title = candidate, candidate_title
    if w >= 800 and h >= 450:
        break
```

未找到时的报错文案区分模式:MuMu 分支提示"未找到 MuMu 窗口。请确认模拟器已启动,且关键字与模拟器标题一致。"

两处 `self._scroll_palette(ox, oy, w, h, to_bottom=...)` 调用补 `use_drag=is_mumu`。

- [ ] **Step 3: start_paint 传参 + 版本号**

`start_paint` 中线程参数改为:

```python
t = threading.Thread(
    target=self.painter.paint,
    args=(matrix_copy, keyword, delay, skip_white, self.client_type.get()),
    daemon=True,
)
```

`APP_VERSION = "1.2.0"` 改为 `"1.3.0"`。

- [ ] **Step 4: PC 模式回归(手动)**

Run: `py arknights_pixel_autofill.py`,PC 客户端打开游戏 24×24 画板,用小图(如纯 2 色 24×24 位图)跑一次自动填充。
Expected: 与 v1.2.0 行为一致——找到窗口、自动校准、两阶段滚动填色、F8 可停。

- [ ] **Step 5: Commit**

```bash
git add arknights_pixel_autofill.py
git commit -m "feat: 自动填色支持MuMu模拟器分支"
```

---

### Task 9: 实测校准(需用户配合 MuMu 环境)

**Files:**
- Modify: `arknights_pixel_autofill.py`(仅当实测暴露参数/坐标问题)
- Modify: `quantize.py`(仅当预设效果与参考差距明显)

**Interfaces:**
- Consumes: 全部已实现功能

- [ ] **Step 1: MuMu 集成测试**(请用户启动 MuMu + 明日方舟并进入 24×24 画板;模拟器内分辨率 16:9、游戏 UI 100%)

清单:客户端类型切 MuMu → 开始填充 → 依次验证:窗口命中的是渲染区(校准状态显示"自动校准 NNN×NNN px")、调色板拖拽滚动到达底部且第二阶段颜色 25–40 可点、全图填充结果与预览一致、F8 急停生效。
Expected: 全部通过;若拖拽误差大,调 `_scroll_palette_drag` 的 240/640 端点或 drag 的 duration/settle。

- [ ] **Step 2: 算法效果对比**

在 scratchpad 的 ark-pd 克隆里 `npm install && npm run dev`,同一张测试图分别在网页版与本工具出图,逐预设目视对比(重点:平衡推荐/轮廓清晰/经典取样)。
Expected: 构图与主色一致、细节差异可接受;差距明显时调 `PRESETS` 参数(方向:vote_w 控色块纯度,smooth/passes 控杂点,dark_bias 控轮廓)。

- [ ] **Step 3: 回归单测**

Run: `py -m pytest tests/ -v`
Expected: 16 passed(若调过参数,`test_quantize_flat_single_color` 等仍须通过)

- [ ] **Step 4: Commit**(仅当有改动)

```bash
git add arknights_pixel_autofill.py quantize.py
git commit -m "fix: 按MuMu实测校准滚动端点与预设参数"
```

---

### Task 10: README + 打包验证

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: 全部功能定稿

- [ ] **Step 1: 更新 README**

- 「v1.2.0 本次更新」整节替换为「v1.3.0 本次更新」:客户端类型下拉正式支持 MuMu 模拟器(子窗口定位 + 拖拽滚动);生成算法下拉(10 预设)替换旧的缩放取样/颜色匹配/抖动/减少过渡色;每图统计缓存,切换预设即时。
- 「主要功能」同步改写;删除旧算法相关行,补「生成算法」预设表(10 行,含各自适用场景一句话)。
- 「安卓模拟器」节改为「MuMu 模拟器(正式支持)」:客户端类型选 MuMu、模拟器 16:9 + 游戏 UI 100%、关键字默认 MuMu 可手改;其他模拟器仍属自行适配。
- 「项目结构」加 `quantize.py`、`tests/test_quantize.py`。
- 新增「算法致谢」小节:量化算法思路参考 [ark-pd](https://gitee.com/guluzz/ark-pd)(PolyForm Noncommercial 1.0.0),本仓库为独立重写实现,保持免费非商业分发。

- [ ] **Step 2: 打包冒烟**

Run: `pyinstaller --noconfirm --clean arknights_pixel_autofill.spec`
Expected: 构建成功;`dist/明日方舟像素画自动填色.exe` 启动正常,「生成算法」「客户端类型」两下拉在打包版可用(quantize.py 已被自动收集)。

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: v1.3.0 说明与MuMu使用指引"
```

---

### Task 11(**执行前需用户单独确认**):综合优点新增 2 个预设

按用户要求放在最后:先保证 10 个原有预设落地,再综合两边优点补充。候选方案(确认时可改):

- **「照片抖动」**:承接旧版 Floyd–Steinberg 能力,适合照片/渐变。不走投票/ICM:取每格 `mean_rgb` 组成 24×24 图,做 FS 误差扩散 + CIELAB 最近邻。
- **「精简16色」**:承接旧版"减少过渡色"。跑一遍"平衡推荐"→ 统计用色频率 → 保留前 16 色 → 其余色成本置为无穷大后重跑 argmin+ICM。

**Files:**
- Modify: `quantize.py`
- Test: `tests/test_quantize.py`

**Interfaces:**
- Produces: `PRESETS` 增至 12 项(两个新预设的 `Preset` 参数复用"平衡推荐",实际分派逻辑在 `quantize_matrix` 内按名称走特殊管线);GUI 无需改动(下拉 values 取自 PRESETS)

- [ ] **Step 1: 写失败测试**(追加)

```python
def test_photo_dither_diffuses_gradient():
    img = Image.linear_gradient("L").resize((q.SUPER, q.SUPER)).convert("RGB")
    stats = q.compute_cell_stats(img)
    matrix = q.quantize_matrix(stats, "照片抖动")
    assert all(0 <= v < 40 for row in matrix for v in row)
    # 渐变经抖动后的用色数应多于经典取样(误差扩散引入过渡)
    classic = q.quantize_matrix(stats, "经典取样")
    assert len({v for r in matrix for v in r}) >= len({v for r in classic for v in r})


def test_reduced_16_limits_colors():
    # 三通道错位渐变,保证基线用色远超 16,使上限断言真正生效
    ramp = Image.linear_gradient("L").resize((q.SUPER, q.SUPER))
    img = Image.merge("RGB", (ramp, ramp.rotate(90), ramp.rotate(180)))
    stats = q.compute_cell_stats(img)
    baseline = q.quantize_matrix(stats, "平衡推荐")
    assert len({v for row in baseline for v in row}) > 16   # 前提成立才有意义
    matrix = q.quantize_matrix(stats, "精简16色")
    assert len({v for row in matrix for v in row}) <= 16
```

- [ ] **Step 2: 运行确认失败**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 2 FAIL,`KeyError: '照片抖动'`

- [ ] **Step 3: 实现**

`PRESETS` 末尾加两项(参数占位,分派按名称):

```python
    "照片抖动": Preset(1.00, 0.00, 0.00, 1.01, 0.00, 0.00, 0),
    "精简16色": Preset(1.00, 1.35, 0.25, 0.30, 0.72, 0.42, 4),
```

`quantize_matrix` 改为按名称分派:

```python
def quantize_matrix(stats, preset_name):
    if preset_name == "照片抖动":
        return _quantize_dither(stats)
    if preset_name == "精简16色":
        return _quantize_reduced(stats, keep=16)
    preset = PRESETS[preset_name]
    flat = icm_smooth(stats, build_costs(stats, preset), preset)
    return [flat[y * N:(y + 1) * N] for y in range(N)]


def _quantize_dither(stats):
    """每格均值色 + Floyd–Steinberg 误差扩散,适合照片与渐变。"""
    work = [[list(stats[y * N + x].mean_rgb) for x in range(N)] for y in range(N)]
    result = [[3] * N for _ in range(N)]

    def add_error(x, y, er, eg, eb, factor):
        if 0 <= x < N and 0 <= y < N:
            cell = work[y][x]
            cell[0] = max(0.0, min(255.0, cell[0] + er * factor))
            cell[1] = max(0.0, min(255.0, cell[1] + eg * factor))
            cell[2] = max(0.0, min(255.0, cell[2] + eb * factor))

    for y in range(N):
        for x in range(N):
            old = tuple(int(round(v)) for v in work[y][x])
            idx = nearest_palette_index(old)
            result[y][x] = idx
            nr, ng, nb = PALETTE[idx]
            er, eg, eb = old[0] - nr, old[1] - ng, old[2] - nb
            add_error(x + 1, y, er, eg, eb, 7 / 16)
            add_error(x - 1, y + 1, er, eg, eb, 3 / 16)
            add_error(x, y + 1, er, eg, eb, 5 / 16)
            add_error(x + 1, y + 1, er, eg, eb, 1 / 16)
    return result


def _quantize_reduced(stats, keep=16):
    """先按平衡预设出图,保留高频 keep 色后重新优化,合并零散过渡色。"""
    preset = PRESETS["平衡推荐"]
    costs = build_costs(stats, preset)
    flat = icm_smooth(stats, costs, preset)
    counts = [0] * len(PALETTE)
    for idx in flat:
        counts[idx] += 1
    retained = set(sorted(range(len(PALETTE)), key=counts.__getitem__, reverse=True)[:keep])
    for row in costs:
        for i in range(len(PALETTE)):
            if i not in retained:
                row[i] = float("inf")
    flat = icm_smooth(stats, costs, preset)
    return [flat[y * N:(y + 1) * N] for y in range(N)]
```

- [ ] **Step 4: 运行确认通过**

Run: `py -m pytest tests/test_quantize.py -v`
Expected: 18 passed

- [ ] **Step 5: 手动验证 + Commit**

GUI 切换两个新预设出图正常后:

```bash
git add quantize.py tests/test_quantize.py
git commit -m "feat: 新增照片抖动与精简16色预设"
```

---

## 未来方向(本期不实现,仅存档)

1. **多方案并排预览**:Toplevel 弹窗一次生成全部预设缩略图,点选应用;stats 已缓存,总耗时预计 <2s。
2. **差异填充**:开始前截图识别画布每格当前颜色(网格校准已给出格心坐标,取格心像素 CIELAB 最近邻即可),只点击与目标不同的格子;「跳过白色格」成为其特例。断点续画/二次修正大提速。

## 风险与回退

- MuMu 渲染子窗口结构与预期不符 → `find_mumu_window` 自动回退顶层客户区,网格校准兜底;仍失败则校准状态会显示"比例坐标",按 Task 9 清单排查。
- 拖拽滚动在个别 MuMu 版本触发惯性 → 调大 `settle`;极端情况改为多次短拖。
- 纯 Python 统计耗时超预期(>2s)→ 把 `px[x,y]` 改为 `list(img.getdata())` 一次读出 + 索引计算(不引入 numpy)。
