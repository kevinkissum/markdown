# Camera FW架构简析

[TOC]

 工欲善其事必先利其器, Camera FW大量用到Binder和AIDL, 学习其架构, 需要了解如何使用.

- [Binder](http://gityuan.com/2015/11/22/binder-use/)

- [AIDL](https://www.jianshu.com/p/d1fac6ccee98)

  

由之前的分析可知, camera通过代理类**CameraAgent**, 对camera进行open的操作, 该类被两个子类继承.

- AndroidCamera2AgentImpl
- AndroidCameraAgentImpl --> SprdAndroidCameraAgentImpl

可以看出, 这是因为camera2和camera1的区别, 会依据条件选择性的实现子类.

先上一个乞丐版的框架图

![](https://raw.githubusercontent.com/kevinkissum/picture/master/camera_fw.png)



Java层是请求事件的发起均是从Camera的代理类**CameraAgent**开始.

当camera未open时, 是由**CameraManager**通过IPC完成open, 并获得cameraDevice, 而后续camera的close, preview是由cameraDevice类完成, 均是通过与ICameraService通信完成,但尴尬的是一直未能找到ICameraService具体实现在哪, 所以server端的去向就不明确了, 暂Block     :( 

当cameraOpen之后, 进行的Capture等操作也是由**CameraAgent.Proxy**完成.



### TODO

Find the implement of ICameraService!

