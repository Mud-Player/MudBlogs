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

### 3.3 瓦片加载

通过上一步，我们得到了视口内的瓦片编号，也知道当前所处的层级。所以，根据z/x/y.jpg规则读取本地图片，将其添加到场景中显示。首先看一下对瓦片的结构定义：

```
struct TileSpec {
        quint8 type;  ///< 瓦片类型（用于标识不同路劲资源的地图)
        quint8 zoom;  ///< 瓦片层级
        quint32 x;    ///< 瓦片X轴编号
        quint32 y;    ///< 瓦片Y轴编号
        ……
};
```

1. 获取文件名

   ```
       int tileCount = qPow(2, tileSpec.zoom);
       QString fileName = QString("%1/%2/%3/%4")
               .arg(m_path)
               .arg(tileSpec.zoom)
               .arg(tileSpec.x)
               .arg(m_bTMS ? tileCount - tileSpec.y - 1 : tileSpec.y);
   ```

   其中，`m_path`是瓦片地图资源路径，`tileSpec`是具体的某一个瓦片信息，`m_bTMS`保存是否为TMS协议，如果是的TMS协议，则需要对y编号进行翻转，即`tileCount - tileSpec.y - 1`。

2. 加载图片并设置Z值

   ```
       auto tileItem = new QGraphicsPixmapItem(fileName);
       tileItem->setZValue(tileSpec.zoom - 20);
   ```

   Z值将始终不大于0，并且`tileSpec.zoom - 20`使得低层级的瓦片靠下，高精度的瓦片靠上，然后我们可以加一些优化，实现某些瓦片地图缺失的时候，使用下层的瓦片地图来代替的效果。

3. 将瓦片转换缩放位移

   ```
       double xOff = TILE_LEN * (tileSpec.x  - tileCount/2.0);
       double yOff = TILE_LEN * (tileSpec.y  - tileCount/2.0);
       double scaleFac = 1.0 / qPow(2, (tileSpec.zoom-ZOOM_BASE));
       QTransform transform;
       transform.scale(scaleFac, scaleFac)
               .translate(xOff, yOff);
       tileItem->setTransform(transform);
   ```

   其中，`TILE_LEN`是场景长宽，ZOOM_BASE也就是前文提到的层级基准（建议10层级)。`tileItem`是在上一步加载好的瓦片。

   Qt的QTransform不遵从SRT规则，如果按照OpenGL三维矩阵来思考的话，我们先是把图片缩放到合适的大小，然后在将它移动到场景上的某个位置，这样多个瓦片就排列好了。可是对于Qt而言，如果先放大2倍，再位移到(10, 10)，对于位移这一步，Qt会在放大2倍后的坐标系上进行位移，所以会位移到(20,20)。
   因此，要将Qt里的瓦片转换正确，需要这样做：先按照瓦片的原始大小将所有瓦片排列好成为一整完整的地图，然后再将这张完整的地图当作一个整体缩放到场景的大小。如果先位移到(10,10)，再放大2倍，位移量(10, 10)也会重新定位到(20, 20)。所以呢，QTransform是先执行`scale`再执行`translate`。
   另外：涉及到一个场景原点问题，可以以地图中心为原点，也可以以地图左上角为原点，亦或者以地图左下角为原点，etc。无论哪种方式对地图显示本身不影响，但是为了方便经纬高与场景坐标的转换算法，最好还是以地图中心为原点。

### 3.4 层级切换

实现层级切换的效果，可以通过QGraphicsView::setTransform对视口进行缩放，以达到对地图放到和缩小的操作。

视口在未缩放情况下，与base层级基准的差值是`level - base`，那么要缩放到指定层级，缩放因子就是$scale = 2^{level - base}$，因此，下面这行代码就能实现缩放到指定的层级：
`QGraphicsView::setTransform(QTransform::fromScale(scale, scale)`。

### 3.5 GIS计算

注：地图中心为场景原点坐标。

