## 一. ModeList
[TOC]
Modelist的加载分为两个部分 init和update
#### Init
```java
CameraActivity.onResumeTasks() 
→ CameraAppUI.resume() 
→ CameraAppUI.showModeCoverUntilPreviewReady() 
→CameraActivity.initModelistview() 
→ ModeListView.init() && CameraAppUI.initModeListView()
```
在CameraActivity onResumeTask中先初始化modelistView, 设置其一系列listener.
#### Update
在PhotoModule创建成功,startPreview时,通过设置了一系列的callback,和线程去通知CameraActivity执行modelListView.init.
###### TODO
1.这里的callback较复杂, 出现callback嵌套的情况,callback的触发也是在framework/ex下的portability.CameraAgent实现, 尚未去阅读.

总体流程如下:
![model listview init](https://raw.githubusercontent.com/kevinkissum/picture/master/model_list_view_init.png)
###### 细节补充:
CameraAppUI通过一层层回调,最终由CameraActivity实现ModelListLoadingTask,该task中会先通过一个helper类加载好所有的items.再去init

**CameraActivity.java:1417**

```java
    Runnable mModeListLoadTask = new Runnable() {
		mModeListViewHelper = new ModeListView.ModeListViewHelper(....
        mModeListView.init(mModeListViewHelper);
			...
```

ModeListViewHelper 会先去get之前ModuleInfo加载的所有的modules的mode值存起来,并移除超出最大值的models.
```java
new ModeListView.ModeListViewHelper(CameraActivity.this,
                    mModuleManager.getSupportedModeIndexList());
                    
public ModeListViewHelper(Context context, List<Integer> modeIndexList) {
...
            for (int i = modeIndexList.size() - 1; i >= 0
                    && modeIndexList.get(i) > 	context.getResources().getInteger(
                    R.integer.camera_mode_max_index); i--) {
                modeIndexList.remove(modeIndexList.get(i));
            }
```
然后通过与camera_modes_in_nav_drawer_if_supported 这个数组中的模式匹配, 筛选出同时存在的作为最后的mSupportedModes, 

**ModelListView.java:2596**

```java
int[] modeSequence = context.getResources()
              .getIntArray(R.array.camera_modes_in_nav_drawer_if_supp	orted);

 mSupportedModes = new ArrayList<Integer>();
	for (int i = 0; i < modeSequence.length; i++) {
   		int mode = modeSequence[i];
      		if (modeIsSupported.get(mode, false)) {
       		    mSupportedModes.add(mode);
        	}
  }
```
最后通过mSupportedModes 按照不同models的属性进行赋值初始化ModeSelectorItem.
这步完成之后CameraAPPUI会重新init ModeListVIew, 为每个item实现click监听,再UpdateModeList,将items add 到ListView中.
每次的点击均会导致modelListView进行刷新.



#### Photo Video 切换

主界面有一个button, 点击会触发switch mode逻辑.

```xml
            <com.android.camera.ui.RotateImageView
                android:id="@+id/btn_mode_switch"
                android:layout_width="48dp"
                android:layout_height="48dp"
                android:layout_gravity="center"
                android:clickable="true"
                android:onClick="switchMode"  //点击监听
                android:src="@drawable/ic_switch_to_video_sprd"
                android:visibility="visible"/>
```
该button的点击监听由**switchMode**实现, 该实现在CameraActivity中

```java
    public void switchMode(View v) {
		...
        //GIF module下的冻屏操作
        if (mCurrentModule.getModuleTpye() == DreamModule.GIF_MODULE) {
            mCurrentModule.freezeScreenforGif(CameraUtil.isFreezeBlurEnable(), false);
        } else {
            //通知module进行cameraDevice.stopPreview
            mCameraRequestHandler.post(mPreSwitchRunnable);
			//filter 和 wideangle 模式下的冻屏操作
            if (mCurrentModule.getModuleTpye() == DreamModule.FILTER_MODULE
                    || mCurrentModule.getModuleTpye() == DreamModule.WIDEANGLE_MODULE) {
                mCurrentModule.freezeScreen(CameraUtil.isFreezeBlurEnable(), false);
            } else {
                //通用的冻屏处理
                freezeScreenCommon();
            }
			...
            mMainHandler.post(new Runnable() {
                @Override
                public void run() {
                    if (!mPaused) {
                        //处理module切换 Photo -> Video or Video -> Photo
                        switchModeSpecial();
                    }
                    waitToChangeMode = false;
                }
            });
        }
    }
```

如果处于SurfaceView界面, freezeScreenCommon会先截图, 然后由CameraAppUI进行设置

```java
    private void freezeScreenCommon() {
        if (mCurrentModule.isUseSurfaceView()) {
            Size displaySize = CameraUtil.getDefaultDisplaySize();
            //截图
            Bitmap screenShot = SurfaceControl.screenshot(displaySize.getWidth(), displaySize.getHeight());
            mCameraAppUI.freezeScreenUntilPreviewReady(screenShot, true);
        } else {
            mCameraAppUI.freezeScreenUntilPreviewReady();
        }
    }
```
在CameraAppUI中, 会依据displaySize 生成一个矩形区域 **RectF**, 最后将虚化后的bitmap画在该区域

```java
    public void freezeScreenUntilPreviewReady(Bitmap bitmap, boolean hasViews) {
        RectF previewArea = getPreviewArea();
		...
                Size displaySize = CameraUtil.getDefaultDisplaySize();
        		//生成矩形区域 
                previewArea = new RectF(0, 0, displaySize.getWidth(), displaySize.getHeight());
        		//虚化截图
                Bitmap blurredBitMap = CameraUtil.blurBitmap(CameraUtil.computeScale(bitmap, 0.2f),
                        mController.getAndroidContext());
                freezeScreen(blurredBitMap, previewArea);
            }
        }
    }

    @SafeVarargs
    private final <T> void freezeScreen(T... ts) {
		...
        } else if (ts.length >= 2 && ts[0] instanceof Bitmap && ts[1] instanceof Boolean) {
            bitmap = mCameraModuleScreenShotProvider.getScreenShot((Bitmap) ts[0], (Boolean) ts[1]);
        }
 		mModeTransitionView.showImageCover(bitmap);
		...
	}


```
在Switchmode中还进行了模式的切换, 会获取到默认的module Id, 然后切换到该module.

```java
    public void switchModeSpecial() {
        int curModule = mCurrentModule.getMode();
        int nextModule = DreamUtil.VIDEO_MODE;
        switch (curModule) {
            case DreamUtil.PHOTO_MODE:
                nextModule = DreamUtil.VIDEO_MODE;
                break;
            case DreamUtil.VIDEO_MODE:
                nextModule = DreamUtil.PHOTO_MODE;
                break;
        }
        DreamUtil dreamUtil = new DreamUtil();
		//前置or后置
        int cameraId = mDataModule.getInt(Keys.KEY_CAMERA_ID);
		//依据next module 获取module默认模式
        int modeIndex = dreamUtil.getRightMode(mDataModule,
                nextModule, cameraId);
        mDataModule.set(Keys.KEY_CAMERA_ID, cameraId);
        mCameraAppUI.changeToVideoReviewUI();
		//切到对应module
        onModeSelected(modeIndex);
    }
```
onModeSelected中有一个非常重要的方法**openModule**,就是在该方法中去执行moduleList数据的更新.

```java
    private void openModule(CameraModule module) {
        long start = System.currentTimeMillis();
        //初始化PhotoModule or VideoModule, 
        module.init(this, isSecureCamera(), isCaptureIntent());
		...
        // module.init 会更新moduleList的数据, showCurrentUI则会依据此数据加载ListView
        getCameraAppUI().showCurrentModuleUI(mCurrentModeIndex);
		...
    }
```
module.init会触发**DataModuleManager**更新module数据, 最终由父类**DataModuleBasic**去initData

```java
initializeData  java.lang.Throwable
at com.dream.camera.settings.DataModuleBasic.initializeData(DataModuleBasic.java:316)
at com.dream.camera.settings.DataModuleVideo.changeCurrentModule(DataModuleVideo.java:36)
at com.dream.camera.settings.DataModuleManager.changeModuleStatus(DataModuleManager.java:93)
at com.android.camera.VideoModule.init(VideoModule.java:537)
at com.android.camera.CameraActivity.openModule(CameraActivity.java:4541)
at com.android.camera.CameraActivity.onModeSelected(CameraActivity.java:4242)
at com.android.camera.CameraActivity.switchModeSpecial(CameraActivity.java:5403)
```
initData流程和之前module List iniData流程一样, 不再叙述,简要流程如下:

![](https://raw.githubusercontent.com/kevinkissum/picture/master/camera_switch_mode.png)