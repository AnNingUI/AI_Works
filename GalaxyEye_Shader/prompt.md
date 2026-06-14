# Role & Task
你现在是全球顶级的 WebGPU 渲染工程师兼技术美术（TA），同时精通 Web Audio API 声音合成。
请你编写一个纯 WebGPU 的单文件 `index.html`，实现一个 AAA 级游戏官网风格的「交互式时序流体星云与果冻 Metaball」视觉特效。

# Strict Constraints (绝对限制)
1. **单一文件**：HTML、CSS、JS、WGSL 全部写在一个文件中。
2. **零外部依赖**：绝对禁止引入 Three.js、PixiJS 等任何库，禁止加载外部图片或音频。所有纹理（通过 Canvas2D 生成 Mipmap）和声音（Web Audio API）必须在运行时程序化生成。
3. **WGSL 极严类型安全**：WGSL 不允许隐式类型转换！标量与向量计算必须显式强转（例如 `p * 2.0 + vec2f(t * 0.05)`），所有 for 循环必须显式声明类型（例如 `for(var i: i32 = 0; i < 5; i = i + 1)`）。
4. **零 GC（0 Garbage Collection）**：在 `requestAnimationFrame` 的 `frame()` 循环中，**绝对禁止**调用 `device.createBindGroup` 等分配内存的 API。必须采用两套（A/B）静态绑定的 Swap-chain 设计处理 Ping-Pong 渲染。

# Technical Specifications (核心技术规格)

## 1. 资源生成层 (JS CPU端)
- **程序化银河纹理**：使用 Canvas2D 绘制含旋臂的 1024x1024 高清星系，使用 `drawImage` 生成完整的 Mipmap 层级。
- **纹理 Usage 关键点**：目标 GPUTexture 的 usage 必须包含 `TEXTURE_BINDING | COPY_DST | RENDER_ATTACHMENT`（必须有 RENDER_ATTACHMENT，否则 copyExternalImageToTexture 会报错）。

## 2. 渲染管线架构 (3-Pass HDR Pipeline)
所有中间 RenderTarget 必须使用 `rgba16float` 格式以支持超亮高光（HDR）。

### Pass 1: Scene Pass (主场景着色器)
- **多层视差星空**：3层不同速度的 FBM 噪声星空。
- **3D 步进体积云 (Raymarching)**：构建视线 `ro` 和 `rd`，进行 16 步轻量级 Raymarch。利用 3D FBM 计算密度，加入基于步进距离的自遮挡衰减吸收（Transmittance），生成厚重的立体体积云。
- **8节点 SDF Metaball**：解析求导计算法线，并通过高频 FBM 噪声扰动场强，实现边缘沸腾（Edge Evolution）。
- **材质渲染**：
  - **五通道物理色散 (5-Tap Spectral Dispersion)**：沿着法线按红、黄、绿、青、蓝五个步长采样银河纹理。
  - **次表面透光 (SSS)**：在 SDF 轮廓内侧叠加偏红橙色的温暖内发光。
  - **边缘霓虹发光**：计算高亮外轮廓。
  - **鼠标引力**：基于鼠标坐标对空间 `p` 产生基于指数衰减的引力畸变。

### Pass 2: Temporal Feedback Fluid (时序流体平流)
- 接收 Pass 1 输出与上一帧的结果（Ping-Pong 纹理 A/B）。
- **Curl Noise Advection**：通过 FBM 梯度计算 Curl 矢量场，扰动历史纹理的 UV。
- **混合公式**：`mix(prevAdvectedColor, currentSceneColor, 0.08)`（92% 历史拖尾，8% 实时注入）。

### Pass 3: Cinematic Post-Processing (电影级后处理)
- 接收 Pass 2 输出。
- **物理镜头径向色散 (Radial Chromatic Aberration)**：基于屏幕中心距离的 RGB 偏移采样。
- **真·多采样大光晕 (True Bloom)**：提取 HDR 亮度（>0.75），使用极坐标方向进行 12 次双半径（Near/Far）Gather 采样模糊叠加。
- **水平变形拉丝 (Anamorphic Glare)**：水平方向 `[-6, 6]` 跨度采样极高亮区域，生成横向镜头拉丝。
- **镜头脏污 (Lens Dirt)**：生成高频噪声图案，乘以后处理画面亮度的 Luma 激发脏污显影。
- **暗角与颗粒 (Vignette & Film Grain)**。
- **ACES Tonemapping & Gamma (2.2)** 校正后输出到屏幕 `swapchain`。

## 3. Web Audio API (生成式氛围音乐)
- 点击屏幕解除静音，隐藏提示文字。
- **全局效果器**：包含 Gain 控制和反馈 Delay Loop（Delay -> Gain -> Delay -> Master）。
- **Sub Bass Drone**：使用 Sine 和 Triangle 振荡器，微调产生拍频，通过 Lowpass filter 铺底。
- **LFO 扫频和弦 (Generative LFO Pads)**：
  - 定义四个宇宙和弦（如 Fmaj9, G6, Em9, Am9）。
  - 使用 `setTimeout` 每 10 秒切换和弦。
  - 每个音符都有独立缓慢的 Attack/Release Envelope（起音/释音包络），交错发声。
  - 音符输出进 Lowpass Filter，该 Filter的频率由极慢的 LFO 控制，产生呼吸感。

# Output Requirements
给出完整、无删减、无需二次修改的 `index.html` 源码。切勿使用省略号略过代码。确保 WGSL 中绝对没有 `curlNoise` 未声明、隐式类型转换或绑定异常的问题。
