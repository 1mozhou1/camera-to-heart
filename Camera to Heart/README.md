# Camera to Heart 💖

前置摄像头手势识别的 3D 粒子爱心交互动画。

**捏合手指 → 爱心从手心成形 → 拖拽移动 → 松手崩散入海。**

---

## 快速开始

1. 用本地服务器打开（不能用 file:// 协议）：
   - VS Code: 装 Live Server 插件，右键 HTML → "Open with Live Server"
   - 命令行: `npx serve .` 然后访问 `http://localhost:3000`
   - 或部署到 GitHub Pages / Vercel 等平台
2. 允许摄像头权限
3. 等待「就绪」提示消失
4. 拇指食指捏合 + 其余三指弯曲 → 爱心成形
5. 捏紧锁定后拖拽移动爱心
6. 松手 → 爱心崩散

> 无摄像头时自动降级为纯动画演示模式。

---

## 技术架构

```
单文件 HTML ≈ 620 行 + 本地模型文件
├── 渲染: Canvas 2D，直接 arc+fill 画粒子
├── 手势: MediaPipe Hands 0.4（本地自托管，mediapipe-hands/ 目录）
├── 爱心: 3D 穹顶投影 + 深度排序 + Phong 光照
└── 海面: Gerstner 波 + 环形涟漪 + 漂流 + 泡沫
```

## 渲染管道

```
背景渐变 → 海底微光 → 海面粒子(4000) → 爱心粒子(3000)
         → 轮廓线 → 外层柔光 → 融化尾迹 → 崩散粒子(1500) → 预览窗
```

## 手势状态机

```
未检测 ─(6帧防抖)→ 张开 ─(捏合>0.15)→ 升起 ─(5帧≥0.78)→ 锁定
                                                    ←(5帧≤0.3)←
锁定解除 → 崩散(1500粒子爆炸坠落) → 海面融合 → 再捏合锁定 → 崩散状态自动复位
```

---

## 参数调整指南

所有参数在 `Camera to Heart.html` 中，搜索对应变量名即可修改。

### 粒子数量

| 变量 | 默认 | 说明 |
|------|------|------|
| `SEA_N` | `4000*quality` | 海面粒子（自适应缩放）|
| `HEART_N` | `3000*quality` | 爱心粒子（含前后半球）|
| `SHATTER_N` | `1500*quality` | 崩散粒子池 |
| `SPLASH_N` | `200*quality` | 溅射/尾迹粒子池 |
| `GLOW_N` | `50*quality` | 海底微光数量 |
| `SPARKLE_N` | `80*quality` | 闪烁粒子池 |

`quality = min(1, min(W,H)/700)`，手机自动减半。

### 爱心外观

| 变量/位置 | 默认 | 说明 |
|-----------|------|------|
| `heartX(t)` | `16*sin³(t)` | 心形宽度，改 16 调宽窄 |
| `heartY(t)` | `-(13cos(t)-5cos(2t)-2cos(3t)-cos(4t))` | 心形方程 |
| `BULGE` | `10.5` | 3D 球面鼓出幅度，越大越立体 |
| `L_KEY` | `(-0.58, 0.56, 0.70)` | 主光源方向 |
| `L_AMB` | `0.035` | 环境光强度（暗面底线）|
| 颜色梯度 | 搜索 `bright < 0.14` | 4 段亮度→颜色映射，按需改 RGB |

### 爱心颜色

搜索 `// ── 颜色（粉雾→蜜桃 渐变）──`，4 段亮度映射：

```javascript
// 暗面 → 中间 → 亮面 → 高光
bright < 0.14: (80,20,40) → (120,40,65)    // 深玫红
bright < 0.34: (120,40,65) → (195,90,120)  // 暖粉
bright < 0.56: (195,90,120) → (235,150,180) // 蜜桃
else:          (235,150,180) → (252,210,230) // 亮白粉
```

### 手势灵敏度

