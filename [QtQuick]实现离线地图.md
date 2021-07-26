# [QtQuick]实现离线地图

- Qt版本：5.12.8

## 1 需求分析

使用Qt实现离线地图，大多数软件是通过*GraphicsView*框架，结合瓦片地图相关算法来实现的，但这些绝大多数都不太讨喜。从项目时间、成本和质量的角度考虑，我们需要一个开发周期短、人力成本低、软件质量有保障的方案。

自QtLocation 5.0开始，Qt推出了*Map QML Type*，这意味着我们可以使用Qt自带的地图，而不用自己再去实现非常底层的算法了。

> The Map type is used to display a map or image of the Earth, with the capability to also display interactive objects tied to the map's surface.

## 2 方案设计

从Qt文档可以看出，Qt通过Map类型作为地图显示介质，通过六种地图插件提供数据驱动。地图插件见下表：

| 插件名称                           | 描述                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| Qt Location Esri Plugin            | Uses Esri for location services.                             |
| Qt Location HERE Plugin            | Uses the relevant services provided by HERE.                 |
| Qt Location Items Overlay Plugin   | Provides an empty map intended to be used as background for an overlay layers for map items. |
| Qt Location Mapbox GL Plugin       | Uses Mapbox GL for location services.                        |
| Qt Location Mapbox Plugin          | Uses Mapbox for location services.                           |
| Qt Location Open Street Map Plugin | Uses Open Street Map and related services.                   |

最后一个*Qt Location Open Street Map Plugin*，这个插件默认就支持离线地图。先罗列OSM支持的几种地图加载模式：

1. 网络地图

   通过TMS瓦片服务器获取瓦片地图。

   OSM默认提供TMS服务器配置地址，在本机搭建一个TMS瓦片服务器，就能实现加载本地瓦片地图。**这个方案不够高效，放弃该方案**。

2. 离线地图

   加载本机存储介质上的瓦片地图。

   按照固定格式命名瓦片地图文件，就可以实现加载本地瓦片地图。**这个方案可行，但有优化的地方**。

3. 缓存地图

   读取网络地图后，存放在本机的缓存瓦片地图。**这属于Qt内部的缓存机制，暂时不考虑入手**。
   

从实现离线的地图的目的来说，前两种方式都可行。但为了简单可靠，我们还是从它离线地图的功能开始研究。离线地图需要按照的固定命名格式，但我们下载的资源碰巧不符合；换个方向从源码入手，修改其读取文件名称的规则又有了新的解决方案。方案总结如下：
- 按照官方的命名规则使用离线地图。
- 修改官方源码，定义我们自己的瓦片命名规则。

### 2.1 固定瓦片规则

按照官方的规定，瓦片地图需要按照`osm_100-<l|h>-<map_id>-<z>-<x>-<y>.<extension>`的命名方式存放在同一个文件夹。

