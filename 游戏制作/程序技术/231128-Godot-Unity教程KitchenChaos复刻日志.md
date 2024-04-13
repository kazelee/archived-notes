# Unity 教程 KitchenChaos 在 Godot 中复刻日志

项目基于 Godot 4.2 dev 版本，采用 C# 语言编写；考虑到 Unity 与 Godot 的诸多不同，一些参数会有所改变，很多功能不可能面面俱到，仅供参考。

项目地址：（待补）

教程视频：[Learn Unity Beginner/Intermediate 2023 (FREE COMPLETE Course - Unity Tutorial) - YouTube](https://www.youtube.com/watch?v=AmGSEH7QcDg)
<!-- B站搬运链接就不放了，播放会莫名其妙地卡顿，这里还是使用油管源，目录分得更细 -->

作者的网站：[Learn to make a Game with Unity! Beginners and Intermediates - Code Monkey (unitycodemonkey.com)](https://unitycodemonkey.com/kitchenchaoscourse.php)

参考笔记：[KitchenChaos笔记（类胡闹厨房游戏demo） (yuque.com)](https://www.yuque.com/wocaibuqinamochangdemingzi/akh7w4/ka1o8oxb7723ndg9?singleDoc#)

## 准备阶段

### Unity 素材转 Godot 格式

参考文章[Move From Unity to Godot Engine in Seconds –](https://gamefromscratch.com/move-from-unity-to-godot-engine-in-seconds/)，将原项目中的 Unity 格式素材转换成 Godot 可用的格式，转换后的素材已经在项目的 Assets 文件夹中了。

一些 Unity 与 Godot 的文件格式对比：

Unity | Godot | 补充
-- | -- | --
.fbx | .gltf/.mesh | 3D文件（Godot原生不支持fbx，需要插件转换）
.meta | .import | 素材导入的数据文件，包括id/路径等
.prefab | .tscn | Unity的预制件类似Godot的场景
.mat | .tres/.material | 纹理文件，Godot建议使用文本tres
.anim等 | .tres | Godot资源一般都是该格式，很多未必能通用

### Godot 3D 游戏制作相关

Unity 中新建 3D Object 可以类比成 Godot 的 MeshInstance3D（考虑单个有形状的物体；如果空形状可以使用 Node 3D，是多个物体选择可以用 GirdMap）

## 教程内容

### 01 游戏场景：摄像机、光线、环境（后处理）

使用 MeshInstance3D 添加地面（由于本游戏不考虑玩家跳跃和重力，所以不需要设置碰撞体，确保操控的玩家只在水平方向运动即可）

MSAA 抗锯齿：项目 - 项目设置 - 渲染 - 抗锯齿 - MSAA 3D（原教程设置为×8，个人觉得×4的效果足够好，所以选择了×4）

后处理：对应 WorldEnvironment 的 Adjustments，这里将 Contrast 调节为 1.1，Saturation 调节为 1.1；Glow 的 Bloom 调节为 0.05）

【补充】发觉 Godot 的光线阴影锯齿严重，调节 Light3D 的 blur 值做适当模糊化（调节为 2）

边缘黑边的效果，需要自行写 Shader 完成，步骤是在 3D 场景下新建 CanvasLayer，并新建子节点 ColorRect， 然后新建 shader，参考：[Color vignette - Godot Shaders](https://godotshaders.com/shader/color-vignetting/)

```shader
shader_type canvas_item;

uniform float vignette_intensity = 0.4;
uniform float vignette_opacity : hint_range(0.0, 1.0) = 0.5;
uniform vec4 vignette_rgb : source_color = vec4(0.0, 0.0, 0.0, 1.0);
uniform sampler2D SCREEN_TEXTURE : hint_screen_texture, filter_linear_mipmap;

float vignette(vec2 uv){
	uv *= 1.0 - uv.xy;
	float vignette = uv.x * uv.y * 15.0;
	return pow(vignette, vignette_intensity * vignette_opacity);
}

void fragment(){
	vec4 color = texture(SCREEN_TEXTURE, SCREEN_UV);
	vec4 text = texture(TEXTURE, UV);
	text.rgba *= (vignette_rgb.rgba);
	text.rgba *= (1.0 - vignette(UV));
	COLOR = vec4((text.rgb)*color.rgb,text.a);
}
```

<!-- 需要对代码进行微调，由于缺乏 shader 知识，且考虑到对实际效果的影响不大，所以略过这部分。
（画面的处理不是必须的，可以自行酌情调整；初学完全可以先跳过这部分内容） -->

其中，`uniform` 类似 `export` 导出变量，可以在右侧检查器中修改参数。这里设置 `vignette_intensity` 为 0.5，`vignette_opacity` 为 0.1.

### 02 角色控制器、动画、相机

新建 Character3D 节点作为 Player，然后在节点下面设置图像，确保逻辑与画面分离。

Godot 没有 Unity 的 InputManager 模块，这里直接使用 physics_process 监听输入。