- 场景坐标转经纬度

  ```
      auto radLon = point.x() * 2 * M_PI/ SCENE_LEN;
      auto radLat = 2 * qAtan(qPow(M_E, 2*M_PI*point.y()/SCENE_LEN)) - M_PI_2;
      return  QGeoCoordinate(qRadiansToDegrees(-radLat), qRadiansToDegrees(radLon));
  ```

- 经纬度转场景坐标

  ```
      double radLon = qDegreesToRadians(coord.longitude());
      double radLat = qDegreesToRadians(coord.latitude());
      double x = SCENE_LEN * radLon / 2.0 / M_PI;
      double y = SCENE_LEN / 2.0 / M_PI * qLn( qTan(M_PI_4+radLat/2.0) );
      return QPointF(x, -y);
  ```

### 3.6 优化

#### 3.6.1 异步加载

1. 创建一个多线程类`GraphicsMapThread`，负责接收`QGraphicsView`的瓦片请求：
   `connect(this, &GraphicsMap::tileRequested, m_mapThread, &GraphicsMapThread::requestTile, Qt::QueuedConnection);`
   地图窗口监听鼠标事件、缩放事件等，在当前视口内瓦片变化的时候，发送`tileRequested`信号；在多线程的requestTile函数里面，实现异步加载瓦片文件。

2. 地图窗口所在的主线程接收多线程的瓦片加载和卸载信号，实现地图的更新：

   ```
    connect(m_mapThread, &GraphicsMapThread::tileToAdd, this->scene(), QGraphicsScene::addItem, Qt::QueuedConnection);
    connect(m_mapThread, &GraphicsMapThread::tileToRemoce, this->scene(), QGraphicsScene::removeItem, Qt::QueuedConnection);
   ```
   
#### 3.6.2 瓦片缓存

我们在操作地图过程中，拖动地图或者缩放地图就意味着新的瓦片被加载，旧的瓦片被卸载。我们不应该已发现有瓦片看不见，就立马卸载它并delete掉，因为它很可能下一秒又需要加载显示，所以这时候需要一个瓦片缓存的机制。

QGraphicsView只是负责瓦片的显示和隐藏，而瓦片载体QGraphicsPixmapItem的生命周期是多线程`GraphicsMapThread`管理的，`GraphicsMapThread`负责瓦片的new和delete。我们可以利用`QCache`来自动管理生存周期，当瓦片数量超过缓存区的的时候，再把最不常访问的瓦片删掉。

#### 3.6.3 缺省瓦片

全球地形相当之大，我们通常不会全部存储在本机上。所以，我们通常下载底层级的全球瓦片地图，再根据项目实际需求下载局部范围的高层级瓦片地图。

如果我们在浏览一些没有地图的区域的时候呢，这个时候对应这一层级的瓦片文件是不存在的，那么我们如何处理？

有些地图框架会使用纯色背景或者带文字的背景，来替代无法加载的瓦片。但是我们有一个更好的办法，就是利用上一层级的瓦片地图来代替。比如：我们正在加载`10/15/15.jpg`瓦片，但是本地没有下载这张瓦片，那么我们尝试加载`9/7/7.jpg`，如果`9/7/7.jpg`也没有，那么再尝试加载8/3/3.jpg，依次规律，我们直到加载`0/0/0.jpg`位置，总不会这一张都没有吧！！！

因此，这里需要一种递归算法，来解决缺省瓦片的问题，直到找到最顶层瓦片：

```
void GraphicsMapThread::createAscendingTileCache(const GraphicsMap::TileSpec &tileSpec, QSet<GraphicsMap::TileSpec> &sets)
{
    auto tileCacheItem = m_tileCache.object(tileSpec);
    if(!tileCacheItem) {
        tileCacheItem = new GraphicsMapThread::TileCacheNode;
        tileCacheItem->value = loadTileItem(tileSpec);
        tileCacheItem->tileSpec = tileSpec;
        m_tileCache.insert(tileSpec, tileCacheItem);
    }
    sets.insert(tileSpec);

    if(!tileCacheItem->value && tileSpec.zoom != 0)
        createAscendingTileCache(tileSpec.rise(), sets);
}
```

