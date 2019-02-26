## DreamCamera主界面分解

[TOC]

Camera 由 CameraActivity启动, CameraActivity中加载的界面是dream_main.xml, 
这个布局文件中主要包含的就是dream_camera.xml布局文件.  DreamCamera界面上的布局几乎都是以该文件作为入口.
由dream_camera.xml可知, DreamCamera App在界面显示上由上到下主要分为几下几个部分:

* top_panel
* side_panel
* extend_panel
* bottom_panel
* slide_panel

左右两侧分别是ModeList 和 Settings页面.
CameraActivity在启动的时候会去init所有的module,

```java
public static void setupDreamModules(Context context, ModuleManager 		moduleManager,OneCameraFeatureConfig config) {
    Resources res = context.getResources();
    int photoModuleId = context.getResources().getInteger(R.integer.camera_mode_auto_photo);
    registerAutoPhotoModule(moduleManager, photoModuleId, SettingsScopeNamespaces.AUTO_PHOTO);
    moduleManager.setDefaultModuleIndex(photoModuleId);
    registerManualPhotoModule(moduleManager, res.getInteger(R.integer.camera_mode_manual),
            SettingsScopeNamespaces.MANUAL);...
```

然后根据不同的module 再去create 不同的UI, 不同的ModuleUI有不同的继承关系,
最主要的就是继承自**DreamPhotoUI**和**DreamVideoUI**

Photo相关涉及到主要的类有:
* PhotoUI
* DreamPhotoUI
* CameraAppUI

**CameraAppUI **是加载 R.id.activity_root_view, 该view可以说是 root view, 几乎包含所有布局.

**DreamPhotoUI** 是所有Photo类型module的父类, 各panel的加载即由它开始执行.

```java
	public void onPreviewStarted() {
        mActivity.getCameraAppUI().initBottomBar();
        mActivity.getCameraAppUI().initExtendPanel();
        updateSidePanel();
        mActivity.getCameraAppUI().initSidePanel();
        updateBottomPanel();
        updateSlidePanel();
        updateTopPanel();
         ...
```
其也实现了所有top_panel上所有button的加载, 
```java
	    public void bindSettingsButton(View settingsButton) {
    	public void bindFlashButton() {
    	public void bindCountDownButton() {
    	public void bindHdrButton() {
    	public void bindCameraButton() {
```
**PhotoUI** 为DreamPhotoUI的父类, 该类实现了SurfaceHolder.Callback, 当Surface创建成功,其才会通知module开始加载 UI.



## 二.Panel加载
### panel初始化
所有panel均由DreamPhotoUI完成加载, 过程大同小异,这仅以top_panel为例,流程如下:
![top panel](https://raw.githubusercontent.com/kevinkissum/picture/master/top_panel_init.png)
###### 细节补充:
CameraActivity 会加载默认的module,默认init AutoPhotoModule, 而init方法在父类PhotoModule中实现,之后有PhotoModule创建其子类的UI.
```java
setModuleFromModeIndex(getModeIndexDefault());

PhotoModule.java:671
     public void init(CameraActivity activity, boolean isSecureCamera,
        mUI = createUI(mActivity);
        
PhotoModule.java:37
    public PhotoUI createUI(CameraActivity activity) {
        return new AutoPhotoUI(activity, this, activity.getModuleLayoutRoot());
```
## panel显示分析
暂只分析了top_panel中不同button的初始化即控制流程, 
Button 由ButtonManagerDream进行加载, 用户点击的行为会通过MultiToggleImageButton触发, 通过设置的callback,分别传递给PhotoModule 和ButtonManagerDream进行处理,流程如下:
![top panel bt](https://raw.githubusercontent.com/kevinkissum/picture/master/bt_init.png)

###### TODO:
其它panel加载之后的处理流程.








