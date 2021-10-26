# [Qt]Graphics View实现简单离线地图

## 1. 瓦片地图概述

地图资源本身是图片资源，多张256×256分辨率的图片拼接起来，给用户显示的将是一个完整的地图效果。这些256×256的图片被称作瓦片地图。瓦片地图有着典型的金字塔结构，如下图：

![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/03GraphicsViewMap/tile_structure.png)

在第0层级，仅有1张图片即可表示整个世界地图；在第1层级，需要4张图片表示整个世界地图，在第2层级，需要16张图片表示整个世界地图。所以规律就是层级每提升一级，那么原先1张图片就要拆分为4张图片，对应的瓦片数量也要扩大4倍。

同时，这里将每张图片进行分类和命名，才能快速索引！对于本地资源，每张图片是按照`z/x/y.jpg`(或png)的方式存放的，目录结构如下：

```
├── 0
│   └── 0
│       └── 0.jpg
├── 1
│   ├── 0
│   │   ├── 0.jpg
│   │   └── 1.jpg
│   └── 1
│       ├── 0.jpg
│       └── 1.jpg
├── 2
│   └── 0
│   │   ├── 0.jpg
│   │   ├── 1.jpg
│   │   ├── 2.jpg
│   │   └── 3.jpg
│   ├── 1
│   │   ├── 0.jpg
│   │   ├── 1.jpg
│   │   ├── 2.jpg
│   │   └── 3.jpg
│   ├── 2
│   │   ├── 0.jpg
│   │   ├── 1.jpg
│   │   ├── 2.jpg
│   │   └── 3.jpg
│   └── 3
│       ├── 0.jpg
│       ├── 1.jpg
│       ├── 2.jpg
│       └── 3.jpg
├── 3
│ ……………………
```
`z`是瓦片资源的最一层文件夹，代表瓦片所属的层级，以0为起始随层级增大而递增；
`x`是瓦片资源的第二层文件夹，代表瓦片在水平方向的编号，以0为起始从左侧向右侧递增；
`y.jpg`是瓦片资源的第二层文件夹里面的具体图片文件，代表瓦片在垂直方向的编号。以0为起始递增，值得注意的是，TMS协议的y编号是从下向上递增，而XYZ协议的y编号是从上向下递增。

## 2. 实现思路

首先，我们需要QGraphicsPixmapItem来显示瓦片；其次，我们还要所有的瓦片正确拼接，以及正确且及时地替换层级先后关系。那么，最简单的方法是将多个层级的图片缩放到同样的大小，再绘制到场景上，示意图如下：

![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/03GraphicsViewMap/methoad_scale.png)

上图中，我们是以第2层级的地图为基准，所以QGraphicsScene场景的大小是（256×4，256×4）。在实际开发中，瓦片地图资源通常是0到20层级，所以，建议将第10层级作为基准，这时候0层级的瓦片需要放大1024倍，20层级需要缩小0.0009765625倍。如果极端一点选用0层级作为基准，那么20层级需要缩小0.00000095367431640625倍，这会引起浮点数的精度问题！！！设`base`为层级基准，那么某一层级的瓦片缩放公式如下：
$$
scale = 1.0\  /\  2^{level-base}
$$
单从加载图片的流程上来说，我们可以分为下面几步：

1. 根据层级、水平方向编号、垂直方向标号找到本机对应的瓦片文件z/x/y.jpg；
2. 使用QGraphicsPixmapItem加载找到的瓦片文件；
3. 对QGraphicsPixmapItem进行转换，调整大小、移动位置；
4. 将QGraphicsPixmapItem添加到QGraphicsScene。

## 3. Graphics View实现

### 3.1 场景尺寸

场景尺寸，亦即QGraphicsScene的画布大小。我们需要给场景一个固定的大小，不会随着显示层级变化而变化。

对于base层级基准，场景的长和宽都是`(1<<base) * 256`，也就是水平/垂直方向瓦片个数乘以256。

### 3.2 计算可见瓦片编号

1. 首先，有一个成员变量存储当前的层级level，因此可以得到当前层级瓦片的水平方向和垂直方向的数量，即`qPow(2, int(level))`。

2. 将地图窗口四个角映射到场景坐标，场景坐标再映射到瓦片编号。摘抄自源码如下：

   ```
       auto topLeftPos = mapToScene(viewport()->geometry().topLeft()-QPoint(256,256));
       auto topRightPos = mapToScene(viewport()->geometry().topRight());
       auto bottomLeftPos = mapToScene(viewport()->geometry().bottomLeft());
       qint32 xOrigin = (topLeftPos.x()+SCENE_LEN/2) / SCENE_LEN * tileCount;
       qint32 yOrigin = (topLeftPos.y()+SCENE_LEN/2) / SCENE_LEN * tileCount;
       qint32 xHor = (topRightPos.x()+SCENE_LEN/2) / SCENE_LEN * tileCount;
       qint32 yHor = (topRightPos.y()+SCENE_LEN/2) / SCENE_LEN * tileCount;
       qint32 xVer = (bottomLeftPos.x()+SCENE_LEN/2) / SCENE_LEN * tileCount;
       qint32 yVer = (bottomLeftPos.y()+SCENE_LEN/2) / SCENE_LEN * tileCount;
   ```

   考虑到地图旋转问题，源码中使用xOrigin/yOrigin表示起始瓦片，xHor/yHor表示视口水平方向末端瓦片，xVer/yVer表示视口垂直方向末端瓦片。并且在垂直方向以XYZ协议为准（向下递增），在加载本地图片资源的时候可以根据实际情况对y进行翻转。

### 3.4 层级切换

实现层级切换的效果，可以通过QGraphicsView::setTransform对视口进行缩放，以达到对地图放到和缩小的操作。

视口在未缩放情况下，与base层级基准的差值是`level - base`，那么要缩放到指定层级，缩放因子就是$scale = 2^{level - base}$，因此，下面这行代码就能实现缩放到指定的层级：
`QGraphicsView::setTransform(QTransform::fromScale(scale, scale)`。

### 3.

其中关键点是第二步，如何转换？Qt的QTransform不遵从SRT规则，在Qt里需要先位移再缩放。
同样以上图为例，我们要将Level 1转换到Level 2为基准的场景中，即转换到（256×4，256×4）的场景大小上。

1. 第一步，位移：涉及到一个场景原点问题，可以以地图中心为原点，也可以以地图左上角为原点，亦或者以地图左下角为原点，etc。无论哪种方式对地图显示本身不影响，但是为了方便经纬高与场景坐标的转换算法，最好还是以地图中心为原点。所以Level

