# C++生成球体网格

[OpenGL Sphere (songho.ca)](http://www.songho.ca/opengl/gl_sphere.html)

[C++ 生成球体网格 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/364320044)

## UV sphere

UV 球体是许多三维建模工具如 Blender 中的标准几何体。网格沿着经线和纬线排布，极点周围是三角形，其他位置是四边形。

代码很容易理解，先添加所有的顶点，然后添加对应的三角形面和四边形面。



> [OpenGL Sphere (songho.ca)](http://www.songho.ca/opengl/gl_sphere.html)中所定义的如下

球面是一个3D闭合表面，表面上所有的点与球心的距离（半径）相等。等式如下：

![equation of sphere](http://www.songho.ca/opengl/files/gl_sphere_eq01.png)
$$
x^{2} + y^{2} + z^{2} = r^{2}
$$
由于我们无法在球体上绘制所有点，因此我们只能通过将球体除以扇区sector（longitude经度）和堆栈stacks（latitude纬度）来采样有限数量的点。 然后将这些采样点连接在一起以形成球体的表面。 

![Parametric equation of a sphere](http://www.songho.ca/opengl/files/gl_sphere01.png)

![Sectors and stacks of a sphere](http://www.songho.ca/opengl/files/gl_sphere02.png)

球面上的任意点 (x, y, z) 可以通过具有相应扇形角 $\theta$ 和stack角 $\phi$ 的参数方程来计算。 
$$
\begin{gather}
x =& (r \cdot \cos{\phi} \cdot \cos{\theta}) \\
y =& (r \cdot \cos{\phi} \cdot \sin{\theta}) \\
z =& r \cdot \sin{\phi}
\end{gather}
$$
扇形角度的范围是从 0 到 360 度，堆叠角度从 90（顶部）到 -90 度（底部）。 每个步骤的扇区和堆叠角可以通过以下方式计算； 
$$
\begin{gather}
\theta =& 2\pi \cdot \frac{sectorStep}{sectorCount} \\
\phi =& \frac{\pi}{2} - \pi \cdot \frac{stackStep}{stackCount}
\end{gather}
$$


以下 C++ 代码为给定的半径、扇区和堆栈生成球体的所有顶点。 它还创建其他顶点属性； 表面法线和纹理坐标。

```c++
// clear memory of prev arrays
std::vector<float>().swap(vertices);
std::vector<float>().swap(normals);
std::vector<float>().swap(texCoords);

float x, y, z, xy;                              // vertex position
float nx, ny, nz, lengthInv = 1.0f / radius;    // vertex normal
float s, t;                                     // vertex texCoord

float sectorStep = 2 * PI / sectorCount;
float stackStep = PI / stackCount;
float sectorAngle, stackAngle;

for(int i = 0; i <= stackCount; ++i)
{
    stackAngle = PI / 2 - i * stackStep;        // starting from pi/2 to -pi/2
    xy = radius * cosf(stackAngle);             // r * cos(u)
    z = radius * sinf(stackAngle);              // r * sin(u)

    // add (sectorCount+1) vertices per stack
    // the first and last vertices have same position and normal, but different tex coords
    for(int j = 0; j <= sectorCount; ++j)
    {
        sectorAngle = j * sectorStep;           // starting from 0 to 2pi

        // vertex position (x, y, z)
        x = xy * cosf(sectorAngle);             // r * cos(u) * cos(v)
        y = xy * sinf(sectorAngle);             // r * cos(u) * sin(v)
        vertices.push_back(x);
        vertices.push_back(y);
        vertices.push_back(z);

        // normalized vertex normal (nx, ny, nz)
        nx = x * lengthInv;
        ny = y * lengthInv;
        nz = z * lengthInv;
        normals.push_back(nx);
        normals.push_back(ny);
        normals.push_back(nz);

        // vertex tex coord (s, t) range between [0, 1]
        s = (float)j / sectorCount;
        t = (float)i / stackCount;
        texCoords.push_back(s);
        texCoords.push_back(t);
    }
}
```

为了在 OpenGL 中绘制球体的表面，我们必须对相邻顶点进行三角剖分以形成多边形。 可以使用单个三角形条来渲染整个球体。 但是，如果共享顶点具有不同的法线或纹理坐标，则不能使用单个三角形带。 

![vertex indices of sphere](http://www.songho.ca/opengl/files/gl_sphere03.png)

Stack中的每个扇区需要 2 个三角形。 如果当前stack的第一个顶点索引是k1，下一个stack是k2，那么2个三角形的顶点索引的逆时针顺序是； 

*k1 ⟶ k2 ⟶ k1+1*
*k1+1 ⟶ k2 ⟶ k2+1*

但是，顶部和底部堆栈每个扇区只需要一个三角形。 生成球体所有三角形的代码片段可能如下所示：

```c++
// generate CCW index list of sphere triangles
// k1--k1+1
// |  / |
// | /  |
// k2--k2+1
std::vector<int> indices;
std::vector<int> lineIndices;
int k1, k2;
for(int i = 0; i < stackCount; ++i)
{
    k1 = i * (sectorCount + 1);     // beginning of current stack
    k2 = k1 + sectorCount + 1;      // beginning of next stack

    for(int j = 0; j < sectorCount; ++j, ++k1, ++k2)
    {
        // 2 triangles per sector excluding first and last stacks
        // k1 => k2 => k1+1
        if(i != 0)
        {
            indices.push_back(k1);
            indices.push_back(k2);
            indices.push_back(k1 + 1);
        }

        // k1+1 => k2 => k2+1
        if(i != (stackCount-1))
        {
            indices.push_back(k1 + 1);
            indices.push_back(k2);
            indices.push_back(k2 + 1);
        }

        // store indices for lines
        // vertical lines for all stacks, k1 => k2
        lineIndices.push_back(k1);
        lineIndices.push_back(k2);
        if(i != 0)  // horizontal lines except 1st stack, k1 => k+1
        {
            lineIndices.push_back(k1);
            lineIndices.push_back(k1 + 1);
        }
    }
}
```



这种方法生成的球面会有如下问题：

由于此 C++ 类使用圆柱纹理映射，因此在北极和南极区域存在挤压/变形。 这个问题可以使用 Icosphere 或 Cubesphere 来解决。 

> 在songho.ca的文章和[catch 22 - Andreas Kahler's blog: Creating an icosphere mesh in code](http://blog.andreaskahler.com/2009/06/creating-icosphere-mesh-in-code.html)这篇文章中，介绍了如何生成Icosphere。内核思想是迭代的细分一个正二十面体 icosahedron。最后生成的是一个纯三角形网格，顶点分布及大小更加规则。