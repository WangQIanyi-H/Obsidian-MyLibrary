本文将介绍，在使用RenderDoc软件在游戏中进行截帧后，该如何利用rdc文件，来导出世界坐标的模型，从而还原游戏场景。

本篇文章将从以下几个方面对该流程进行阐述
- 导出单个模型：使用renderdoc的python接口、fbxsdk对单个drawcall进行导出，在此处创建材质，匹配贴图，介绍导出贴图的一些坑（renderdoc版本，python版本）
- 找到变换矩阵：通过找到vp矩阵或者vp矩阵的逆，将模型从local space转换到world space（在转换过程中会有误差）
- 进行批量导出：该如何进行批量导出，批量导出时需要注意到的一些坑
- 导入Unity中并生成Prefab，对场景进行还原：将模型导入Unity后，将如何批量生成Prefab及其lod，并且采用Prefab的实例化方式来代替一个个的模型

