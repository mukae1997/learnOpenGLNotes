learn-opengl学习笔记

[TOC]

# Basic

## shader语法 

声明变量由三部分组成。

`uniform vec3 nameOfVar`

`uniform/in/out`：指示数据在流中的位置(?)（不知道术语）

`vec3`：数据类型

`nameOfVar` ：变量名，需唯一。

## 输入处理

```c++
// process all input: query GLFW whether relevant keys are pressed/released this frame and react accordingly
// ---------------------------------------------------------------------------------------------------------
void processInput(GLFWwindow *window)
{
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);

    if (glfwGetKey(window, GLFW_KEY_UP) == GLFW_PRESS)
    {
        mixValue += 0.001f; // change this value accordingly (might be too slow or too fast based on system hardware)
        if(mixValue >= 1.0f)
            mixValue = 1.0f;
    }
    if (glfwGetKey(window, GLFW_KEY_DOWN) == GLFW_PRESS)
    {
        mixValue -= 0.001f; // change this value accordingly (might be too slow or too fast based on system hardware)
        if (mixValue <= 0.0f)
            mixValue = 0.0f;
    }
}
```





# 图形渲染pipeline

1. 顶点着色器 vertex shader
2. shape(primitive) shader
3. geometry shader
4. rasterization
5. fragment shader
6. tests and blending



# hello, triangle

写入缓存数据过程：gen->bind->copy， 输入顶点数据发送给GPU

```c++
glGenBuffers(1, &EBO); // 创建缓冲
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO); // 
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW); // 把数据复制到缓冲中
```



shader 过程：create->bind->compile，指示了GPU如何在顶点和片段着色器中处理它



连接顶点数据和shader的桥梁：顶点属性vertex attribute.

```c++
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
```

通过这两行调用，决定内存中的数据应该如何解析，从而连接到顶点属性。



不使用VAO的绘图过程：

```c++
// 0. 复制顶点数组到缓冲中供OpenGL使用
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 1. 设置顶点属性指针
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
// 2. 当我们渲染一个物体时要使用着色器程序
glUseProgram(shaderProgram);
// 3. 绘制物体
someOpenGLFunctionThatDrawsOurTriangle();
```

使用VAO的绘图过程：

```c++
glUseProgram(shaderProgram); // 调用shader
glBindVertexArray(VAO); // 绑定VAO
glDrawArrays(GL_TRIANGLES, 0, 3); // 绘图
```

## glVertexAttribPointer()

设定如何parse数据。

## VAO, VBO, Buffer, EBO

`Buffer`：一块缓冲空间，不同于应用程序的地址空间。用于保存拷贝过来的绘图数据。

`VBO`：管理缓存空间的object。要通过它操作`Buffer`。

`VAO`：保存顶点属性的设置，通过保存`attribute ptr`（指向VBO相应数据）实现。

# Shader

输入：顶点属性。上限：

```
> Maximum nr of vertex attributes supported: 16
```

书写语言：`GLSL`.

基本语法：

- 用in和out指定输出输入。

  - 如果想把输入数据发送出去，必须新声明一个`out`变量再复制该`in`数据。

- 用`location`指定输入。

- 数据类型`uniform`

  - 全局变量
  - 使用`uniform`可以通过应用程序控制着色器。查询uniform地址不要求你之前使用过着色器程序，但是更新一个uniform之前你**必须**先使用程序（调用`glUseProgram`)，因为它是在当前激活的着色器程序中设置uniform的。控制在`renderLoop`中进行。
  ```c++
  int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor"); // ourColor 是 uniform变量

  // FRAG shader:
  #version 330 core
  out vec4 FragColor; 
  uniform vec4 ourColor; // 在OpenGL程序代码中设定这个变量
  void main()
  {
      FragColor = ourColor;
  }
  ```

  ​

特点：

**可传递性。**：只要一个输出变量与下一个着色器阶段的输入匹配，它就会传递下去。但在顶点和片段着色器中会有点不同

片段着色器的输出一定为`vec4`。

## shader和应用程序的配合

> 情景：在VBO中写入颜色数据，使得shader可以读取，免去设置多个uniform的麻烦。

应用程序：设定顶点属性的parsing

```c++
  // position attribute
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);
    // color attribute
    glVertexAttribPointer(1,// 第一个参数即为[location]
                          3, GL_FLOAT, GL_FALSE, 
                          6 * sizeof(float),  // 顶点间的stide
                          (void*)(3 * sizeof(float)));
						//初始offset，即第一个color vec的offset
    glEnableVertexAttribArray(1);
```

