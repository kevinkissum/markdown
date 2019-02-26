# Sprd_滤镜实现
OpenGL ES是OpenGL的一个子集，它针对 移动端或嵌入式系统做了部分精简，而Android系统中集成了OpenGL ES，方便我们通过其接口充分使用GPU的计算和渲染能力。
GLSurfaceView是管理OpenGL surface的一个特殊的View，它可以帮助我们把OpenGL的surface渲染到Android的View上，并且封装了很多创建OpenGL环境所需要的配置，使我们能够更方便地使用OpenGL。其实使用GLSurfaceView非常简单，只要实现**GLSurfaceView.Render**接口就好了，然后通过 **GLSurfaceView.setRenderer(GLSurfaceView.Render renderer)**方法把实现的接口传到GLSurfaceView即可.

#### 自定GLSurfaceView
`public class SprdGLSurfaceView extends GLSurfaceView`
#### setRender
`this.setRenderer(m_Renderer);`

GLSurfaceView.Renderer中有非常终于的三个callback，
```java
    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        //SurfaceView创建成功会回调这个，在这里我们需要设置对camera 出帧Frames的监听。
        m_SurfaceTexture.setOnFrameAvailableListener(this);//very important callback, when preview data update, this callback will be invoked.
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {

    }

    @Override
    public void onDrawFrame(GL10 gl) {
    //实现具体效果，黑白，旋转等 eg.
    //清空颜色缓冲区和深度缓冲区
    GLES20.glClear( GLES20.GL_DEPTH_BUFFER_BIT |GLES20.GL_COLOR_BUFFER_BIT);
    GLES20.glUseProgram(programId);//指定使用刚才创建的那个程序
    
    //启用顶点数组,aPositionHandle就是我们传送数据的目标位置
    GLES20.glEnableVertexAttribArray(aPositionHandle);
    
    GLES20.glVertexAttribPointer(aPositionHandle, 3, GLES20.GL_FLOAT, false,
            12, vertexBuffer);
    GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3);
    }
    //监听预览更新情况
    @Override
    public void onFrameAvailable(SurfaceTexture surfaceTexture) {
        Log.v(TAG, "onFrameAvailable");
        this.requestRender();//来触发OpenGl的重绘
    }    
```


