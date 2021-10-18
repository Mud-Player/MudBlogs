# [OSG]大数据顶点坐标造成图元闪烁分析与解决

## 1. 问题描述

某一次需要在地球上绘制模型，当时通过osg::Geometry绘制模型的时候，将ECEF坐标系上的值作为了模型的顶点坐标，发现绘制出来的模型顶点变形，在相机移动的时候，模型的三角面也疯狂地闪烁抖动。

## 2. 分析

首先，模型能够看得见，说明模型本身的顶点坐标是有效的。

其次，模型是静态模型，没有更新顶点，在相机移动过程中，看到模型的图元闪烁，这里的直觉是顶点乘以MVPW矩阵之后的结果产生了误差！

刚好又想到一个地方，我没有手动使用着色器，也就是使用的OpenGL固定渲染管线，OSG在每一帧更新的时候会向渲染管线设置顶点坐标、ModelView矩阵、Projection矩阵、Window矩阵。在固定渲染管线里面，用的是单精度浮点数，地球的半径在6371000米左右，如果其中一个顶点坐标的值是（6371000.5, 0, 0)，当被传到渲染管线的时候，就变成了（6371000.00, 0, 0)，对于渲染管线来说，顶点坐标不就出错了吗！！！

所以，要避免精度损失，就得将顶点坐标变小，尽量靠近局部坐标系原点。

那么将ECEF的值，减去一个偏移量Offset，作为模型的顶点坐标，然后给这个模型一个偏移Offset的矩阵。因为图元最终屏幕坐标是
$$
ScreenPos = Position * Model * View * Projection * Window
$$
这里其实也有一个问题，Model 矩阵也包含很大数据的浮点数用来做位移变换，同样有精度损失问题。但还好对于渲染管线来说是这样计算图元屏幕坐标：
$$
ScreenPos = Position * ModelView * Projection * Window
$$
ModelView矩阵就是Model矩阵和View矩阵的结合，OSG会在每一帧更新ModelView矩阵到固定管线。Model矩阵带有大数据浮点数，View矩阵通常也带有大数据浮点数，提前在CPU使用双精度浮点数计算 Model * View ，Model、View矩阵的大数据浮点数可以相互抵消得到一个较小数据的浮点数（靠近相机坐标系原点），即便传到了渲染管线，单精度浮点数也不会出现精度损失。

## 3. 解决方法

使用osg::MatrixTransform作为Model矩阵，代码示例：

```
    // ECEF顶点/世界坐标系顶点
    osg::ref_ptr<osg::Vec3Array> ecefArray = new osg::Vec3Array;
    ecefArray->push_back(osg::Vec3d(6371000.00, 6371000.00, 0));
    ecefArray->push_back(osg::Vec3d(6371002.00, 6371000.00, 0));
    ecefArray->push_back(osg::Vec3d(6371000.00, 6371002.00, 0));
    // 模型局部坐标系顶点
    osg::ref_ptr<osg::Vec3Array> vertexArray = new osg::Vec3Array;
    osg::Vec3d origin = ecefArray->at(0);
    for (auto &vertex : *ecefArray) {
        vertexArray->push_back(vertex - origin);
    }
    osg::ref_ptr<osg::Geometry> geometry = new osg::Geometry;
    osg::ref_ptr<osg::Geode> geo = new osg::Geode();
    geo->addDrawable(geometry);
    geometry->setVertexArray(vertexArray);
    geometry->addPrimitiveSet(new osg::DrawArrays(osg::PrimitiveSet::POLYGON, 0, 3));
    // Model矩阵
    osg::ref_ptr<osg::MatrixTransform> model = new osg::MatrixTransform;
    model->addChild(geo);
    model->setMatrix(osg::Matrix::translate(origin));
```