`shader program`：设定`in`的`layout`的`position`

```c++
#version 330 core
layout (location = 0) in vec3 aPos;   // 位置变量的属性位置值为 0 
layout (location = 1) in vec3 aColor; // 颜色变量的属性位置值为 1
out vec3 ourColor; // 向片段着色器输出一个颜色
void main(){
    gl_Position = vec4(aPos, 1.0);
    ourColor = aColor; // 将ourColor设置为我们从顶点数据那里得到的输入颜色
}
```



# Texture

```c++
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
```

使用程序：

gen-bind-setParameter-(loadData, 调用库，从图片文件到`unsigned char *`) - writedata-genMipmap

```c++
unsigned int texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
// 为当前绑定的纹理对象设置环绕、过滤方式
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);   
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
// 加载并生成纹理
int width, height, nrChannels;
unsigned char *data = stbi_load("container.jpg", &width, &height, &nrChannels, 0);
if (data)
{
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
    glGenerateMipmap(GL_TEXTURE_2D);
}
else
{
    std::cout << "Failed to load texture" << std::endl;
}
stbi_image_free(data);
```

## 与shader的配合

在数据vertices中加入 texture坐标：

```c++
float vertices[] = {
  // positions          // colors           // texture coords
  0.5f,  0.5f, 0.0f,   1.0f, 0.0f, 0.0f,   1.0f, 1.0f,  
        ...
```

设置parse方法：

```c++
// texture coord attribute
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(6 * sizeof(float)));
    glEnableVertexAttribArray(2);
```



在vertex shader中创建一个入口：

```c++
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aTexCoord; // 入口

out vec3 ourColor;
out vec2 TexCoord;

void main()
{
    gl_Position = vec4(aPos, 1.0);
    ourColor = aColor;
    TexCoord = aTexCoord;
}
```

在frag shader中，把纹理对象传给片段着色器，使用`uniform sample2D`对象。

```c++
#version 330 core
out vec4 FragColor;

in vec3 ourColor;
in vec2 TexCoord;

uniform sampler2D ourTexture;

void main()
{//第一个参数是纹理采样器，第二个参数是对应的纹理坐标
    FragColor = texture(ourTexture, TexCoord);
}
```

在应用程序中，先激活Shader program，再设定相应uniform的值。

使用glUniform1i，我们可以给纹理采样器分配一个**位置值**，这样的话我们能够在<u>一个片段着色器中设置多个纹理</u>。一个纹理的位置值通常称为一个纹理单元(Texture Unit)。一个纹理的默认纹理单元是0，它是默认的激活纹理单元，所以教程前面部分我们没有分配一个位置值。

```c++
// 第二个参数对应 glActiveTexture()的参数。
glUniform1i(glGetUniformLocation(ourShader.ID, "texture1"), 0);
glUniform1i(glGetUniformLocation(ourShader.ID, "texture2"), 1);
```

最后，在renderLoop中，在激活shader前激活texture。

```c++
 // bind textures on corresponding texture units
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, texture1);
        glActiveTexture(GL_TEXTURE1); // 也可写GL_TEXTURE0 + 1
        glBindTexture(GL_TEXTURE_2D, texture2);
```

## 坐标系问题

OpenGL要求y轴`0.0`坐标是在图片的底部的，但是图片的y轴`0.0`坐标通常在顶部。很幸运，`stb_image.h`能够在图像加载时帮助我们翻转y轴，只需要在加载任何图像前加入以下语句即可：

```
stbi_set_flip_vertically_on_load(true);
```

# 变换



```c++
// 给出待变换变量和 变换本身的变量表示
glm::vec4 vec(1.0f, 0.0f, 0.0f, 1.0f);
glm::mat4 trans;
// 调用位移变换：translate，生成变换矩阵trans。
trans = glm::translate(trans, glm::vec3(1.0f, 1.0f, 0.0f));
vec = trans * vec; // 得到变换后的向量
std::cout << vec.x << vec.y << vec.z << std::endl;

glm::mat4 trans;
trans = glm::rotate(trans, glm::radians(90.0f), glm::vec3(0.0, 0.0, 1.0));
trans = glm::scale(trans, glm::vec3(0.5, 0.5, 0.5)); 
```

## 和shader的配合

通过`uniform`传数据。

```c++
uniform mat4 transform;

void main()
{
    gl_Position = transform * vec4(aPos, 1.0f);
    TexCoord = vec2(aTexCoord.x, 1.0 - aTexCoord.y);
}

```

应用程序中设定`uniform`：

