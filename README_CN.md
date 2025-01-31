# TIFFImageryProvider

在Cesium中加载GeoTIFF/COG（Cloud optimized GeoTIFF）。

[![gzip size](http://img.badgesize.io/https://unpkg.com/tiff-imagery-provider@latest?compression=gzip&label=gzip)](https://unpkg.com/tiff-imagery-provider) ![npm latest version](https://img.shields.io/npm/v/tiff-imagery-provider.svg) ![license](https://img.shields.io/npm/l/tiff-imagery-provider)

[中文readme](./README_CN.md)

## 特点

- 三波段渲染。
- 多模式颜色渲染。
- 支持在地图上查询TIFF值。
- 支持任何投影的TIFF。
- Web Workers 加速。
- WebGL 加速渲染。
- 波段计算。

## 安装

```bash
#npm
npm install --save tiff-imagery-provider
#pnpm
pnpm add tiff-imagery-provider
```

## 使用

基本用法

```ts
import { Viewer } from "cesium";
import TIFFImageryProvider from 'tiff-imagery-provider';

const cesiumViewer = new Viewer("cesiumContainer");

const provider = new TIFFImageryProvider({
  url: 'https://oin-hotosm.s3.amazonaws.com/56f9b5a963ebf4bc00074e70/0/56f9c2d42b67227a79b4faec.tif',
});
provider.readyPromise.then(() => {
  cesiumViewer.imageryLayers.addImageryProvider(provider);
})

```

如果TIFF的投影不是EPSG：4326，您可以传递``projFunc``来处理投影

```ts
import proj4 from 'proj4';

new TIFFImageryProvider({
  url: YOUR_TIFF_URL,
  projFunc: (code) => {
    if (code === 32760) {
      proj4.defs("EPSG:32760", "+proj=utm +zone=60 +south +datum=WGS84 +units=m +no_defs +type=crs");
      return proj4("EPSG:32760", "EPSG:4326").forward
    }
  }
});
```

波段计算

```ts
// NDVI
new TIFFImageryProvider({
  url: YOUR_TIFF_URL,
  renderOptions: {
    single: {
      colorScale: 'rainbow',
      domain: [-1, 1],
      expression: '(b1 - b2) / (b1 + b2)'
    }
  }
});
```

## API

```ts
class TIFFImageryProvider {
  ready: boolean;
  readyPromise: Promise<void>
  bands: Record<number, {
    min: number;
    max: number;
  }>;
  constructor(options: TIFFImageryProviderOptions);

  get isDestroyed(): boolean;
  destroy(): void;
}

interface TIFFImageryProviderOptions {
  url: string;
  credit?: string;
  tileSize?: number;
  maximumLevel?: number;
  minimumLevel?: number;
  enablePickFeatures?: boolean;
  hasAlphaChannel?: boolean;
  renderOptions?: TIFFImageryProviderRenderOptions;
  /** 投影函数，将[经度，纬度]位置转换为EPSG：4326 */
  projFunc?: (code: number) => (((pos: number[]) => number[]) | void);
  /** 缓存生存时间，默认为60 * 3000毫秒 */
  cache?: number;
  /** geotiff 重采样方法, 默认为 nearest */
  resampleMethod?: 'nearest' | 'bilinear' | 'linear';
}

type TIFFImageryProviderRenderOptions = {
  /** 无效值，默认从tiff meta读取 */
  nodata?: number;
  /** 尝试将多波段cog渲染为rgb，优先级 1 */
  convertToRGB?: boolean;
  /** 优先级 2 */
  multi?: MultiBandRenderOptions;
  /** 优先级 3 */
  single?: SingleBandRenderOptions;
}

interface SingleBandRenderOptions {
  /** 波段索引从1开始，默认为1 */
  band?: number;

  /**
   * 使用的颜色比例尺图像。
   */
  colorScaleImage?: HTMLCanvasElement | HTMLImageElement;

  /**
   * 使用的命名颜色比例尺的名称。
   */
  colorScale?: ColorScaleNames;

  /** 自定义插值颜色，[stopValue(0-1), color]或[color]，如果后者，表示等分布 
   * @example
   * [[0, 'red'], [0.6, 'green'], [1, 'blue']]
  */
  colors?: [number, string][] | string[];

  /** 默认为连续 */
  type?: 'continuous' | 'discrete';

  /**
   * 将值域缩放到颜色。
   */
  domain?: [number, number];

  /**
   * 将呈现的值的范围，超出范围的值将透明。
   */
  displayRange?: [number, number];

  /**
   * 设置是否应使用displayRange。
   */
  applyDisplayRange?: boolean;

  /**
   * 是否对域以下的值进行夹紧。
   */
  clampLow?: boolean;

  /**
   * 是否对域以上的值进行夹紧（如果未定义，则默认为clampLow值）。
   */
  clampHigh?: boolean;
  
  /**
   * 设置要在绘图上评估的数学表达式。表达式可以包含具有整数/浮点值、波段标识符或带有单个参数的GLSL支持函数的数学运算。
   * 支持的数学运算符为：add '+', subtract '-', multiply '*', divide '/', power '**', unary plus '+a', unary minus '-a'。
   * 有用的GLSL函数例如：radians、degrees、sin、asin、cos、acos、tan、atan、log2、log、sqrt、exp、ceil、floor、abs、sign、min、max、clamp、mix、step、smoothstep。
   * 这些函数的完整列表可以在[GLSL 4.50规范](https://www.khronos.org/registry/OpenGL/specs/gl/GLSLangSpec.4.50.pdf)的第117页找到。
   * 不要忘记设置domain参数!
   * @example 
   * '-2 * sin(3.1415 - b1) ** 2'
   * '(b1 - b2) / (b1 + b2)'
   */
  expression?: string;
}

interface MultiBandRenderOptions {
  /** 波段值从1开始 */
  r?: {
    band: number;
    min?: number;
    max?: number;
  };
  g?: {
    band: number;
    min?: number;
    max?: number;
  };
  b?: {
    band: number;
    min?: number;
    max?: number;
  };
}

type ColorScaleNames = 'viridis' | 'inferno' | 'turbo' | 'rainbow' | 'jet' | 'hsv' | 'hot' | 'cool' | 'spring' | 'summer' | 'autumn' | 'winter' | 'bone' | 'copper' | 'greys' | 'ylgnbu' | 'greens' | 'ylorrd' | 'bluered' | 'rdbu' | 'picnic' | 'portland' | 'blackbody' | 'earth' | 'electric' | 'magma' | 'plasma' | 'redblue' | 'coolwarm' | 'diverging_1' | 'diverging_2' | 'blackwhite' | 'twilight' | 'twilight_shifted';
```

## 示例

[在线示例](https://tiff-imagery-provider-example.vercel.app/)

- 使用 [Next.js](https://github.com/vercel/next.js) 搭建。
- 使用 [Semi-UI](<https://github.com/DouyinFE/semi-design>) 实现暗黑模式。
- 实现了简单的 COG 自定义渲染方法。

在 `demo` 文件夹中启动应用程序，然后访问 <http://localhost:3000/>：

```node
pnpm install
cd example
pnpm start
```

![screenshot.png](/pictures/screenshot.png) | ![classify.png](/pictures/classify.png) | ![landsat.png](/pictures/landsat.png)
| ------- | ------- | -------- |

## 已知问题

- Cesium@1.101 中的位置错误

## 计划

- [x] 使用 Web Workers 生成瓦片图像
- [x] 使用 GPU 加速计算
- [ ] 使用 Web Workers 加速Webgl渲染，共享上下文

## 致谢

<https://github.com/geotiffjs/geotiff.js>

<https://github.com/santilland/plotty>