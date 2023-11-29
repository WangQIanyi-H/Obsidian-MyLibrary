本文将介绍，在使用RenderDoc软件在游戏中进行截帧后，该如何利用rdc文件，来导出世界坐标的模型，从而还原游戏场景。

本篇文章将从以下几个方面对该流程进行阐述
- 导出单个模型：使用renderdoc的python接口、fbxsdk对单个drawcall进行导出，在此处创建材质，匹配贴图，介绍导出贴图的一些坑（renderdoc版本，python版本）
- 找到变换矩阵：通过找到vp矩阵或者vp矩阵的逆，将模型从local space转换到world space（在转换过程中会有误差）
- 进行批量导出：该如何进行批量导出，批量导出时需要注意到的一些坑
- 导入Unity中并生成Prefab，对场景进行还原：将模型导入Unity后，将如何批量生成Prefab及其lod，并且采用Prefab的实例化方式来代替一个个的模型

# 单个模型导出
想要还原整个世界场景，首先要从导出单个模型开始。
我们需要先选择我们想要导出的那个模型被绘制的那个Action。
官方文档有给出对Action Tree进行遍历的[示例代码](https://renderdoc.org/docs/python_api/examples/renderdoc/iter_actions.html)，可以参考示例代码找到特定的Action。
找到特定的Action后，我们就可以导出模型信息了。
我们可以在Mesh Viewer界面看到模型的相关信息。如下图所示，VS Input指的是输入进顶点着色器的模型信息，VS Output指的是经过MVP变换之后的模型信息。
![[Pasted image 20231128154333.png]]

> [!NOTE] Event
> 在RenderDoc中，事件（Event）指的是各种图形API的调用。这些事件可以包括 
> 1. **绘制调用**（Draw Calls）：这些是图形应用程序中最常见的事件之一，涉及到将几何数据（如顶点）送入图形管线进行渲染。
> 2. **资源操作**：包括创建、修改、删除纹理、缓冲区等图形资源的事件。
> 3. **分发计算调用**：在使用计算着色器时执行的调用，这些着色器不直接涉及渲染，而是用于各种计算任务。
> 4. **内存操作**：例如复制或更新缓冲区或纹理的内容。
> 在RenderDoc中，可以通过Event Browser来查看这些事件。它允许用户逐一检查每个事件。

> [!NOTE] Action
> 在RenderDoc中，“Action"是指引发GPU执行工作或影响内存及资源（如纹理和缓存区）的具体调用或事件。这些动作通常包括但不限于以下几种：
> 1. **绘制调用**（Draw Calls）：这是最常见的动作类型，涉及将几何数据（例如顶点和索引）发送到GPU进行渲染处理。
> 2. **分发调用**（Dispatch Calls）：这些是计算着色器的调用，它们不是直接进行图形渲染，而是执行例如数据处理等计算任务。
> 3. **清除操作**（Clears）：清除特定图形资源，如帧缓冲、纹理或深度缓冲区。
> 4. **清除操作**（Clears）：清除特定图形资源，如帧缓冲、纹理或深度缓冲区。
> 5. **解析操作**（Resolves）：涉及多采样纹理的处理，如将多采样帧缓冲区解析为单采样纹理
> 在RenderDoc中，Action一定是Event，Event并不一定是Aciton。

## 提取模型信息
RenderDoc提供了相应的接口让我们可以拿到VS Input和VS output中的数据。官方文档中也有关于解析Mesh数据的[示例代码](https://renderdoc.org/docs/python_api/examples/renderdoc/decode_mesh.html)可以参考。
## 提取贴图信息
## 生成FBX文件
光有Mesh的数据还不够，我们还需要将这些数据转换成fbx的格式。
我们可以使用FBX SDK来创建FBX文件。下面是根据Mesh信息创建fbx文件的示例。
```Python
def SaveAsFbx(DataFrame, saveName):  
    fbxName = os.path.basename(saveName).split(".")[0]  
    fbxManager = FbxManager.Create()  
    fbxScene = FbxScene.Create(fbxManager, "")  
    rootNode = fbxScene.GetRootNode()  
    FbxSystemUnit.ConvertScene(FbxSystemUnit.m, fbxScene)  
  
    # Mesh node创建  
    newNode = FbxNode.Create(fbxScene, fbxName)  
    FbxNode.AddChild(rootNode, newNode)  
    mesh = FbxMesh.Create(fbxScene, fbxName + "_Mesh")  
    newNode.SetNodeAttribute(mesh)  
  
    DataFrame_change = DataFrame.drop_duplicates(subset=["IDX"], keep="first", inplace=False)  
    DataFrame_change = DataFrame_change.sort_values(by="IDX", ascending=True) 
    # -----------------添加Mesh----------------#  
    mesh.InitControlPoints(len(DataFrame_change))  
    pos = []  
    idx_binding_info = []  
    for i in range(0, len(DataFrame_change)):  
        pos.append(np.array([DataFrame_change.iloc[i][attribute_export[0] + ".x"],  
                             DataFrame_change.iloc[i][attribute_export[0] + ".y"],  
                             DataFrame_change.iloc[i][attribute_export[0] + ".z"]]))  
        # 重组位置数据  
        vertexPos = FbxVector4(DataFrame_change.iloc[i][attribute_export[0] + ".x"],  
                               DataFrame_change.iloc[i][attribute_export[0] + ".y"],  
                               DataFrame_change.iloc[i][attribute_export[0] + ".z"])  
        mesh.SetControlPointAt(vertexPos, i)  
        idx_binding_info.append(int(DataFrame_change.iloc[i]["IDX"]))  
   
    # 重构三角面  
    for j in range(0, int(vertexCount / 3.0)):  
        # 将vertex与point相互绑定  
        mesh.BeginPolygon(j)  
        mesh.AddPolygon(idx_binding_info.index(int(DataFrame.iloc[j * 3]["IDX"])))  
        mesh.AddPolygon(idx_binding_info.index(int(DataFrame.iloc[j * 3 + 1]["IDX"])))  
        mesh.AddPolygon(idx_binding_info.index(int(DataFrame.iloc[j * 3 + 2]["IDX"])))  
        mesh.EndPolygon()  
  
    # ----------------添加Normal----------------#  
    if attribute_export[1] + ".x" in DataFrame_change:  
        # 创建Normal Layer  
        normalLayer = mesh.GetLayer(0)  
        if normalLayer is None:  
            mesh.CreateLayer()  
            normalLayer = mesh.GetLayer(0)  
  
        # 创建layerElementNormal  
        layerElementNormal = FbxLayerElementNormal.Create(mesh, "")  
        layerElementNormal.SetMappingMode(FbxLayerElement.EMappingMode(1))  # EMappingMode.eByControlPoint  
        layerElementNormal.SetReferenceMode(FbxLayerElement.EReferenceMode(0))  # EReferenceMode.eDirect  
  
        # 从DataFrame_change中获取Normal数组  
        normalArray = []  
        for i in range(0, len(DataFrame_change)):  
            normalArray.append(np.array([DataFrame_change.iloc[i][attribute_export[1] + ".x"],  
                                         DataFrame_change.iloc[i][attribute_export[1] + ".y"],  
                                         DataFrame_change.iloc[i][attribute_export[1] + ".z"],  
                                         DataFrame_change.iloc[i][attribute_export[1] + ".w"]]))  
  
        # 将Normal数组传入layerElementNormal中  
        for i in range(0, len(DataFrame_change)):  
            vertexNormal = FbxVector4(normalArray[i][0], normalArray[i][1], normalArray[i][2], normalArray[i][3])  
            layerElementNormal.GetDirectArray().Add(vertexNormal)  
  
        normalLayer.SetNormals(layerElementNormal)  
  
    # ----------------添加Vertex Color----------------#  
    # 创建Vertex Color Layer  
    if attribute_export[3] + ".x" in DataFrame_change:  
        vcolorLayer = mesh.GetLayer(0)  
        if vcolorLayer is None:  
            mesh.CreateLayer()  
            vcolorLayer = mesh.GetLayer(0)  
  
        # 创建layerElementVcolor  
        layerElementVcolor = FbxLayerElementVertexColor.Create(mesh, "")  
        layerElementVcolor.SetMappingMode(FbxLayerElement.EMappingMode(1))  # EMappingMode.eByControlPoint  
        layerElementVcolor.SetReferenceMode(FbxLayerElement.EReferenceMode(0))  # EReferenceMode.eDirect  
  
        # 从DataFrame_change中获取Normal数组  
        vcolorArray = []  
        for i in range(0, len(DataFrame_change)):  
            vcolorArray.append(np.array([DataFrame_change.iloc[i][attribute_export[3] + ".x"],  
                                         DataFrame_change.iloc[i][attribute_export[3] + ".y"],  
                                         DataFrame_change.iloc[i][attribute_export[3] + ".z"],  
                                         DataFrame_change.iloc[i][attribute_export[3] + ".w"]]))  
        # 将Vertex Color数组传入layerElementVcolor中  
        for i in range(0, len(DataFrame_change)):  
            vertexColor = FbxColor(vcolorArray[i][0], vcolorArray[i][1], vcolorArray[i][2], vcolorArray[i][3])  
            layerElementVcolor.GetDirectArray().Add(vertexColor)  
        normalLayer.SetVertexColors(layerElementVcolor)  
  
    # ----------------添加UV0----------------#  
    # 创建UV0 Layer  
    if attribute_export[2] + ".x" in DataFrame_change:  
        uv0Layer = mesh.GetLayer(0)  
        if uv0Layer is None:  
            mesh.CreateLayer()  
            uv0Layer = mesh.GetLayer(0)  
  
        # 创建layerElementUV0  
        layerElementUV0 = FbxLayerElementUV.Create(mesh, attribute_export[2])  
        layerElementUV0.SetMappingMode(FbxLayerElement.EMappingMode(1))  # EMappingMode.eByControlPoint  
        layerElementUV0.SetReferenceMode(FbxLayerElement.EReferenceMode(0))  # EReferenceMode.eDirect  
  
        # 从DataFrame_change中获取TEXCOORD0数组  
        uv0Array = []  
        for i in range(0, len(DataFrame_change)):  
            uv0Array.append(np.array([DataFrame_change.iloc[i][attribute_export[2] + ".x"],  
                                      DataFrame_change.iloc[i][attribute_export[2] + ".y"]]))  
        # 将uv0Array传入layerElementUV0中  
        for i in range(0, len(DataFrame_change)):  
            vertexUV0 = FbxVector2(uv0Array[i][0], uv0Array[i][1])  
            layerElementUV0.GetDirectArray().Add(vertexUV0)  
        uv0Layer.SetUVs(layerElementUV0)  
  
    # ----------------FBX导出----------------#  
    saveScene(saveName, fbxManager, fbxScene, pAsASCII=True)  
    fbxManager.Destroy()  
    del fbxManager, fbxScene, DataFrame
```
与此同时，我们也需要导出当前DrawCall下所使用的贴图资源，可以参照RenderDoc官网的[示例代码](https://renderdoc.org/docs/python_api/examples/renderdoc/save_texture.html)对贴图进行导出。
在贴图命名正确的情况下，可以根据命名规律找出Diffuse、Normal等贴图。然后导出模型时通过FBX SDK来创建模型材质来绑定这些贴图，这样子在后续模型进入Unity后，模型会自动关联到这些贴图，而不至于是白模。（在Unreal引擎做的游戏所截的帧中得到的rdc文件，通常贴图没有一个可识别的命名，这种就无能为力了）
# 还原世界坐标
我们现在可以拿到VS Input和VS Output中的Mesh信息，但是想要还原整个世界场景，我们还需要将VS Input（模型空间）或是VS Output（屏幕空间）的Mesh信息转换到世界空间。
想要将模型空间坐标或是屏幕空间坐标还原到世界空间，我们首先需要简单了解一下MVP变换。
## MVP变换介绍
在图形流水线中，MVP变换指的是一系列坐标空间变换，这些转换通过将模型变换（Model Transform）、视图变换（View Transform）和投影变换（Projection Transform）相结合来实现。这三种变换共同组成了MVP变换，它们将3D场景中的对象转换到一个二维图像上，以便在屏幕上渲染。下面详细介绍每个组成部分：
### 模型变换（Model Transform）
- 用于将对象从模型空间（对象自己的局部坐标系统）转换到世界空间（场景的全局坐标系统）。
- 它包括平移（Translation）、旋转（Rotation）和缩放（Scale）操作。
### 视图变换（View Transform）
- 将对象从世界空间转换到视图空间（也称为摄像机空间或眼睛空间）。
- 在视图空间中，观察点（通常是虚拟摄像机）位于原点，所有对象都相对于这个观察点进行定位。
- 这个变换通常由摄像机的位置和方向决定。
### 投影变换（Projection Transform）
- 将视图空间中的3D坐标转换到剪裁空间（Clip Space），并最终通过透视除法转换为归一化设备坐标（Normalized Device Coordinates, NDC）。
- 这个变换定义了视锥体（View Frustum），它决定了哪些部分的场景被渲染到屏幕上。
- 投影变换可以是透视投影（Perspective Projection）或正交投影（Orthographic Projection）。
### 引擎中的MVP变换
以Unity为例。在Unity中，我们可以直接拿到MVP变换矩阵。

MVP变换通常是在顶点着色器中进行。在引擎中，顶点着色器会输出顶点在裁剪空间的位置，然后图形管线会根据顶点着色器的输出，将这些坐标自动进行NDC变换，转换到NDC空间，以便于后续的光栅化和像素着色阶段。

> [!NOTE] RenderDoc中的VS Input和VS Output
> 我们在RenderDoc的MeshViewer中看见的VS Input是未经过MVP变换的Mesh信息，VS Output是经过了MVP变换转换到了裁剪空间，但是并没有转换到NDC空间。

在Unity的实际编程中，通常来说并不需要直接操作这些矩阵，因为Unity提供了更高层次的抽象。但是我们需要了解这些以便于更好的去理解后面的步骤。
### 利用变换矩阵还原世界坐标

## 找到变换矩阵
我们现在知道了
- 介绍MVP变换
- 找到矩阵
- 还原
# 批量导出