```c++
unsigned int transformLoc = glGetUniformLocation(ourShader.ID, "transform");
glUniformMatrix4fv(transformLoc,   // uniform的位置值
                   1, // 发送1个矩阵
                   GL_FALSE, 
                   glm::value_ptr(trans));

```

我们首先查询uniform变量的地址，然后用有`Matrix4fv`后缀的glUniform函数把矩阵数据发送给着色器。

第三个参数询问我们我们是否希望对我们的矩阵进行置换(Transpose)，也就是说交换我们矩阵的行和列。OpenGL开发者通常使用一种内部矩阵布局，叫做列主序(Column-major Ordering)布局。GLM的默认布局就是列主序，所以并不需要置换矩阵，我们填`GL_FALSE`。

最后一个参数是真正的矩阵数据，但是GLM并不是把它们的矩阵储存为OpenGL所希望接受的那种，因此我们要先用GLM的自带的函数value_ptr来变换这些数据。

# 坐标空间

一个顶点坐标将会根据以下过程被变换到裁剪坐标：

$$V_{clip}=M_{projection}⋅M_{view}⋅M_{model}⋅V_{local}$$

最后的顶点应该被赋值到顶点着色器中的gl_Position，OpenGL将会自动进行透视除法和裁剪。

- model matrix: 放置被摄物
- view matrix：放置camera（包括位置，角度）
- projection matrix：成像过程（将会去除不在取景范围内的）



## local space

## world space 

## view space 

[projection matrix， 标准化到[-1,1]] 

由投影矩阵创建的**观察箱**(Viewing Box)被称为平截头体(Frustum)，每个出现在平截头体范围内的坐标都会最终出现在用户的屏幕上。平截头体定义了可见的坐标，它由由**宽、高、近(Near)平面和远(Far)平面**所指定。任何出现在近平面之前或远平面之后的坐标都会被裁剪掉。

### 正射投影(无透视)

要创建一个正射投影矩阵，我们可以使用GLM的内置函数`glm::ortho`：

```c++
glm::ortho(0.0f, 800.0f, 0.0f, 600.0f, 0.1f, 100.0f);
// ortho(left, right, bottom, top, near, far)
```

这个投影矩阵会将处于这些x，y，z值范围内的坐标变换为标准化设备坐标。

### 透视投影

这个投影矩阵将给定的平截头体范围映射到裁剪空间，除此之外还修改了每个顶点坐标的w值，从而使得离观察者越远的顶点坐标w分量越大。被变换到裁剪空间的坐标都会在-w到w的范围之间（任何大于这个范围的坐标都会被裁剪掉）。OpenGL要求所有可见的坐标都落在-1.0到1.0范围内，作为顶点着色器最后的输出，因此，一旦坐标在裁剪空间内之后，透视除法就会被应用到裁剪空间坐标上：$out=(x/w，y/w，z/w)$.

顶点坐标的每个分量都会除以它的w分量，距离观察者越远顶点坐标就会越小。

在GLM中可以这样创建一个透视投影矩阵：

```c++
glm::mat4 proj = glm::perspective(glm::radians(45.0f), // fov
                                  (float)width/(float)height, 
                                  0.1f, 
                                  100.0f);
```

它的第一个参数定义了 fov 的值，它表示的是视野(Field of View)，并且设置了观察空间的大小。如果想要一个真实的观察效果，它的值通常设置为**45.0f**，但想要一个末日风格的结果你可以将其设置一个更大的值。

第二个参数设置了宽高比，由视口的宽除以高所得。

第三和第四个参数设置了平截头体的**近**和**远**平面。我们通常设置近距离为0.1f，而远距离设为100.0f。所有在近平面和远平面内且处于平截头体内的顶点都会被渲染。



## clip space

## screen space

OpenGL然后对**裁剪坐标**执行**透视除法**从而将它们变换到**标准化设备坐标**。OpenGL会使用glViewPort内部的参数来将标准化设备坐标映射到**屏幕坐标**，每个坐标都关联了一个屏幕上的点

## 实现3D

向深度轴的正轴（朝屏幕外的方向）旋转X轴：

```c++
glm::mat4 model;
model = glm::rotate(model, glm::radians(-55.0f), glm::vec3(1.0f, 0.0f, 0.0f));
```

将坐标系往里平移：

```c++
glm::mat4 view;
// 注意，我们将矩阵向我们要进行移动场景的反方向移动。
view = glm::translate(view, glm::vec3(0.0f, 0.0f, -3.0f));
```

定义投影矩阵：

```c++
glm::mat4 projection;
projection = glm::perspective(glm::radians(45.0f), screenWidth / screenHeight, 0.1f, 100.0f);
```

