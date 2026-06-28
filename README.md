# 旋转与变换
| 202311081051 | 王婧怡 | 计算机科学与技术 |
| --- | --- | --- |


# 实验目的
1. 深入理解 3D 空间中的坐标变换流程（模型-视图-投影/MVP(Model-View-Projection)变换）。
2. 独立推导并用代码实现模型变换（Model）、视图变换（View）和投影变换（Projection）矩阵。
3. 掌握面向数据编程框架 Taichi 的基本语法与矩阵操作。

# 实验原理
##  三维图形坐标变换流程
在现代图形渲染中，一个物体通常需要经过如下坐标变换：

```c
模型坐标(Model Space)
↓ Model Matrix
世界坐标(World Space)
↓ View Matrix
相机坐标(View Space)
↓ Projection Matrix
裁剪坐标(Clip Space)
↓ Perspective Divide
NDC坐标
↓ Viewport Transform
屏幕坐标(Screen Space)
```

整个过程可以表示为：\[ P_{screen}=Projection\times View\times Model\times P \]

由于本实验采用列向量，因此矩阵按照右乘顺序进行计算：

\[ MVP=M_{projection}\times M_{view}\times M_{model} \]



## Model模型变换
模型变换用于改变物体的位置、姿态和大小。

本实验要求实现绕 Z 轴旋转。

设旋转角度为 θ，则旋转矩阵为：

\[ R_z= \begin{bmatrix} \cos\theta&-\sin\theta&0&0\\ \sin\theta&\cos\theta&0&0\\ 0&0&1&0\\ 0&0&0&1 \end{bmatrix} \]

由于 Python 三角函数采用弧度，因此程序首先完成\[ \theta=\theta\times\frac{\pi}{180} \]

随后利用

```plain
c = ti.cos(rad)
s = ti.sin(rad)
```

构造旋转矩阵。

当按下 A 或 D 键时，仅修改 angle，模型即可围绕 Z 轴连续旋转。



## View视图变换
View Matrix 的作用是把摄像机移动到世界坐标原点。

实验中摄像机位置为Eye=(0,0,5)

因此需要把整个场景反方向移动：\[ T= \begin{bmatrix} 1&0&0&-x\\ 0&1&0&-y\\ 0&0&1&-z\\ 0&0&0&1 \end{bmatrix} \]

即\[ T= \begin{bmatrix} 1&0&0&0\\ 0&1&0&0\\ 0&0&1&-5\\ 0&0&0&1 \end{bmatrix} \]

经过该矩阵后，相机位于坐标原点。



## Projection透视投影
透视投影是本实验最重要的部分。

首先根据视场角计算近平面的边界：

\[ t=\tan\left(\frac{fov}{2}\right)|n| \]

\[ b=−t \]

\[ r=aspect×t \]

\[ l=−r \]

实验采用

+  FOV=45° 
+  Aspect=1 
+  Near=0.1 
+  Far=50 

随后按照实验要求，先完成

### 透视压缩矩阵
\[ M_{persp\rightarrow ortho} = \begin{bmatrix} n&0&0&0\\ 0&n&0&0\\ 0&0&n+f&-nf\\ 0&0&1&0 \end{bmatrix} \]

作用是把视锥体压缩成长方体。



### 正交投影矩阵
正交投影由平移矩阵和缩放矩阵组成：

缩放：\[ S= \begin{bmatrix} \frac2{r-l}&0&0&0\\ 0&\frac2{t-b}&0&0\\ 0&0&\frac2{n-f}&0\\ 0&0&0&1 \end{bmatrix} \]

平移：\[ T= \begin{bmatrix} 1&0&0&-\frac{r+l}{2}\\ 0&1&0&-\frac{t+b}{2}\\ 0&0&1&-\frac{n+f}{2}\\ 0&0&0&1 \end{bmatrix} \]

最终得到：\[ M_{projection} = M_{ortho} \times M_{persp\rightarrow ortho} \]



## 透视除法
经过 MVP 变换后，顶点变为(x,y,z,w)

