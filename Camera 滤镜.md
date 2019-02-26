# Camera 滤镜

[TOC]



### GLSurfaceView使用

**GlSurfaceView**继承自**SurfaceView**类，专门用来显示OpenGL渲染的，简单理解可以显示视频，图像及3D场景这些的。

#### 实现思路

* 以一个**SurfaceTexture**作为接收相机预览数据的载体，这个**SurfaceTexture**就是处理器的输入。

* **SurfaceView**、**TextureView**或者是**Surface**，提供**SurfaceTexture**或者**Surface**给处理器作为输出，来接收处理结果。
* 重点就是处理器了。处理器利用**GLSurfaceView**提供的GL环境，以相机数据作为输入，进行处理，处理的结果渲染到视图提供的输出的**Surface**上。

在此之前还需要初始化**GLSurfaceView**， 如果说**GLSurfaceView**是一个画布，那么光有一块白纸是没用的，得要在白纸上画图，通过什么画图呢？就是接下来要说的**Renderer**。**Renderer**是一个接口，它主要包含3个抽象函数：**onSurfaceCreated**、**onDrawFrame**、**onSurfaceChanged**; 从名字就可以看出，分别是在**SurfaceView**创建时调用、在绘制图形时调用以及在视图大小发生改变时调用。而GL就是我们的画笔，GL提供了丰富的API供我们绘制2D，3D的效果。openGL由C实现，Java层调用大致要经过以下几个步骤：

##### 1. 编写及初始化OpenGL着色器程序

*着色器程序语法与C语言很像，顶点着色器和片段着色器都包含一个main函数，main函数外定义了三种不同类型的变量：uniform、attribute和varying。uniform变量是外部程序传递给着色器的变量，类似C语言的const变量，在OpenGL着色器程序的一次渲染过程中保持不变；attribute变量只在顶点着色器中使用，一般用来表示一些顶点的数据，如顶点坐标，法线，纹理坐标，顶点颜色等；varying变量是顶点着色器和片段着色器之前传递数据用的，它作为顶点着色器的输出，经过图元装配和栅格化后，作为片段着色器的输入。着色器中也内置了一些变量和函数，本文中介绍两个最最常用的内置变量：*

1. *gl_Position：顶点着色器中必须对其赋值，其输入序列作为图元装配过程的组成点、线或三角形的坐标序列。*
2. *gl_FragColor：片段着色器中必须对其赋值，作为像素点的输出值。*

```java
//顶点着色器
private final String vertexShaderCode = 
    "uniform mat4 textureTransform;\n" +
        "attribute vec2 inputTextureCoordinate;\n" +
        "attribute vec4 position;            \n" +//NDK坐标点
        "varying   vec2 textureCoordinate; \n" +//纹理坐标点变换后输出
        "\n" +
        " void main() {\n" +
        "     gl_Position = position;\n" +
        "     textureCoordinate = inputTextureCoordinate;\n" +
        " }";
```

```java
//片段着色器
    private final String fragmentShaderCode =
            //使用外部纹理必须支持此扩展
            "#extension GL_OES_EGL_image_external : require\n" +
            "precision mediump float;\n" +
                    //外部纹理采样器
            "uniform samplerExternalOES videoTex;\n" +
            "varying vec2 textureCoordinate;\n" +
            "\n" +
            "void main() {\n" +
            "    vec4 tc = texture2D(videoTex, textureCoordinate);\n" +
            "    float color = tc.r * 0.3 + tc.g * 0.59 + tc.b * 0.11;\n" +  //所有视图修改成黑白
            "    gl_FragColor = vec4(color,color,color,1.0);\n" +
            "}\n";
```

##### 2. 编译&链接

OpenGL的程序像极了C程序， 需要编译，链接才能使用，我们有了顶点着色器程序和片段着色器程序将其编译链接后就可以使用了。
GL已经提供好现成的API：  

```java
        //通常做法
        int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, vertexShaderCode);
        int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, fragmentShaderCode);
        // 将两个Shader链接至program中
        // 创建空的OpenGL ES程序
        mProgram = GLES20.glCreateProgram();
        // 添加顶点着色器到程序中
        GLES20.glAttachShader(mProgram, vertexShader);
        // 添加片段着色器到程序中
        GLES20.glAttachShader(mProgram, fragmentShader);
        // 创建OpenGL ES程序可执行文件
        GLES20.glLinkProgram(mProgram);
        // 释放shader资源
        GLES20.glDeleteShader(vertexShader);
        GLES20.glDeleteShader(fragmentShader);
```

##### 3. 激活

经过以上步骤，我们处理相机流数据的顶点着色器和片段着色器程序就准备好了，最后得到的program就是一个OpenGL ES程序对象，我们可以调用glUseProgram函数，用刚创建的程序对象作为它的参数，以激活这个程序对象。

`
GLES20.glUseProgram(mProgram);
`

##### 4. 为着色器程序传递参数

前面提到， 我们编写着色器程序，定义了好几个变量，**attribute**，**varying**，需要先拿到其对应的句柄才能进行传参操作。这两种类型参数获取句柄的方法略有不同，以获取上文中**attribute**类型参数**position**和**uniform**类型参数**textureTransform**为例，获取句柄方法分别如下：

`uPosHandle = GLES20.glGetAttribLocation(mProgram, "position");`
`aTexHandle = GLES20.glGetAttribLocation(mProgram, "inputTextureCoordinate");`
`mMVPMatrixHandle = GLES20.glGetUniformLocation(mProgram, "textureTransform");`

获取到句柄后，接下来就是把真正的参数值传进句柄了。

