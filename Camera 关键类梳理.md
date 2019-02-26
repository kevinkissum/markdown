# Camera 关键类梳理

[TOC]



## View

### CameraAppUI
CameraAppUI是包含root_view的自定义view, 它主要负责对界面的操控以及对各种状态的监听

```java
  mCameraAppUI = new CameraAppUI(this,
          (MainActivityLayout) findViewById(R.id.activity_root_view), isCaptureIntent());
```

#### 界面操控

```java
//主要包括界面的Init、Update、show、hide、 transition等.
initBottomBar()
...
hideBottomBar()
...
updateModeList()
...
showBottomBar()
...
transitionToCapture()
...
//其它还有冻屏, get参数等
freezeScreen()
getBottomHeight()
getModuleIndex()  
```

#### 自定义Interface

```java
    public static interface BottomPanel {
        public final int VIEWER_NONE = 0;
        public final int VIEWER_PHOTO_SPHERE = 1;
        public final int VIEWER_REFOCUS = 2;
        public final int VIEWER_OTHER = 3;

        void setListener(Listener listener);
        void setClingForViewer(int viewerType, Cling cling);
        void clearClingForViewer(int viewerType);
        Cling getClingForViewer(int viewerType);
        void setVisible(boolean visible);
        void setEditButtonVisibility(boolean visible);
        void setEditEnabled(boolean enabled);
        void setViewerButtonVisibility(int state);
        void setViewEnabled(boolean enabled);
        void setPuzzleButtonVisibility(boolean visible);
        void setPuzzleEnabled(boolean enabled);
        void setTinyPlanetEnabled(boolean enabled);
        void setDeleteButtonVisibility(boolean visible);
        void setDeleteEnabled(boolean enabled);
        void setShareButtonVisibility(boolean visible);
        void setShareEnabled(boolean enabled);
        void setProgressText(CharSequence text);
        void setProgress(int progress);
        void showProgressError(CharSequence message);
        void hideProgressError();
        void showProgress();
        void hideProgress();
        void showControls();
        void hideControls();
        void updateControlPadding();

        public static interface Listener {
            public void onExternalViewer();
            public void onEdit();
            public void onTinyPlanet();
            public void onDelete();
            public void onShare();
            public void onProgressErrorClicked();
            public void onPuzzle();
        }
    }
```

唯一实现 `class FilmstripBottomPanel implements CameraAppUI.BottomPanel` 用于控制幻灯片View的行为

```java
    public interface CameraModuleScreenShotProvider {
        public Bitmap getPreviewFrame(int downSampleFactor);
        public Bitmap getPreviewFrameWithoutTransform(int downSampleFactor);
        public Bitmap getPreviewOverlayAndControls();
        public Bitmap getScreenShot(int previewDownSampleFactor);
        public Bitmap getScreenShot(Bitmap bmp);
        public Bitmap getScreenShot(Bitmap bmp, RectF previewArea);
        public Bitmap getScreenShot(Bitmap bmp, boolean hasViews);
        public Bitmap getBlackPreviewFrame(int downSampleFactor);
        public Bitmap getBlackPreviewFrameWithButtons();
    }
```

CameraAppUI内部实现该接口, 唯一注册是在moduleListView中,  提供给ModelListView操作屏幕的一些方法,如截图等

```java
    public interface PanelsVisibilityListener {
        void onPanelsHidden();
        void onPanelsShown();
    }
```

只在UcamFilterPhotoUI和TDPhotoModule这个俩个module中有实现, 控制panel show or hide.



#### 内部类

**public static class BottomBarUISpec**

给module提供callback以及参数

**private class MyGestureListener extends GestureDetector.SimpleOnGestureListener**

手势监听类

**private class MyTouchListener implements View.OnTouchListener**

touch监听



#### CameraAppUI实现的接口

```java
ModeListView.ModeSwitchListener
	    public void onModeButtonPressed(int modeIndex);
        public void onModeSelected(int modeIndex);
        public int getCurrentModeIndex();
        public void onSettingsSelected();
```

当点击切换module时, 主界面会有的一些操作, 尤其是**onModeSelected**.

```java
TextureView.SurfaceTextureListener
        public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height);
        public void onSurfaceTextureSizeChanged(SurfaceTexture surface, int width, int height);
        public boolean onSurfaceTextureDestroyed(SurfaceTexture surface);
        public void onSurfaceTextureUpdated(SurfaceTexture surface);
```

主界面包含SurfaceTextureView, 此处对其进行状态监听



### DreamPhotoUI & DreamVideoUI

该类是一个抽象类, 被不同module所对应的UI继承,

