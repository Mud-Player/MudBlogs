# [QtQuick]解决地图出现灰块

- Qt版本：5.12.8

## 1 问题描述

在使用到QtQuick自带Map地图类型的时候，地图中会随机性出现灰色方块。

## 2 解决方法

该问题是已知Qt的Bug，可通过设置Map的透明度opacity属性解决，比如设置到0.999，参见下面代码：

```
Map {
    opacity: 0.999
    //....
    //...
}
```

## 3 BUGREPORTS

- <https://bugreports.qt.io/browse/QTBUG-62463>
- <https://bugreports.qt.io/browse/QTBUG-67169>