其中`tileSpec.rise()`实现的是向上一层级提升。

#### 3.6.4 地图交互

地图交互包括滚轮缩放、鼠标拖动、鼠标点击等等操作。这里建议继承一个新的类，专门用来管理交互事件。同时提出一种操作器的概念，专用于交互地图的事件委托类，分离交互逻辑，使代码更清晰明了。

可交互地图：

```
class GRAPHICSMAPLIB_EXPORT InteractiveMap : public GraphicsMap
{
    Q_OBJECT
public:
    InteractiveMap(QWidget *parent = nullptr);

    /// 创建地图元素
    template<class T>
    T *addMapItem();
    /// 删除地图元素
    template<class T>
    void removeMapItem(T* item);
    /// 清空该类管理的所有圆形
    template<class T>
    void clearMapItem();

    /// 设置事件交互操作器，传nullptr可以取消设置
    void setOperator(InteractiveMapOperator *op = nullptr);
    /// 保持对象居中，传空值可以取消设置
    void setCenter(const MapObjectItem *obj);
    /// 设置鼠标是否可以交互缩放
    void setZoomable(bool on);

protected:
    virtual void wheelEvent(QWheelEvent *e) override;
    //  将要传递给操作器的事件
    virtual void keyPressEvent(QKeyEvent *event) override;
    virtual void keyReleaseEvent(QKeyEvent *event) override;
    virtual void mouseDoubleClickEvent(QMouseEvent *event) override;
    virtual void mouseMoveEvent(QMouseEvent *event) override;
    virtual void mousePressEvent(QMouseEvent *event) override;
    virtual void mouseReleaseEvent(QMouseEvent *event) override;

private:
    InteractiveMapOperator *m_operator;     ///< 操作器
    const MapObjectItem    *m_centerObj;    ///< 居中对象
    QGraphicsView::DragMode m_dragMode;     ///< 拖拽模式(用于取消居中之后回到之前的模式)
    QGraphicsView::ViewportAnchor m_anchor; ///< 鼠标锚点(用于取消居中之后回到之前的模式)
    //
    bool  m_scaleable;  ///< 是否可以鼠标缩放
};
```

可交互地图操作器：

```
class GRAPHICSMAPLIB_EXPORT InteractiveMapOperator : public QObject
{
    Q_OBJECT
    friend InteractiveMap;
public:
    InteractiveMapOperator(QObject *parent = nullptr);
    inline QGraphicsScene *scene() const {return m_scene;};
    inline GraphicsMap *map() const {return m_map;};

private:
    inline void setScene(QGraphicsScene *scene) {m_scene = scene;};
    inline void setMap(InteractiveMap *map) {m_map = map;};

protected:
    virtual void ready(){}; /// 重新被设置为操作器的时候将会被调用
    virtual void end(){};   /// 操作器被取消的时候将会被调用
    virtual bool keyPressEvent(QKeyEvent *) {return false;};
    virtual bool keyReleaseEvent(QKeyEvent *) {return false;};
    virtual bool mouseDoubleClickEvent(QMouseEvent *) {return false;};
    virtual bool mouseMoveEvent(QMouseEvent *) {return false;};
    virtual bool mousePressEvent(QMouseEvent *) {return false;};
    virtual bool mouseReleaseEvent(QMouseEvent *) {return false;};

protected:
    QGraphicsScene *m_scene;
    InteractiveMap *m_map;
};
```

## 4. 源码

代码测试：

```
   auto map = new GraphicsMap;
   map->setTilePath("E:/map/sate");
   map->show();
```

![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/02GraphicsMapLib/quick_sate.png)

源码地址：<https://github.com/Mud-Player/GraphicsMapLib>