```java
// 允许顶点着色器读取uPosHandle  aTexHandle 对应GPU的数据
GLES20.glEnableVertexAttribArray(uPosHandle);
GLES20.glEnableVertexAttribArray(aTexHandle);
// 将mPosBuffer中的数据传入到句柄uPosHandle中
GLES20.glVertexAttribPointer(uPosHandle, 2, GLES20.GL_FLOAT, false, 0, mPosBuffer);
GLES20.glVertexAttribPointer(aTexHandle, 2, GLES20.GL_FLOAT, false, 0, mTexBuffer);
```

此处涉及到两个OpenGL ES相关的函数调用：

> glEnableVertexAttribArray调用后允许顶点着色器读取句柄对应的GPU数据。默认情况下，出于性能考虑，所有顶点着色器的attribute变量都是关闭的，意味着数据在着色器端是不可见的，哪怕数据已经上传到GPU.由glEnableVertexAttribArray启用指定属性，才可在顶点着色器中访问逐顶点的attribute数据。glVertexAttribPointer或VBO只是建立CPU和GPU之间的逻辑连接，从而实现了CPU数据上传至GPU。但是，数据在GPU端是否可见，即着色器能否读取到数据，由是否启用了对应的属性决定，这就是glEnableVertexAttribArray的功能，允许顶点着色器读取GPU数据。
>
> glVertexAttribPointer函数的参数非常多：第一个参数指定句柄；第二个参数指定顶点属性的大小，每个坐标点包含x和y两个float值；第三个参数指定数据的类型，这里是GL_FLOAT(GLSL中vec*都是由浮点数值组成的)；第四个参数定义我们是否希望数据被标准化(Normalize)。如果我们设置为GL_TRUE，所有数据都会被映射到0（对于有符号型signed数据是-1）到1之间，这里我们把它设置为GL_FALSE；第五个参数叫做步长(Stride)，它告诉我们在连续的顶点属性组之间的间隔，由于下个组位置数据在2个GLfloat之后，我们把步长设置为2* sizeof(GLfloat)；最后一个参数就是数据buffer。

##### 5. 渲染帧数据

前面步骤都完成后，调用OpenGL ES的渲染指令倒是比较简单了.

```java
mSurfaceTexture.updateTexImage();
//将纹理矩阵传给片段着色器
GLES20.glUniformMatrix4fv(mMVPMatrixHandle, 1, false, mMVPMatrix, 0);
//绘制两个三角形（mPosCoordinate.length / 2 个顶点）
GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, mPosCoordinate.length / 2);
```

>SurfaceTexture的updateTexImage方法会更新接收到的预览数据到其绑定的OpenGL纹理中。该纹理会默认绑定到OpenGL Context的GL_TEXTURE_EXTERNAL_OES纹理目标对象中。GL_TEXTURE_EXTERNAL_OES是OpenGL中一个特殊的纹理目标对象，与GL_TEXTURE_2D是同级的，有兴趣的同学可以网上搜教程深入了解一下。调用此方法后，我们前面创建的OpenGL纹理中就有了最新的相机预览数据了。要注意的是，此方法只能在生成该纹理的OpenGL线程中调用，所以这个地方通过GLSurfaceView的queueEvent方法将该调用放入GL线程队列中执行。

>SurfaceTexture的getTransformMatrix方法可以获取到图像数据流的坐标变换矩阵。一般情况下，相机流数据方向并不是用户正常拿手机的竖屏方向，且前后摄像头数据还存在镜像的问题。如何对摄像头数据进行旋转或镜像得到旋转正确的数据呢？getTransformMatrix获取到的变换矩阵可以帮助我们完成这个看起来很复杂的任务。其实我们不用关心这个矩阵的值到底是什么，只需要在OpenGL 着色器处理顶点数据时直接将其传入作为纹理坐标变换矩阵即可。



#### GLSurfaceView 与 Camera的设置

打开摄像头以后，我们需要为相机设置一个预览的SurfaceTexture接收来自相机的图像数据流。

**SurfaceTexture**和**OpenGL ES**一起使用可以创造出无限可能，下面我们先来看看如何创建一个OpenGL纹理并把它绑定到一个**SurfaceTexture**，然后将该**SurfaceTexture**设置为相机预览数据接收器：

```java
private int createOESTextureObject() {
    int[] tex = new int[1];
    //生成一个纹理
    GLES20.glGenTextures(1, tex, 0);
    //将此纹理绑定到外部纹理上
    GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, tex[0]);
    //设置纹理过滤参数
    GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
            GL10.GL_TEXTURE_MIN_FILTER, GL10.GL_NEAREST);
    GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
            GL10.GL_TEXTURE_MAG_FILTER, GL10.GL_LINEAR);
    GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
            GL10.GL_TEXTURE_WRAP_S, GL10.GL_CLAMP_TO_EDGE);
    GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
            GL10.GL_TEXTURE_WRAP_T, GL10.GL_CLAMP_TO_EDGE);
    GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, 0);
    return tex[0];
}
mSurfaceTexture = new SurfaceTexture(createOESTextureObject());
//将该SurfaceTexture设置为相机预览数据接收器
mCamera.setPreviewTexture(mSurfaceTexture);
```



##### 设置SurfaceTexture回调，通知摄像头预览数据已更新

SurfaceTexture有一个很重要的回调：OnFrameAvailableListener。通过名字也可以看出该回调的调用时机，当相机有新的预览帧数据时，此回调会被调用。所以我们为前面的SurfaceTexture设置一个回调，来通知我们相机预览数据已更新：

```java
mSurfaceTexture.setOnFrameAvailableListener(new SurfaceTexture.OnFrameAvailableListener() {
    @Override
    public void onFrameAvailable(SurfaceTexture surfaceTexture) {
        requestRender();
    }
});
```





