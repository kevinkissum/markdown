[TOC]



## 一.各Module的关系

Camera中每种mode都对应一种Module, 所有的modules都是在ModuleInfos.java中进行注册
和创建. 不同的Modules有不同的继承关系,最主要的就是继承自DreamPhotoModule和DreamVideoModule.其父类分别为PhotoModule和VideoModule, 这两个类既负责Model的处理,同时也承担了Control的角色.

## 二.Module注册及UI创建

Camera一启动就通过CameraActivity 另起线程**mOncreateModulesInfoAndCurrentModule** 去init Module, PhotoModule 和VideoModule流程大同小异, 此处以PhotoModule为例, 流程如下:
![model init](https://raw.githubusercontent.com/kevinkissum/picture/master/module_init.png)


###### 细节补充:
ModulesInfo通过向ModuleManager注册所有的Module

**ModulesInfo.java:393**

```java
    public static void setupDreamModules(Context context, ModuleManager ,
		registerAutoPhotoModule...
		registerManualPhotoModule...
		registerIntervalPhotoModule...
		registerIntentCaptureModule...
		registerIntentVideoModule....
		....
```
并初始化ModuleManager的callBack: ModuleAgent

**ModulesInfo.java:545**

```java
   private static void registerAutoPhotoModule(
   		ModuleManager moduleManager, 
      	moduleManager.registerModule(new ModuleManager.ModuleAgent()...
```
注册多少ModuleManager就存多少.

**ModuleManagerImpl.java:44**

```java
    public void registerModule(ModuleAgent agent) {
        mRegisteredModuleAgents.put(moduleId, agent);
```
之后CameraActivity通过查询module的id获取到callback:ModuleAgent, 并完成module的创建, 

**CameraActivity.java:4328**

```java
agent = mModuleManager.getModuleAgent(modeIndex);
mCurrentModule = (CameraModule) agent.createModule(this, getIntent());
```
注意这里AutoPhotoModule被强转为其父类的父类,所以接下来的module init会执行父类的init.

**CameraActivity.java:2751**

` mCurrentModule.init(this, isSecureCamera(), isCaptureIntent()); `

###### TODO:
特殊module 的流程分析尚未梳理, 需要完善.

## 三.Module对应关系
 继承自DreamPhotoModule
| Module | UI | Controller |
| -------- | ----- | :----: |
| IntervalPhotoModule | IntervalPhotoUI | DreamController |
| VgesturePhotoModule | VgesturePhotoUI | |
| FrontBlurRefocusModule | FrontBlurRefocusUI | |
| BlurRefocusModule | BlurRefocusUI | |
| TDPhotoModule | TDPhotoUI | |
| TDNRPhotoModule | TDNRPhotoUI | |
| AutoPhotoModule | AutoPhotoUI | |
| DreamIntentCaptureModule | DreamIntentCaptureUI | |
| ScenePhotoModule | ScenePhotoUI | |
| AudioPictureModule | AudioPictureUI | |
| ContinuePhotoModule | ContinuePhotoUI | |
| ManualPhotoModule | ManualPhotoModule | |
继承自DreamVideoModule
| Module | UI | Controller |
| -------- | ----- | :----: |
| DreamIntentVideoModule | DreamIntentVideoUI | VideoController |
| TimelapseVideoModule | TimelapseVideoUI | |
| AutoVideoModule | AutoVideoUI | |
| SlowmotionVideoModule | SlowmotionVideoUI | |
| TDNRVideoModule | TDNRVideoUI | |
| TDVideoModule | TDVideoUI | |
其他的继承关系
| Parent | Module | UI| Controller |
| -------- | ----- | ----- | :----: |
| ReuseModule | QrCodePhotoModule | QrCodePhotoUI |ModuleController|
| RefocusModule | DreamRefocusModule | DreamRefocusUI|RefocusController|
| WideAnglePanoramaModule | DreamPanoramaModule |DreamPanoramaUI |WideAnglePanoramaController|
| (Fake)UcamFilterPhotoModule | DreamFilterModule | DreamFilterUI|ModuleController|
| SprdSceneryModule | DreamSceneryModule |DreamSceneryUI |ModuleController|
| FilterModuleAbs | FilterModuleSprd |FilterModuleUISprd |DreamFilterLogicControlInterface|