必须进行\[ (x',y',z') = \left( \frac{x}{w}, \frac{y}{w}, \frac{z}{w} \right) \]得到 NDC 坐标。

程序中对应：

```plain
v_ndc = v_clip / v_clip[3]
```

随后映射到 GUI：

```plain
screen=(ndc+1)/2
```

即可得到屏幕坐标。





# 实验任务实现
## 模型变换矩阵 get_model_matrix(angle)
程序首先将输入角度由角度制转换为弧度制：

```plain
rad = angle * math.pi / 180.0
```

随后利用 Taichi 提供的三角函数计算旋转矩阵中的余弦和正弦值：

```plain
c = ti.cos(rad)
s = ti.sin(rad)
```

根据绕 Z 轴旋转矩阵的数学表达式构建四阶齐次矩阵：\[ M_{model}= \begin{bmatrix} \cos\theta &-\sin\theta&0&0\\ \sin\theta&\cos\theta&0&0\\ 0&0&1&0\\ 0&0&0&1 \end{bmatrix} \]

代码如下：

```plain
return ti.Matrix([
    [c, -s, 0.0, 0.0],
    [s,  c, 0.0, 0.0],
    [0.0,0.0,1.0,0.0],
    [0.0,0.0,0.0,1.0]
])
```

当用户按下 A 或 D 键时，程序分别增加或减少旋转角度，并重新计算模型矩阵，从而实现三角形围绕 Z 轴连续旋转。



## 视图变换矩阵 get_view_matrix(eye_pos)
本实验中相机位置设置为：

```plain
eye_pos = ti.Vector([0.0, 0.0, 5.0])
```

因此，需要将整个场景沿相反方向平移，即分别减去相机坐标：

```plain
return ti.Matrix([
    [1.0,0.0,0.0,-eye_pos[0]],
    [0.0,1.0,0.0,-eye_pos[1]],
    [0.0,0.0,1.0,-eye_pos[2]],
    [0.0,0.0,0.0,1.0]
])
```

对应的视图矩阵为：\[ M_{view}= \begin{bmatrix} 1&0&0&-x_e\\ 0&1&0&-y_e\\ 0&0&1&-z_e\\ 0&0&0&1 \end{bmatrix} \]

其中\[ (x_e,y_e,z_e)=(0,0,5) \]



## 透视投影矩阵 get_projection_matrix()
采用两步构造方法，先进行透视到正交变换，再进行正交投影。

### 计算近平面边界
根据输入参数计算近平面的上下左右边界。

先将视场角转换为弧度：

```plain
fov_rad = eye_fov * math.pi / 180.0
```

随后根据几何关系计算：

```plain
t = ti.tan(fov_rad / 2.0) * ti.abs(n)
b = -t
r = aspect_ratio * t
l = -r
```

由于实验采用右手坐标系，相机沿 −Z 方向观察，程序将输入的近远平面距离转换为负值，保证投影矩阵满足图形学中的坐标定义：

```plain
n = -zNear
f = -zFar
```



### 构造透视到正交矩阵
将透视平截头体转换成长方体，构造透视压缩矩阵：

```plain
M_p2o = ti.Matrix([
    [n,0.0,0.0,0.0],
    [0.0,n,0.0,0.0],
    [0.0,0.0,n+f,-n*f],
    [0.0,0.0,1.0,0.0]
])
```

对应数学表达式为：\[ M_{persp\rightarrow ortho}= \begin{bmatrix} n&0&0&0\\ 0&n&0&0\\ 0&0&n+f&-nf\\ 0&0&1&0 \end{bmatrix} \]

### 构造正交投影矩阵
完成透视压缩后，将长方体缩放并平移到标准立方体 \[ [-1,1]^3 \]。

分别构造缩放矩阵和平移矩阵M_ortho_scale，负责完成各坐标轴方向的归一化：

```plain
[2/(r-l),0,0,0]
[0,2/(t-b),0,0]
[0,0,2/(n-f),0]
```

随后构造平移矩阵M_ortho_trans，将长方体中心移动至坐标原点。

最后两者相乘得到正交投影矩阵：M_ortho = M_ortho_scale @ M_ortho_trans

整个投影矩阵为：return M_ortho @ M_p2o

即\[ M_{projection} = M_{ortho} M_{persp\rightarrow ortho} \]



## MVP 变换及屏幕坐标映射
在 `compute_transform()` 函数中，分别获取模型矩阵、视图矩阵和投影矩阵：

```plain
model = get_model_matrix(angle)
view = get_view_matrix(eye_pos)
proj = get_projection_matrix(...)
```

然后按照列向量的计算规则组合得到 MVP 矩阵：

```plain
mvp = proj @ view @ model
```

程序将每个顶点补充齐次坐标：

```plain
v4 = ti.Vector([v[0], v[1], v[2], 1.0])
```

经过 MVP 变换后得到裁剪空间坐标：

```plain
v_clip = mvp @ v4
```

然后进行透视除法：

```plain
v_ndc = v_clip / v_clip[3]
```

将顶点转换为标准设备坐标（NDC）。

最后，将 NDC 坐标映射到 GUI 的 [0,1][0,1][0,1] 坐标系：

```plain
screen_coords[i][0] = (v_ndc[0] + 1.0) / 2.0
screen_coords[i][1] = (v_ndc[1] + 1.0) / 2.0
```

完成三维坐标到二维屏幕坐标的转换，并利用 `gui.line()` 绘制三角形三条边。当用户按下 A 或 D** **键时，程序重新计算模型矩阵和 MVP 矩阵，实现三角形绕 Z 轴的实时旋转，达到实验要求的交互显示效果。



# 实验结果
程序运行后成功创建一个 700×700 的 GUI 窗口。

实验结果如下：

1.  三角形能够正确显示在窗口中央。 
2.  三条边分别绘制为： 红色 、绿色 、蓝色 
3. 便于观察旋转方向。按下 A 键时，三角形逆时针旋转。；按下 D 键时，三角形顺时针旋转。 ；按下 ESC 键程序正常退出。 
<img width="1052" height="1098" alt="录屏_20260629_010252" src="https://github.com/user-attachments/assets/01d518f1-228b-45c2-981b-264234ee245a" />
<img width="1052" height="1098" alt="D" src="https://github.com/user-attachments/assets/6d3ea5f1-206f-4809-982a-049adeb2fb62" />