- public class AudioPictureUI extends DreamPhotoUI

- public class AutoPhotoUI extends DreamPhotoUI

- public class ContinuePhotoUI extends DreamPhotoUI 

  ...

内部实现了主界面button的bind, 子类便可按需选择

- public void bindFlashButton() {

- public void bindCountDownButton() {

- public void bindHdrButton() {

  ...

#### 实现的接口

```java
implements DreamInterface
public void fitTopPanel(ViewGroup topPanelParent);
public void updateSidePanel();
public void fitExtendPanel(ViewGroup extendPanelParent);
public void updateBottomPanel();
public void updateSlidePanel();
public int getSidePanelMask();
```
控制子类的panel





### PhotoUI & VideoUI

PhotoUI和VideoUI 大同小异, VIdeoUI相对简单一些, 其也只实现了**implements PreviewStatusListener, SurfaceHolder.Callback**这个俩接口

该类是所有UI类父亲的父亲, 有很多方法均是空实现, 会由其子类进行改写， 其次是将逻辑转移到Control中实现

```java
//Init、set、 update、show等操作
public void initUI() {
public void setButtonVisibility(int buttonId, int visibility) {
public void showBurstScreenHint(int count) {
protected void updateExposureUI() {};
    
//在Control中实现
private class ZoomChangeListener implements PreviewOverlay.OnZoomChangedListener {
    @Override
    public void onZoomValueChanged(float ratio) {
        mController.onZoomChanged(ratio);
    }
@Override
public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
       mController.onPreviewUIDestroyed();
       return true;
}
```

#### 实现的接口

```java
implements PreviewStatusListener //该接口继承 TextureView.SurfaceTextureListener 该接口添加了手势和touch等监听
    public GestureDetector.OnGestureListener getGestureListener();
    public View.OnTouchListener getTouchListener();
    public void onPreviewLayoutChanged(View v, int left, int top, int right,
            int bottom, int oldLeft, int oldTop, int oldRight, int oldBottom);
    public interface PreviewAreaChangedListener {
        public void onPreviewAreaChanged(RectF previewArea);
    }
    public boolean shouldAutoAdjustTransformMatrixOnLayout();
    public void onPreviewFlipped();
    public interface PreviewAspectRatioChangedListener {
        public void onPreviewAspectRatioChanged(float aspectRatio);
    }
```

监听SurfaceTexture状态以及gesture和touch等.

```java
 CameraAgent.CameraFaceDetectionCallback
 public void onFaceDetection(Camera.Face[] faces, CameraProxy camera);

 FaceDetectionController
    public void pauseFaceDetection();
    public void resumeFaceDetection();
```

*framework/ex/camera2*下的接口, 用于face detection





## Module

### PhotoModule

该类是所有Module类父亲的父亲, module架构和UI一样其下也有一个abstract的子类, DreamPhotoModule, 不过该类太简单,没啥好讲的.

PhotoModule不仅包含了很多init操作以及N个update、set,  它负责了很多camera主要的逻辑操作， 绝对核心。

```java
//拍照相关
protected void focusAndCapture() {
public void autoFocus() {
public boolean capture() {
    
//Preview
 public void init(CameraActivity activity, boolean isSecureCamera,
 public void onPreviewUIReady() {
 protected void startPreview(boolean optimize) {
 protected void doStartPreview(CameraAgent.CameraStartPreviewCallback startPreviewCallback, CameraAgent.CameraProxy cameraDevice) {
 
 //etc...
```
#### 实现的接口

```java
//该接口主要是一些拍照控制的逻辑以及Preview状态控制
public interface PhotoController extends OnShutterButtonListener,MakeupListener {
    public static final int PREVIEW_STOPPED = 0;
    public static final int IDLE = 1;  // preview is active
    public static final int FOCUSING = 2;
    public static final int SNAPSHOT_IN_PROGRESS = 3;
    public static final int SWITCHING_CAMERA = 4;
    public void onZoomChanged(float requestedZoom);
    public boolean isImageCaptureIntent();
    public boolean isCameraIdle();
    public void onCaptureDone();
    public void onCaptureCancelled();
    public void onCaptureRetake();
    public void cancelAutoFocus();
    public void stopPreview();
    public int getCameraState();
    public void onSingleTapUp(View view, int x, int y);
    public void updatePreviewAspectRatio(float aspectRatio);
    public void updateCameraOrientation();
    public void onPreviewUIReady();
    public void onPreviewUIDestroyed();
    public void startPreCaptureAnimation();
    public boolean isShutterEnabled();//SPRD:Add for ai detect
    public void setCaptureCount(int count);// SPRD:Add for ai detect 

    //拍照按钮或录制按钮的监听
    public interface OnShutterButtonListener {
        void onShutterButtonFocus(boolean pressed);
        void onShutterCoordinate(TouchCoordinate coord);
        void onShutterButtonClick();
        void onShutterButtonLongPressed();
    }
    //makeup模式下对view状态的监听
    public interface MakeupListener {
        public void onBeautyValueChanged(int[] value);
        public void onBeautyValueReset();
        public void setMakeUpController(MakeupController makeUpController);
        public void updateMakeLevel();
    }
```



该接口为MVC下的C, 目的是当View状态变化时, 通过Control操作module更新, 所以该接口的持有者均是View.

```java
//父类View
public PhotoUI(CameraActivity activity, PhotoController controller, View parent) {
    ...
}
//子类View
public ManualPhotoUI(CameraActivity activity, PhotoController controller,
        View parent) {
    super(activity, controller, parent);
}
```


```java
//该接口主要是控制mode状态
public interface ModuleController extends ShutterButton.OnShutterButtonListener {
    public static final int VISIBILITY_VISIBLE = 0;
    public static final int VISIBILITY_COVERED = 1;
    public static final int VISIBILITY_HIDDEN = 2;
    public void init(CameraActivity activity, boolean isSecureCamera, boolean isCaptureIntent);
    public void resume();
    public void pause();
    public void destroy();
    public void onPreviewVisibilityChanged(int visibility);
    public void onLayoutOrientationChanged(boolean isLandscape);
    public abstract boolean onBackPressed();
    public void onCameraAvailable(CameraAgent.CameraProxy cameraProxy);
    public HardwareSpec getHardwareSpec();
    public BottomBarUISpec getBottomBarSpec();
    public boolean isUsingBottomBar();
```

该接口唯一的持有者是CameraActivity. 其create module, 所以持有moduleControl.

`mCurrentModule = (CameraModule) agent.createModule(this, getIntent());`

CameraAppUI通过activity的get方法,也能获得moduleControl, 从而完成注册或者回调.

```java
CameraActivity.java
public CameraModule getCurrentModule() {
    return mCurrentModule;
}
CameraAppUI.java
ModuleController moduleController = mController.getCurrentModuleController();
applyModuleSpecs(moduleController.getHardwareSpec(),moduleController.getBottomBarSpec());
```


```java
//内部类 MemoryManager的接口
//查询Memory情况
public static interface MemoryListener {
      public void onMemoryStateChanged(int state);
      public void onLowMemory();
}    
```

###### 补充说明

**MemoryManager**的接口有一个实现类 `MemoryManagerImpl.java`, 它也override了该接口

```java
    @Override
    public HashMap queryMemory() {
        return mMemoryQuery.queryMemory();
    }
```

而**mMemoryQuery**是 **MemoryManagerImpl**进行 new MemoryQuery(activityManager)得到的, 所以queryMemory具体操作在**MemoryQuery**中完成, MemoryManagerImpl在**CameraServicesImpl**中完成创建, camera中所有**ManagerImp**类型均在此创建, **CameraServicesImpl**是单例模式, 其内部提供了各种Imp的get方法.

```java
//FocusOverlayManager的内部接口, 负责focus相关
FocusOverlayManager.Listener
        public void autoFocus();
        public void cancelAutoFocus();
        public boolean capture();
        public void startFaceDetection();
        public void stopFaceDetection();
        public void setFocusParameters();
}
```

该接口通过focusManager管理, focusManager在PhotoModel中实例化,

 `mFocusManager = new FocusOverlayManager(mAppController,...`



### VideoModule

整体结构和PhotoModule一样, 只是对应Photo相关的逻辑变成Record, Video模式的核心类

```java
private void startVideoRecording() 
private void stopVideoRecording()
```

##### 实现的接口

这里只列举和PhotoModule相异部分

```java
 //MediaRecord相关的监听类,由frameworks/base/media/触发
 MediaRecorder.OnErrorListener
 MediaRecorder.OnInfoListener
 //video相关逻辑的回调, 实现机制同PhotoModule
 VideoController
```





## Controller

### CameraController

顾名思义, 控制camera的类以及对camera状态的监听 列如:

```java
private static void checkAndOpenCamera(CameraAgent cameraManager,
public void closeCamera(boolean synced)
public CameraId getCurrentCameraId() 
public int getNumberOfCameras()
public void releaseCamera(int id)
public void requestCamera(int id)
...
public void onCameraDisabled(int cameraId)
public void onCameraOpened(CameraAgent.CameraProxy camera)
```

#### 实现的接口

```java
//CameraControl 和 CameraActivity均实现了该接口, 两者均会对camera状态进行监听, 前者操作camera, 后者操作module&UI.
implements CameraAgent.CameraOpenCallback
    public static interface CameraOpenCallback {
        public void onCameraOpened(CameraProxy camera);
        public void onCameraDisabled(int cameraId);
        public void onDeviceOpenFailure(int cameraId, String info);
        public void onDeviceOpenedAlready(int cameraId, String info);
        public void onReconnectionFailure(CameraAgent mgr, String info);
    }
```

CameraAgent内部类接口，该接口在CameraManager请求openCamera成功之后， 会触发callback --> *onCameraOpened(CameraProxy camera)* , Control便拥有camera实例.

```java
public interface CameraProvider {
    public void requestCamera(int id);
    public void requestCamera(int id, boolean useNewApi);
    public boolean waitingForCamera();
    public void releaseCamera(int id);
    public void setCameraExceptionHandler(CameraExceptionHandler exceptionHandler);
    public Characteristics getCharacteristics(int cameraId);
    public CameraId getCurrentCameraId();
    public int getNumberOfCameras();
    public int getFirstBackCameraId();
    public int getFirstFrontCameraId();
    public boolean isFrontFacingCamera(int id);
    public boolean isBackFacingCamera(int id);
    public boolean isNewApi();
}
```

CameraController是唯一实现CameraProvider接口的类, 该接口主要是将camera的部分操作封装成一个接口类, *暂未理解这样写有何深意*.



## Bean类

### DataModuleBasic

该类为抽象类, 只有一个子类DataModuleCamera, 该类依据不同module, init不同的数组, 并提供get、set、update这些数据的操作.

```java
//该方法会依据module类型, init出不同数组的hashMap, 以供之后的操作
public void initializeData(DataStructSetting dataSetting) 
    ...
```

#### 内部类

```java
//该类init SharedPreferences, 并提供get和set方法
public class DataSPAndPath
```



# Camera Callback梳理

#### CameraActivity中:

**mAvailabilityCallback**

```java
private CameraManager.AvailabilityCallback mAvailabilityCallback = new CameraManager.AvailabilityCallback() {

    @Override
    public void onCameraAvailable(String cameraId) {
        if (mCameraAvailableMap != null)
            mCameraAvailableMap.put(cameraId, true);
    	}
            //camera is available, resume mode now
            mMainHandler.post(mResumeRunnable);
    }

    public void onCameraUnavailable(String cameraId) {
        if (mCameraAvailableMap != null)
            mCameraAvailableMap.put(cameraId, false);

        checkAllCameraAvailable();
    }
};
```
该callback监听cameraAvailable状态, 进行module的resume操作, callback由CameraManager进行注册, CameraManager在framework/base下hardware/camera2, 

###### TODO

callback最终由jni方法触发, 需要梳理C++部分流程



**new CameraAgent.CameraPreviewDataCallback()**

```java
@Override
public void setupOneShotPreviewListener() {
    mCameraController.setOneShotPreviewCallback(mMainHandler,
            new CameraAgent.CameraPreviewDataCallback() {
                @Override
                public void onPreviewFrame(byte[] data, CameraAgent.CameraProxy camera) {
                    mCurrentModule.onPreviewInitialDataReceived();
                    mCameraAppUI.onNewPreviewFrame();
                }
            }
            );
}
```
###### TODO

具体细节在`frameworks/ex/camera2/portability/src/com/android/ex/camera2/portability/AndroidCameraAgentImpl.java` , 需要梳理



###### TODO

**mCameraExceptionCallback**



#### PhotoModule & VideoModule中:

**mAutoFocusCallback**

```java
private final class AutoFocusCallback implements CameraAFCallback {
    @Override
    public void onAutoFocus(boolean focused, CameraProxy camera) {
        SessionStatsCollector.instance().autofocusResult(focused);
        mFocusManager.onAutoFocus(focused, false);
    }
}
```
callback在CameraDevices实现类*AndroidCameraAgentImpl*中注册, 

###### TODO

auto focus 由native方法实现, 具体具体流程还需要梳理.



###### TODO

**AutoFocusMoveCallback**

```java
private final class AutoFocusMoveCallback implements CameraAFMoveCallback {
    @Override
    public void onAutoFocusMoving(boolean moving, CameraProxy camera) {
        if (isAudioRecording()) return;
        mFocusManager.onAutoFocusMoving(moving);
        SessionStatsCollector.instance().autofocusMoving(moving);
    }
}
```