| 变量 | 默认 | 说明 |
|------|------|------|
| `pinchCloseRef` | 自适应 | 闭合基准，越大越容易触发 |
| `curl` 阈值 | `palmW*0.55` | 手指弯曲判定，改大=更宽松 |
| `curlBoost` | `0.5 + curl*0.5` | 软增益，两指弯曲+捏合85%即可锁 |
| `SPRING_STIFFNESS` | `0.3` | 爱心形态变化速度 |
| `SPRING_DAMPING` | `0.85` | 爱心形态回弹阻尼 |
| `pinchLocked` 触发 | 5帧 ≥ 0.78 | 锁定所需连续帧数 |
| `pinchLocked` 解除 | 5帧 ≤ 0.3 | 解锁所需连续帧数 |
| 捏合距离 | `dTip*0.7 + min(dTip,dPIP)*0.3` | 指尖主导+食指PIP辅助防抖 |
| 粒子预升起 | `p - 0.08` | 爱心粒子开始升空的捏合阈值（越小越早） |
| 崩散重置 | 锁定时自动复位 | 每次锁定重置崩散状态，支持反复崩散 |

### 海面

| 变量/位置 | 默认 | 说明 |
|-----------|------|------|
| `surfaceY2D()` | 5 层正弦波 | 海浪频率和振幅 |
| `seaTop` | `H*0.65` | 海面基线位置 |
| 涟漪周期 | 每 0.008 概率生成 | 搜索 `ripples.length < 5` |
| 涟漪振幅 | `10` | 搜索 `Math.sin(rPhase) * 10` |

### 动画

| 变量 | 默认 | 说明 |
|------|------|------|
| `自转速度` | `time * 0.35` | 爱心 Y 轴自转，改 0.35 调速度 |
| `自转倾角` | `-0.10` | X 轴倾斜角度 |
| `呼吸频率` | `3.2` Hz | 爱心缩放呼吸 |
| `崩散重力` | `0.10` | 崩散粒子下落速度 |
| `融化尾迹速率` | `meltAlpha * 6` | 尾迹粒子生成速率 |

---

## 性能优化

- **粒子数自适应**：`quality = min(1, min(W,H)/700)`，小屏自动减半
- **离屏 canvas 禁用**：已验证 `drawImage(offscreenCanvas)` 导致 GPU stall
- **`.toFixed()` 禁用**：每帧数千次字符串分配会卡死
- **预收集子集**：如 `edgeParticles` 避免每帧遍历全量

---

## 已知限制

- 需要 HTTPS 或 localhost（file:// 协议不支持摄像头）
- 摄像头帧率影响手势检测频率
- 手掌出画面边缘无法检测（MediaPipe 硬限制）
- 无音频（Web Audio 可后期添加）

---

## 文件结构

```
Camera to Heart/
├── Camera to Heart.html     ← 主文件
├── README.md
├── mediapipe-hands/         ← MediaPipe Hands 0.4 模型文件（本地自托管）
│   ├── hands.js             ←   主库
│   ├── hands_solution_simd_wasm_bin.js  ← WASM 加载器
│   ├── hands_solution_simd_wasm_bin.wasm ← WASM 二进制 (6MB)
│   ├── hand_landmark_full.tflite         ← 手势检测模型 (5.5MB)
│   ├── hand_landmark_lite.tflite         ← 轻量模型 (2MB)
│   ├── hands.binarypb                    ← 模型图描述
│   └── hands_solution_packed_assets.data ← 打包资源 (4.3MB)
├── 历史版本/
│   ├── Heart Gem Model.html
│   ├── Pinch to Heart.html
│   ├── Camera to Heart README.md
│   └── Camera to Heart 调试速查.txt
└── 参考文档/
    ├── 手势粒子爱心_修正指令.txt
    └── 3D粒子爱心手势动画_Claude提示词.txt
```

---

## 依赖

- [MediaPipe Hands 0.4](https://www.npmjs.com/package/@mediapipe/hands) — 已本地自托管至 `mediapipe-hands/`，无需联网加载
- 浏览器需支持 `getUserMedia`（HTTPS 或 localhost，不支持 file://）
