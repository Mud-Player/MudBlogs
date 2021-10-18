# [osgQt]osgQOpenGLWidget实现透明背景

1. 关闭OpenGL背景色更新，保留深度缓冲跟新
   osgQOpenGLWidget::getOsgViewer()->getCamera()->setClearMask(GL_DEPTH_BUFFER_BIT)
2. 设置Qt的OpenGL窗口在其他窗口之后绘制
   osgQOpenGLWidget::setAttribute(Qt::WA_AlwaysStackOnTop)