传入Shader:

```c++
#version 330 core
layout (location = 0) in vec3 aPos;
...
uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
    // 注意乘法要从右向左读
    gl_Position = projection * view * model * vec4(aPos, 1.0);
    ...
}
...
// 应用程序：
    int modelLoc = glGetUniformLocation(ourShader.ID, "model"));
glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
... // 观察矩阵和投影矩阵与之类似
```

### 深度测试

OpenGL存储它的所有深度信息于一个Z缓冲(Z-buffer)中，也被称为深度缓冲(Depth Buffer)。GLFW会自动为你生成这样一个缓冲（就像它也有一个颜色缓冲来存储输出图像的颜色）。深度值存储在每个片段里面（作为片段的**z**值），当片段想要输出它的颜色时，OpenGL会将它的深度值和z缓冲进行比较，如果当前的片段在其它片段之后，它将会被丢弃，否则将会覆盖。这个过程称为深度测试(Depth Testing)，它是由OpenGL自动完成的。

```c++
// render loop 外设定即可
glEnable(GL_DEPTH_TEST);

// render loop..
// 在每次渲染迭代之前清除深度缓冲（否则前一帧的深度信息仍然保存在缓冲中）
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
```

让立方体随时间旋转：

```c++
model = glm::rotate(model, (float)glfwGetTime() * glm::radians(50.0f), glm::vec3(0.5f, 1.0f, 0.0f));
// 第三个参数指定绕哪条轴旋转
```

# 调整camera

实质是根据所需移动计算view矩阵。可以做到放缩，旋转，平移等镜头效果。

设定摄像机需要的4个要素：

- 摄像机位置。世界坐标。
- Direction vector。观察目标指向摄像机的向量。

```c++
glm::vec3 cameraTarget = glm::vec3(0.0f, 0.0f, 0.0f);
// 摄像机位置减去观察目标位置，得到指向摄像机的向量。
glm::vec3 cameraDirection = glm::normalize(cameraPos - cameraTarget);
```

- 右轴，代表摄像机空间x轴正方向。先定义一个*上向量*指向世界坐标系的y正轴，再叉乘direction vector即得到右轴。两个向量叉乘的结果会同时垂直于两向量，因此我们会得到指向x轴正方向的那个向量（如果我们交换两个向量叉乘的顺序就会得到相反的指向x轴负方向的向量）：

```c++
glm::vec3 up = glm::vec3(0.0f, 1.0f, 0.0f); 
glm::vec3 cameraRight = glm::normalize(glm::cross(up, cameraDirection));
```

- 上轴。右向量和方向向量叉乘。

```c++
glm::vec3 cameraUp = glm::cross(cameraDirection, cameraRight);
```

过程如图：![](camera_axes.png)

## LookAt矩阵

```c++
glm::mat4 view;
view = glm::lookAt(glm::vec3(0.0f, 0.0f, 3.0f), 
           glm::vec3(0.0f, 0.0f, 0.0f), 
           glm::vec3(0.0f, 1.0f, 0.0f));
```

glm::LookAt函数需要一个位置、目标和上向量。它会创建一个和在上一节使用的一样的观察矩阵。

绕原点旋转的摄像机：

```c++
// In render loop.
float radius = 10.0f;
float camX = sin(glfwGetTime()) * radius;
float camZ = cos(glfwGetTime()) * radius;
glm::mat4 view;
view = glm::lookAt(glm::vec3(camX, 0.0, camZ), glm::vec3(0.0, 0.0, 0.0), glm::vec3(0.0, 1.0, 0.0)); 
```

自由移动的摄像机：

```c++

// camera
glm::vec3 cameraPos   = glm::vec3(0.0f, 0.0f,  3.0f);
glm::vec3 cameraFront = glm::vec3(0.0f, 0.0f, -1.0f);
glm::vec3 cameraUp    = glm::vec3(0.0f, 1.0f,  0.0f);

...

void processInput(GLFWwindow *window)
{
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);
// deltaTime: 每帧相隔的时间，帧率越高，这个值越小
    float cameraSpeed = 2.5 * deltaTime; 
    if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS)
        // 向前：往深度轴负半轴移动，即向屏幕内移动
        cameraPos += cameraSpeed * cameraFront;
    if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS)
        cameraPos -= cameraSpeed * cameraFront;
    if (glfwGetKey(window, GLFW_KEY_A) == GLFW_PRESS)
        cameraPos -= glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;
    if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS)
        cameraPos += glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;
}
```

