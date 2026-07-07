# solar-system-modern.html 程序化星球生成分析

分析对象:`docs/solar-system-modern.html`
技术栈:浏览器 + Three.js `0.160.0`（ES module + importmap）+ 自实现 Perlin 噪声

## 整体架构

一个基于 **Three.js + 自实现 Perlin 噪声** 的程序化行星生成器。核心思路很经典：**用 3D 噪声对一个球体的每个顶点做径向位移（高度场），再根据高度值上色**，最后叠加海洋层和大气辉光。

生成流程调用链：`init()` → `createPlanet()` → 内部再调用 `createOcean()` / `createAtmosphere()`，由 UI 滑块的 `change` 事件触发重新生成。

## 1. 噪声引擎：Perlin + FBM

`NoiseGenerator` 类实现的是经典的 **Ken Perlin 改进版噪声**（不是页脚宣称的 Simplex）：

- **构造函数**：生成 0–255 的排列表并随机洗牌，复制成 512 长度的 `p` 数组。**关键点：每次 `new NoiseGenerator()` 才会重新洗牌**，但整个页面只在加载时 `new` 了一次（`const noiseGen = new NoiseGenerator()`）。
- `fade()`：五次平滑插值曲线 `6t⁵-15t⁴+10t³`。
- `grad()` / `noise()`：标准三维梯度噪声，对 8 个晶格角三线性插值，输出约 `[-1, 1]`。
- `fbm()`（分形布朗运动）：叠加 6 个倍频程，每层振幅 ×0.5、频率 ×2.0，最后除以 `maxValue` 归一化。低频层决定大陆/海洋轮廓，高频层叠加山脉细节。

```javascript
fbm(x, y, z, octaves = 6) {
    let value = 0, amplitude = 1, frequency = 1, maxValue = 0;
    for (let i = 0; i < octaves; i++) {
        value += amplitude * this.noise(x*frequency, y*frequency, z*frequency);
        maxValue += amplitude;
        amplitude *= 0.5;   // 每层振幅减半
        frequency *= 2.0;   // 每层频率翻倍
    }
    return value / maxValue; // 归一化回 [-1, 1]
}
```

## 2. 顶点位移（地形高度）

在 `createPlanet()` 中建 `SphereGeometry(radius, detail, detail)`，遍历每个顶点：

```javascript
const noiseValue = noiseGen.fbm(x*scale*0.1, y*scale*0.1, z*scale*0.1, 6);
const normalizedNoise = (noiseValue + 1) / 2;              // 映射到 [0,1]
const heightOffset = (normalizedNoise - 0.5) * strength * 0.5;
const scale = 1 + Math.max(-0.3, heightOffset);           // 沿径向缩放
positions.setXYZ(i, x*scale, y*scale, z*scale);
```

要点：
- **直接用顶点坐标作为噪声采样输入**（3D domain），球面上连续无缝，没有平面贴图接缝。
- `* 0.1` 把世界坐标缩到合适噪声频率；`scale` 滑块进一步控制大陆疏密。
- **沿径向缩放** `pos * (1 + offset)` 实现球形隆起，而非简单加法。
- `Math.max(-0.3, heightOffset)` 给下陷设下限，防止海沟塌成尖点。
- 生成后 `computeVertexNormals()` 重算法线，配合 `flatShading` 开关切换低多边形风格。

## 3. 基于高度的顶点着色

同一循环里按 `normalizedNoise` 与 `oceanLevel` 关系分三段（写入顶点色 `colors`）：

| 条件 | 区域 | 颜色处理 |
|------|------|---------|
| `noise < oceanLevel` | 深/浅海 | `deepWaterColor`↔`waterColor` 按深度插值 |
| `< oceanLevel + 0.05` | 海岸线 | 固定沙滩色 |
| 其余 | 陆地 | `baseColor × (0.7 + 高度×0.3 + 二次噪声×0.1)`，`landHeight>0.75` 再叠白色雪峰 |

陆地段额外采样一次不同频率的 `noise()` 做颜色扰动，避免大片纯色。

## 4. 海洋层与大气层

- **海洋** `createOcean()`：仅当 `oceanLevel > 0.1` 创建，半径 `radius*0.995` 的独立光滑半透明球体，盖住低于海平面的地形。
- **大气** `createAtmosphere()`：半径 `radius*1.15` 的球壳，自定义 `ShaderMaterial` 做**菲涅尔边缘辉光**：

```glsl
float fresnel = pow(1.0 - dot(vNormal, viewDirection), 3.0);
gl_FragColor = vec4(color, fresnel * intensity);
```

配合 `side: BackSide` + `AdditiveBlending` + `depthWrite:false` 做出大气光晕。

## 5. 星球类型与参数系统

`CONFIGS` 预设三种星球，本质只是换参数：

- **terran（类地）**：中等起伏 + 海洋 0.45。
- **gas（气态巨行星）**：`strength=0.15`、`scale=0.5`、`oceanLevel=0`，暖色调。
- **ice（冰封）**：`strength=0.7`、`scale=2.0`（高频破碎），冷白色。

`randomize()` 随机化 strength/scale/oceanLevel 并用 `setHSL` 随机配色。

## 值得注意的点 / 潜在问题

1. **"Regenerate" 不会真正换地形**：噪声排列表只在加载时洗牌一次。要每次生成不同星球，应在 `regenerate()` 里 `noiseGen = new NoiseGenerator()` 重新洗牌，或引入种子。
2. **性能**：`detail=150` 约 2.3 万顶点，每顶点跑 6 层 FBM（每层完整 3D 噪声），纯 CPU。滑块 `change` 每次全量重建，高精度下明显卡顿。高性能做法应放到 GPU 顶点着色器。
3. **命名与实现不符**：页脚宣称 Simplex，实际是 Perlin classic gradient noise。
4. **海洋是独立球体**而非同网格水面顶点：简单、可单独设材质，代价是多一次 draw call 和顶点开销。

## 与其它两个项目的关系

这是三个项目里最"玩具级"的：单球体、CPU 噪声、全量重建、无 LOD、顶点色。适合理解"噪声高度场 + 上色"的最小闭环，但不具备行星尺度或可编辑能力。