市面上常见地图下载软件是按照z/x/y的形式保存的瓦片地图，所以下载完瓦片地图后，需要批量修改文件名称以适应OSM的要求，命名规则详见[QtLocation: using offline map tiles with the OpenStreetMap plugin](https://www.qt.io/blog/2017/05/24/qtlocation-using-offline-map-tiles-openstreetmap-plugin)。

**通过该方案，每一次下载新地图都需要重新命名。**

### 2.2 自定义瓦片规则

修改OSM的源码，再固定瓦片命名规则的基础上，增加主流的文件命名规则。以Arcgis地图为例，在本地是以`z/x/y.png(jpg)`的方式存放，其中`z`代表层级，`x`代表x轴瓦片编号，`y`代表y轴瓦片编号。找到源码里面关于读取离线地图的代码，然后修改其加载我们自定义规则的瓦片数据。设计其加载规则首先是官方规则，如果没有找到文件，那么然后加载我们的自定义规则。我们定义瓦片文件层次结构如下：

1. 第一层：瓦片根路径；
2. 第二层：地图类型（street, satellite, cycle, transit, night-transit, terrain, hiking）；
3. 第三层：地图层级（z）；
4. 第四层：水平瓦片编号（x）；
5. 第五层：竖直瓦片编号（y）。

例如：googlemap/satellite/10/258/346.png

**通过该方案，一劳永逸。**

## 3 实现

### 3.1 批量命名

通过Python、C++等都可以实现批量命名。具体实现略。

### 3.2 源码定制

我们目的是将官方的osm插件修改为mud（或其他名称）插件。修改源码分三步：1. 安装源码 2. 修改源码 3. 编译运行

#### 3.2.1 源码

安装源码有三种办法：

1. 安装Qt的时候，勾选上`Sources`（另外建议勾选MinGW，比MSVC能够调试更多源码）；
2. 下载[完整源码包](http://download.qt.io/official_releases/qt/5.12/5.12.8/single)解压，或者单独下载 [QtLocation源码包](http://download.qt.io/official_releases/qt/5.12/5.12.8/submodules/)解压 (链接以Qt5.12.8为例)；
3. 从Github克隆 <https://github.com/qt/qt5>。

接下来为修改源码做准备：

1. QtCreator添加`子目录项目`命名为`MudMap`(或其他名称），再添加`Qt Quick Application - Empty`项目命名为`MudViewer`（或其他名称，后面用于显示地图）；

2. 打开`qtlocation\src\plugins\geoservices`文件夹，拷贝`osm`文件夹到`MudMap`目录并重命名为`MudPlugin`（或其他名称），再重命名`MudPlugin`种的`osm.pro`为`MudPlugin.pro`，再重命名`MudPlugin`种的`osm_plugin.pro`为`mud_plugin.pro`以作为后面修改源码的基础；

3. 打开`qtlocation`文件夹，拷贝`.qmake.conf`文件到`MudMap`的同目录（可以略过该步以观察报错信息）；

4. 修改`MudMap.pro`文件内容为
   ```
   TEMPLATE = subdirs
   
   SUBDIRS += \
       MudPlugin \
       MudViewer
   ```

准备好后的工程目录如下：

> MudMap
>
> > MudPlugin
> >
> > > MudPlugin.pro
> > >
> > > \*.\* h/cpp/json
> >
> > MudViewer
> >
> > > MudViewer.pro
> > >
> > > \*.\* cpp/qrc/qml
> >
> > .qmake.conf
> >
> > MudMap.pro

最后对`MudMap`工程`执行qmake`。

#### 3.2.2 编码

- 工程修改

  1. 修改`mud_plugin.json`第2-3行内容：

      ```
       "Keys": ["mud"],
       "Provider": "mud",
      ```

  2. 修改`MudPlugin.pro`第1行内容：

      ```
   TARGET = qtgeoservices_mud
      ```

  3. 修改`MudPlugin.pro`第41-42行内容

      ```
   OTHER_FILES += \
       mud_plugin.json
      ```

- 源码修改

  1. 修改`qgeoserviceproviderpluginosm.h`第52-53行（这里修改后，就能够编译过了，如果编译不过，那么前面的步骤有误）：

     ```
         Q_PLUGIN_METADATA(IID "org.qt-project.qt.geoservice.serviceproviderfactory/5.0"
                           FILE "mud_plugin.json")
     ```

  2. 修改`qgeotiledmappingmanagerengineosm.cpp`第72行：

     ```
         const QByteArray pluginName = "mud";
     ```

  3. 全局替换代码`osm.mapping`为`mud.mapping`（共36处）

     值得注意的是，Qt内部实现会对输入给地图插件的参数进行过滤。以osm插件为例，如果传递给mud插件的参数包含一个"osm.mapping.offline.directory"参数，而正好Qt发现有一个叫做osm的插件，"osm."开头的参数就不会传递给mud插件。反之，如果把qtgeoservices_osm.dll文件删掉之后，mud就能收到“osm."开头的参数。

  3. 在`qgeofiletilecacheosm.h/cpp`添加函数：

     ```
     /*!
      * \return 返回自定义规则的文件绝对路径
      */
     QString QGeoFileTileCacheOsm::tileSpecToAbsFilename(const QGeoTileSpec &spec)
     {
         QString subDir;
         QString absFileName;
     
         // mapID地图类型，范围1-7 （文件夹）
         switch (spec.mapId()) {
         case 1:
             subDir += "/street"; break;
         case 2:
             subDir += "/satellite"; break;
         case 3:
             subDir += "/cycle"; break;
         case 4:
             subDir += "/transit"; break;
         case 5:
             subDir += "/night-transit"; break;
         case 6:
             subDir += "/terrain"; break;
         case 7:
             subDir += "/hiking"; break;
         default:
             break;
         }
     
         // 地图层级 (文件夹）
         subDir += "/";
         subDir += QString::number(spec.zoom());
     
         // 水平编号 （文件夹)
         subDir += "/";
         subDir += QString::number(spec.x());
     
         // 竖直编号 （文件）
         QString fileNameFilter = QString::number(spec.y()) + ".*";
     
         // 文件过滤，找到第一个可用的瓦片文件
         QDir fileDir = m_offlineDirectory.path() + subDir;
         QStringList validTiles = fileDir.entryList({fileNameFilter});
         if (validTiles.size()) {
             absFileName = fileDir.absoluteFilePath(validTiles.first());
         }
     
         return absFileName;
     }
     ```

  4. 在`qgeofiletilecacheosm.h/cpp`修改函数`QGeoFileTileCacheOsm::getFromOfflineStorage`的实现：

     ```
     QSharedPointer<QGeoTileTexture> QGeoFileTileCacheOsm::getFromOfflineStorage(const QGeoTileSpec &spec)
     {
         if (!m_offlineData)
             return QSharedPointer<QGeoTileTexture>();
     
         int providerId = spec.mapId() - 1;
         if (providerId < 0 || providerId >= m_providers.size())
             return QSharedPointer<QGeoTileTexture>();
     
         QString fileName;
         const QString fileNameFilter = tileSpecToFilename(spec, QStringLiteral("*"), providerId);
         QStringList validTiles = m_offlineDirectory.entryList({fileNameFilter});
     
         // 使用osm默认的命名规则
         if (validTiles.size()) {
             fileName = m_offlineDirectory.absoluteFilePath(validTiles.first());
         }
         // 如果osm的规则没有找到瓦片文件，那么就使用自定义规则
         else {
             fileName = tileSpecToAbsFilename(spec);
         }
     
         QFile file(fileName);
         if (!file.open(QIODevice::ReadOnly))
             return QSharedPointer<QGeoTileTexture>();
         QByteArray bytes = file.readAll();
         file.close();
     
         QImage image;
         if (!image.loadFromData(bytes)) {
             handleError(spec, QLatin1String("Problem with tile image"));
             return QSharedPointer<QGeoTileTexture>(0);
         }
     
         addToMemoryCache(spec, bytes, QString());
         return addToTextureCache(spec, image);
     }
     ```


#### 3.2.3 编译运行

- 编译

  1. 执行qmake；
  2. 构建；
  3. 在构建路径下将`plugins\geoservices`的`geoservices`文件夹拷贝到构建路径`MudViewer\debug`（或者`MudViewer\release`）下。

- 运行

  1. 拷贝测试代码到`main.qml`并`执行qmake`，注意修改地图中心和根路径：

     ```
     import QtQuick 2.12
     import QtQuick.Window 2.12
     import QtLocation 5.12
     import QtPositioning 5.11
     
     Window {
         id: win
         objectName: "window"
         visible: true
         width: 512
         height: 512
     
         Map {
             id: map
             anchors.fill: parent
             activeMapType: map.supportedMapTypes[1]  // 1代表卫星地图
             center: QtPositioning.coordinate(40.39, 99.79)  // 这里写地图显示中心
             opacity: 0.999	// 防止透明度引起的Bug(这是Qt的Bug)
     
             plugin: Plugin {
                 name: 'mud';
                 PluginParameter {
                     name: "mud.mapping.offline.directory"
                     value: 'D:/googlemaps'  // 这里写地图根路径
                 }
             }
         }
     }
     ```

  2. 运行效果

     ```
     脑补画面---
     ```

  3. 发布

     编译生成的`geoservices`文件夹就可以作为插件使用了，当其他项目使用时，需要`geoservices`文件夹和`exe执行文件`在同一个目录（windows平台）。

## 4. 总结

经过修改源码的osm，不仅保持了原有的功能，还有一点小升级。

除了离线功能外，osm还提供了20多项的参数设置，"osm.mapping.offline.directory"只是其中一项，更多参数参见：[Qt Location Open Street Map Plugin-Parameters](https://doc.qt.io/qt-5/location-plugin-osm.html#parameters)。因此，mud插件同样支持这些参数设置，只需要将“osm."开头的参数全部以”mud."开头，就能让mud和osm有一样的功能了。

**源代码：**

<https://github.com/Mud-Player/MudMap>