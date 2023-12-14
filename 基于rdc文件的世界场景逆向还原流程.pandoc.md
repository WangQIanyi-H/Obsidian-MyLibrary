本文将介绍，在使用RenderDoc软件在游戏中进行截帧后，该如何利用rdc文件，来导出带有世界坐标的模型，从而还原游戏场景。
本文将从以下几个方面对该流程进行阐述： -
如何导出单个模型：介绍了如何提取Mesh信息、如何生成fbx文件以及贴图导出的内容 -
找到变换矩阵：通过找到vp矩阵或者vp矩阵的逆，将模型从local
space转换到world space，或是从Clip Space转到World
Space（在转换过程中会有误差） -
进行批量导出：该如何进行批量导出，批量导出时需要注意到的一些坑 \#
单个模型导出 在RenderDoc的Mesh
Viewer界面中，可以看到网格的相关信息，根据这些信息可以生成并导出fbx模型文件。
同时，在Texture
View界面可以看到输入的贴图信息，我们需要将其批量导出成tga格式。 \##
提取Mesh信息 在RenderDoc的Mesh Viewer界面中，可以发现有VS Input和VS
output两组信息（可能某些地方不叫这个名字，但是大致形式差不多），如下图所示。
\![\[Pasted image 20231214111638.png\]\]
RenderDoc提供了相应的接口让我们可以拿到VS Input和VS
output中的数据，其中，VS Input指的是输入进顶点着色器的模型信息，VS
Output指的是经过MVP变换之后的模型信息。关于mesh信息的导出可以参考官方文档中的[示例代码](https://renderdoc.org/docs/python_api/examples/renderdoc/decode_mesh.html)。
\## 生成FBX文件
参考官方文档的[示例代码](https://renderdoc.org/docs/python_api/examples/renderdoc/decode_mesh.html)可以获取mesh信息。
在获取了mesh信息之后，我们可以使用FBX
SDK来创建FBX文件（想要了解FBX文件结构可以参考[这篇文章](https://zhuanlan.zhihu.com/p/657784007)）。下面是根据Mesh信息创建fbx文件的代码示例。

``` python
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

## 贴图导出

与此同时，我们也需要导出当前DrawCall下所使用的贴图资源，可以参照RenderDoc官网的[示例代码](https://renderdoc.org/docs/python_api/examples/renderdoc/save_texture.html)对贴图进行导出。
在贴图命名正确的情况下，可以根据命名规律找出Diffuse、Normal等贴图。然后导出模型时通过FBX
SDK来创建模型材质来绑定这些贴图，这样子在后续模型进入Unity后，模型会自动关联到这些贴图，而不至于是白模。（在Unreal引擎做的游戏所截的帧中得到的rdc文件，通常贴图没有一个可识别的命名，这种就无能为力了）
\# 世界坐标还原 单个Local
Space的模型导出是简单的，但是想要还原整个世界场景，还需做进一步处理。
想要还原世界坐标，就需要将模型空间坐标或是屏幕空间坐标转换到世界空间。在进行转换之前，我们先简单了解一下MVP变换。
\## MVP变换介绍
在图形流水线中，MVP变换指的是一系列坐标空间变换，这些转换通过将模型变换（Model
Transform）、视图变换（View Transform）和投影变换（Projection
Transform）相结合来实现。这三种变换共同组成了MVP变换，它们将3D场景中的对象转换到一个二维图像上，以便在屏幕上渲染。下面详细介绍每个组成部分：
\### 模型变换（Model Transform） -
用于将对象从模型空间（对象自己的局部坐标系统）转换到世界空间（场景的全局坐标系统）。 -
它包括平移（Translation）、旋转（Rotation）和缩放（Scale）操作。 \###
视图变换（View Transform） -
将对象从世界空间转换到视图空间（也称为摄像机空间或眼睛空间）。 -
在视图空间中，观察点（通常是虚拟摄像机）位于原点，所有对象都相对于这个观察点进行定位。 -
这个变换通常由摄像机的位置和方向决定。 \### 投影变换（Projection
Transform） - 将视图空间中的3D坐标转换到剪裁空间（Clip
Space），并最终通过透视除法转换为归一化设备坐标（Normalized Device
Coordinates, NDC）。 - 这个变换定义了视锥体（View
Frustum），它决定了哪些部分的场景被渲染到屏幕上。 -
投影变换可以是透视投影（Perspective Projection）或正交投影（Orthographic
Projection）。 \### 引擎中的MVP变换
以Unity为例。在Unity中，我们可以通常可以直接拿到MVP变换矩阵来进行MVP矩阵变换。下面是Lit的部分源码

``` c
struct Attributes  
{  
    ...
    float4 positionOS : POSITION;  
    ...
};

struct Varyings  
{  
    ...
    float4 positionCS : SV_POSITION;  
    ...
};

VertexPositionInputs GetVertexPositionInputs(float3 positionOS)  
{  
    VertexPositionInputs input;  
    input.positionWS = TransformObjectToWorld(positionOS);  
    input.positionVS = TransformWorldToView(input.positionWS);  
    input.positionCS = TransformWorldToHClip(input.positionWS);  
  
    float4 ndc = input.positionCS * 0.5f;  
    input.positionNDC.xy = float2(ndc.x, ndc.y * _ProjectionParams.x) + ndc.w;  
    input.positionNDC.zw = input.positionCS.zw;  
  
return input;  
}

Varyings LitPassVertex(Attributes input)  
{
    ...
    VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);
    ...
#if defined(REQUIRES_WORLD_SPACE_POS_INTERPOLATOR)  
    output.positionWS = vertexInput.positionWS;  
#endif
    output.positionCS = vertexInput.positionCS;  
    return output;
}
```

不难发现，输入到顶点着色器（VS
Input）的位置坐标POSITION是处于模型空间，对应的是变量positionOS，最后输出到SV_POSITION的是裁剪空间坐标positionCS。
而在Lit的顶点着色器中，positionCS经过了世界空间、视图空间、裁剪空间的转换，但并没有计算NDC空间的坐标（所以后续我们也不需要进行NDC空间的转换）。
所以，RenderDoc中的VS Input通常代表的是模型空间下的Mesh信息，VS
Output通常代表的是裁剪空间下的Mesh信息（未经过NDC空间转换）。 \##
利用变换矩阵还原世界坐标 现在我们已经知道了，VS
Input中POSITION通过MVP变换（不包括NDC变换）可以得到VS
Output中的SV_POSITION。
那么想要得到模型的世界空间坐标，我们可以有两种做法 -
利用M矩阵进行还原（从模型空间转换到世界空间） -
利用VP矩阵进行还原（从屏幕空间转换到世界空间）
但无论是利用M矩阵还是VP矩阵还原，我们本质上是要拿到平移旋转缩放的信息。这是因为我们后续需要做的操作是拿到了模型空间的模型后，先将模型导入到引擎中，然后通过改变模型的平移旋转缩放值的方式来对场景进行还原。
第一种方法得到的世界空间坐标会更加准确，但缺点是M矩阵在很多情况下很难提取，并且大多数的模型的M矩阵会不一样。第二种方法得到世界空间坐标会有很小的误差（在计算逆矩阵时所导致的误差），尽管VP矩阵在一些情况下提取起来也不方便，但是优点在于在同一帧的情况下，场景中的物体所使用的的VP矩阵通常是不变的，所以此时我们可以人工找到VP矩阵的位置，以此来对世界空间的坐标进行还原。
\### 利用M矩阵还原
在某些情况下，我们截帧是可以截到带有标识的M矩阵的，如下图所示。
\![\[Pasted image 20231130150635.png\]\]
此时可以准确定位到M矩阵所在的CBuffer位置，拿到M矩阵信息。
但是在很多情况下，我们很难准确定位M矩阵的位置，这时就需要反编译DXBC的代码反推出M矩阵的位置，在这里我尝试过这种做法，实现起来难度很大，且识别准确率不高，不是很推荐。
在能够顺利拿到M矩阵的情况下，就可以对M矩阵进行平移旋转缩放信息的提取。下面是提取平移旋转缩放信息的示例代码。

``` python
def extractPosition(matrix):  
    position = [matrix[0, 3], matrix[1, 3], matrix[2, 3]]  
    return position
    
def extractRotation(matrix):  
    forward = [matrix[0, 2], matrix[1, 2], matrix[2, 2]]  
    upwards = [matrix[0, 1], matrix[1, 1], matrix[2, 1]]  
    return lookRotation(forward, upwards)
    
def extractScale(matrix):  
    s1 = magnitude(matrix[0, 0], matrix[1, 0], matrix[2, 0], matrix[3, 0])  
    s2 = magnitude(matrix[0, 1], matrix[1, 1], matrix[2, 1], matrix[3, 1])  
    s3 = magnitude(matrix[0, 2], matrix[1, 2], matrix[2, 2], matrix[3, 2])  
    return scale

def magnitude(e, b, c, d):  
    e = e * e  
    b = b * b  
    c = c * c  
    d = d * d  
    s = e + b + c + d  
    return math.sqrt(s)

def lookRotation(forward, up):  
    forward = normalize(forward)  
    vector = normalize(forward)  
    vector2 = normalize(np.cross(up, vector))  
    vector3 = np.cross(vector, vector2)  
    m00 = vector2[0]  
    m01 = vector2[1]  
    m02 = vector2[2]  
    m10 = vector3[0]  
    m11 = vector3[1]  
    m12 = vector3[2]  
    m20 = vector[0]  
    m21 = vector[1]  
    m22 = vector[2]  
    num8 = (m00 + m11) + m22  
    quaternion = FbxQuaternion(0.0, 0.0, 0.0, 1)  
    if num8 > 0:  
        num = float(math.sqrt(num8 + 1.0))  
        quaternion.SetAt(3, num * 0.5)  
        num = 0.5 / num  
        quaternion.SetAt(0, (m12 - m21) * num)  
        quaternion.SetAt(1, (m20 - m02) * num)  
        quaternion.SetAt(2, (m01 - m10) * num)  
        return quaternion  
    if (m00 >= m11) and (m00 >= m22):  
        num7 = float(math.sqrt(((1.0 + m00) - m11) - m22))  
        num4 = 0.5 / num7  
        quaternion.SetAt(0, 0.5 * num7)  
        quaternion.SetAt(1, (m01 + m10) * num4)  
        quaternion.SetAt(2, (m02 + m20) * num4)  
        quaternion.SetAt(3, (m12 - m21) * num4)  
        return quaternion  
    if m11 > m22:  
        num6 = float(math.sqrt(((1.0 + m11) - m00) - m22))  
        num3 = 0.5 / num6  
        quaternion.SetAt(0, (m10 + m01) * num3)  
        quaternion.SetAt(1, 0.5 * num6)  
        quaternion.SetAt(2, (m21 + m12) * num3)  
        quaternion.SetAt(3, (m20 - m02) * num3)  
        return quaternion  
    num5 = float(math.sqrt(((1.0 + m22) - m00) - m11))  
    num2 = 0.5 / num5  
    quaternion.SetAt(0, (m20 + m02) * num2)  
    quaternion.SetAt(1, (m21 + m12) * num2)  
    quaternion.SetAt(2, 0.5 * num5)  
    quaternion.SetAt(3, (m01 - m10) * num2)  
    return quaternion
```

拿到平移旋转缩放信息后，我们可以将其写入fbx文件中，以便导入unity中时transform会带有此信息。

``` python
def SaveAsFbx(DataFrame, saveName):  
    ...
    newNode.LclScaling.Set(FbxDouble3(scale[0], scale[1], scale[2]))  
    newNode.LclTranslation.Set(FbxDouble3(position[0], position[1], position[2]))  
    newNode.LclRotation.Set(FbxDouble3(rotation[0], rotation[1], rotation[2]))
    ...
```

### 利用VP矩阵还原

同样的，在某些情况下，我们是可以截帧可以截到带有标识的VP矩阵或是VP矩阵的逆。如下图所示。
\![\[Pasted image 20231214113628.png\]\]
而大部分情况下呢，我们是很难直接找到VP矩阵的位置的。
但由于VP矩阵在同一帧的情况下（或者说在一个rdc文件内），矩阵的值通常是相等的，所以这时我们可以通过阅读DXBC源码，判断VP矩阵所存的CBuffer位置。
如下面的在DXBC源码里，VP矩阵被存到了cb2\[17\]、cb2\[18\]、cb2\[19\]、cb\[20\]。

``` c
vs_5_0
      dcl_globalFlags refactoringAllowed
      dcl_constantbuffer cb0[51], immediateIndexed
      dcl_constantbuffer cb1[7], immediateIndexed
      dcl_constantbuffer cb2[10], immediateIndexed
      dcl_constantbuffer cb3[21], immediateIndexed
      dcl_input v0.xyz
      dcl_input v1.xyzw
      dcl_input v2.xyz
      dcl_input v3.xy
      dcl_input v4.xy
      dcl_input v5.xyzw
      dcl_output_siv o0.xyzw, position
      dcl_output o1.xyzw
      dcl_output o2.xyzw
      dcl_output o3.xyzw
      dcl_output o4.xyzw
      dcl_output o5.xyzw
      dcl_output o6.xyzw
      dcl_temps 5
   0: mul r0.xyzw, v0.yyyy, cb2[1].xyzw
   1: mad r0.xyzw, cb2[0].xyzw, v0.xxxx, r0.xyzw
   2: mad r0.xyzw, cb2[2].xyzw, v0.zzzz, r0.xyzw
   3: add r0.xyzw, r0.xyzw, cb2[3].xyzw//这里是乘以M矩阵，模型空间到世界空间的变换
   4: mul r1.xyzw, r0.yyyy, cb3[18].xyzw
   5: mad r1.xyzw, cb3[17].xyzw, r0.xxxx, r1.xyzw
   6: mad r1.xyzw, cb3[19].xyzw, r0.zzzz, r1.xyzw
   7: mad r0.xyzw, cb3[20].xyzw, r0.wwww, r1.xyzw//乘以VP矩阵，世界空间到裁剪空间的变换
   8: mov o0.xyzw, r0.xyzw
   9: eq r1.x, cb0[49].y, l(0)
  10: movc r1.xy, r1.xxxx, v3.xyxx, v4.xyxx
  11: mad o1.zw, r1.xxxy, cb0[50].xxxy, cb0[50].zzzw
  12: mad o1.xy, v3.xyxx, cb0[45].xyxx, cb0[45].zwzz
  13: dp3 r1.y, v2.xyzx, cb2[4].xyzx
  14: dp3 r1.z, v2.xyzx, cb2[5].xyzx
  15: dp3 r1.x, v2.xyzx, cb2[6].xyzx
  16: dp3 r1.w, r1.xyzx, r1.xyzx
  17: rsq r1.w, r1.w
  18: mul r1.xyz, r1.wwww, r1.xyzx
  19: mul r2.xyz, v1.yyyy, cb2[1].yzxy
  20: mad r2.xyz, cb2[0].yzxy, v1.xxxx, r2.xyzx
  21: mad r2.xyz, cb2[2].yzxy, v1.zzzz, r2.xyzx
  22: dp3 r1.w, r2.xyzx, r2.xyzx
  23: rsq r1.w, r1.w
  24: mul r2.xyz, r1.wwww, r2.xyzx
  25: mul r3.xyz, r1.xyzx, r2.xyzx
  26: mad r3.xyz, r1.zxyz, r2.yzxy, -r3.xyzx
  27: mul r1.w, v1.w, cb2[9].w
  28: mul r3.xyz, r1.wwww, r3.xyzx
  29: mov o2.y, r3.x
  30: mul r4.xyz, v0.yyyy, cb2[1].xyzx
  31: mad r4.xyz, cb2[0].xyzx, v0.xxxx, r4.xyzx
  32: mad r4.xyz, cb2[2].xyzx, v0.zzzz, r4.xyzx
  33: add r4.xyz, r4.xyzx, cb2[3].xyzx
  34: mov o2.w, r4.x
  35: mov o2.x, r2.z
  36: mov o2.z, r1.y
  37: mov o3.x, r2.x
  38: mov o4.x, r2.y
  39: mov o3.z, r1.z
  40: mov o4.z, r1.x
  41: mov o3.w, r4.y
  42: mov o4.w, r4.z
  43: mov o3.y, r3.y
  44: mov o4.y, r3.z
  45: mul r0.y, r0.y, cb1[6].x
  46: mul r1.xzw, r0.xxwy, l(0.5000, 0.0000, 0.5000, 0.5000)
  47: mov o5.zw, r0.zzzw
  48: add o5.xy, r1.zzzz, r1.xwxx
  49: mad r0.x, v5.w, l(-2.0000), l(1.0000)
  50: mad o6.w, cb0[42].x, r0.x, v5.w
  51: mov o6.xyz, v5.xyzx
  52: ret
```

通常来说，存储这些矩阵的CBuffer都是可以在RenderDoc的Pipeline
State界面的Vertex Shader下面找到。
在定位好CBuffer的位置后，可以是用RenderDoc提供的API在CBuffer中提取矩阵信息。下面是用来提取CBuffer的示例代码。

``` python
def get_constant_buffer(self, buffer_name):
    buffer_index = -1  
    buffer_name = buffer_name.strip()  
    for i in range(0, len(self.shader_reflection.constantBlocks)):  
        cb_name = self.shader_reflection.constantBlocks[i].name.strip()  
        if cb_name == buffer_name:  
            buffer_index = i  
    if buffer_index == -1:  
        print("没有找到Cbuffer")  
        return None  
    pipeState = self.state.GetGraphicsPipelineObject()  
    entry = self.state.GetShaderEntryPoint(rd.ShaderStage.Vertex)  
    cbuffer = self.state.GetConstantBuffer(rd.ShaderStage.Vertex, buffer_index, 0)  
    cbuffer_vars = self.controller.GetCBufferVariableContents(pipeState, self.shader_reflection.resourceId,  
                                                              rd.ShaderStage.Vertex,  
                                                              entry, buffer_index, cbuffer.resourceId, 0, 0)  
    cbuffer_vars_value = [] 
    for i in cbuffer_vars:  
        cbuffer_vars_value.append(list(i.value.f32v[0:4]))  
    return cbuffer_vars_value
```

在拿到VP矩阵后，还需要计算VP矩阵的逆。然后用positionCS乘以VP矩阵的逆可以得到positionWS。最后用positionWS乘以positionOS的逆可以得到M矩阵，从而可以计算出平移缩放旋转值（上一小节有介绍）。
\# 批量导出
批量导出的话，我们可以让用户输入EventID或是ActionID的起始ID和终止ID。然后通过遍历这些ID，通过ID来查找Action，找到Action后用单个模型导出的逻辑进行导出。
在批量导出的过程中需要注意的是，在DX11或DX12平台下截的帧，需要对使用DrawIndexedInstanced和DrawIndexInstancedIndirect的Action进行特殊处理，因为这两者都在一个Action（或者说是DrawCall）下画了多个不同位置的同一个mesh，也就是说，他们的mesh信息相同，但是平移旋转缩放信息可能不同。
在使用M矩阵进行还原时，就要注意这一个Action下就会要用到多个不同的M矩阵（提取的时候会很麻烦）。在使用VP矩阵进行还原时，会发现RenderDoc提供了各个不同Instance的VS
Output的信息，通过RenderDoc提供的API进行获取即可。
