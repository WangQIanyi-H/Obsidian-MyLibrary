本文将介绍，在使用RenderDoc软件在游戏中进行截帧后，该如何利用rdc文件，来导出世界坐标的模型，从而还原游戏场景。

本篇文章将从以下几个方面对该流程进行阐述
- 导出单个模型：使用renderdoc的python接口、fbxsdk对单个drawcall进行导出，在此处创建材质，匹配贴图，介绍导出贴图的一些坑（renderdoc版本，python版本）
- 找到变换矩阵：通过找到vp矩阵或者vp矩阵的逆，将模型从local space转换到world space（在转换过程中会有误差）
- 进行批量导出：该如何进行批量导出，批量导出时需要注意到的一些坑
- 导入Unity中并生成Prefab，对场景进行还原：将模型导入Unity后，将如何批量生成Prefab及其lod，并且采用Prefab的实例化方式来代替一个个的模型

# 单个模型导出
想要还原整个世界场景，首先要从导出单个模型开始。
想要在RenderDoc中导出模型，我们需要先在Event Browser界面选择我们想要导出的那个模型被绘制的那个DrawCall（一般都是通过遍历Event，来判断这个Event是否是Action，然后在判断这个Action的指令是否是一个DrawCall，如果是一个DrawCall在进行下面的操作）。
这时在Mesh Viewer界面我们可以看到对应模型的信息。如下图所示，VS Input指的是输入进顶点着色器的模型信息，VS Output指的是经过MVP变换之后的模型信息。
利用VS Input中的信息，结合FBX SDK，我们可以生成模型空间下的Mesh。（具体到如何获取，可以查阅RenderDoc的文档[RenderDoc文档](https://renderdoc.org/docs/python_api/examples/renderdoc/decode_mesh.html)和FBX SDK的文档）
VS Output的信息我们在下面也会用到。
与此同时，我们在Texture Viewer界面下的Inputs窗口中，可以找到这个DrawCall下输入进片元着色器的贴图，我们可以通过这些贴图的命名，来识别出Diffuse、Normal等贴图，在导出模型时通过FBX SDK来创建模型材质来绑定这些贴图，这样子在后续模型进入Unity后，模型会自动关联到这些贴图，而不至于是白模。（在Unreal引擎做的游戏所截的帧中得到的rdc文件，通常贴图没有一个可识别的命名，这种就无能为力了）
## Event Browser

### Event
在RenderDoc中，事件（Event）指的是各种图形API的调用。这些事件可以包括
1. **绘制调用**（Draw Calls）：这些是图形应用程序中最常见的事件之一，涉及到将几何数据（如顶点）送入图形管线进行渲染。
2. **状态改变**：这些事件涉及到改变图形管线的状态，例如改变混合模式、剪裁设置或其它渲染状态。
3. **资源操作**：包括创建、修改、删除纹理、缓冲区等图形资源的事件。
4. **分发计算调用**：在使用计算着色器时执行的调用，这些着色器不直接涉及渲染，而是用于各种计算任务。
5. **内存操作**：例如复制或更新缓冲区或纹理的内容。
在RenderDoc中，可以通过Event Browser来查看这些事件。它允许用户逐一检查每个事件。
### Action
在RenderDoc中，“Action"是指引发GPU执行工作或影响内存及资源（如纹理和缓存区）的具体调用或事件。这些动作通常包括但不限于以下几种：
1. **绘制调用**（Draw Calls）：这是最常见的动作类型，涉及将几何数据（例如顶点和索引）发送到GPU进行渲染处理。
2. **分发调用**（Dispatch Calls）：这些是计算着色器的调用，它们不是直接进行图形渲染，而是执行例如数据处理等计算任务。
3. **清除操作**（Clears）：清除特定图形资源，如帧缓冲、纹理或深度缓冲区。
4. **复制操作**（Copies）：在资源之间复制数据，例如从一个纹理复制到另一个纹理。
5. **解析操作**（Resolves）：涉及多采样纹理的处理，如将多采样帧缓冲区解析为单采样纹理
在RenderDoc中，一个Action一定是一个Event，一个Event并不一定是一个Aciton。

# 还原世界坐标

# 批量导出
