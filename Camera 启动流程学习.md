# Camera 启动流程学习

[TOC]



### Camera打开及预览流程

DreamCamera仍然是基于android原生的camera, 所以这里先普及一下 android 原生camera的打开及预览流程.
```java
   // 1.调用Camera的open()方法打开相机。
   camera = Camera.open();  
   // 2.进行camera参数设置
   Camera.Parameters parameters = camera.getParameters();   
   parameters.setPreviewSize(screenWidth, screenHeight);
   // 每秒显示4帧
   parameters.setPreviewFrameRate(4);
   ... 
   camera.setParameters(parameters);   

   // 3.通过SurfaceView显示取景画面
   camera.setPreviewDisplay(surfaceHolder);

   // 4.开始预览
   camera.startPreview();

   // 5.自动对焦
   camera.autoFocus(afcb);

   // 6.调用Camera的takePicture()方法进行拍照.
   camera.takePicture(null, null , myjpegCallback);
```

整体流程大致是先获取系统camera, 再设置camera的参数,设置SurfaceHolder, 然后就可以startPreview了. 

基于此, 下面来看DreamCamera是怎么做的:

##### OpenCamera

在CameraActivity onCreate的时候,  便起了一个线程**mOncreateCameraController**去init CameraControl类, 然后再起一个线程**mOncreateOpencamera**去openCamera,

```java
    @Override
    public void onCreateTasks(Bundle state) {
        ...
        // open camera control task
        CameraFilmstripDataAdapter.THREAD_POOL_EXECUTOR
            .execute(mOncreateCameraController);
        // open camera task
        CameraFilmstripDataAdapter.THREAD_POOL_EXECUTOR
            .execute(mOncreateOpencamera);
        ...
    }

    // oncreate - camera controller
    private Runnable mOncreateCameraController = new Runnable() {

        @Override
        public void run() {
            try {
                mCameraController = new CameraController(mAppContext,
                        CameraActivity.this, mCameraRequestHandler,
                        CameraAgentFactory.getAndroidCameraAgent(mAppContext,
                                CameraAgentFactory.CameraApi.API_1),
                        CameraAgentFactory.getAndroidCameraAgent(mAppContext,
                                CameraAgentFactory.CameraApi.AUTO),
                        mActiveCameraDeviceTracker);
            } catch (AssertionError e) {
            }
        }
    };

    // oncreate - request open camera
    private Runnable mOncreateOpencamera = new Runnable() {

        @Override
        public void run() {
				...
                mCameraController.requestCamera(mDataModuleManager.getInstance(CameraActivity.this).get TempCameraModule().getInt(Keys.KEY_CAMERA_ID, 0),
                        mCurrentModule.useNewApi());
            }
        }
    };
```
CameraController在init的时候会通过*framework*的接口**CameraAgentFactory.getAndroidCameraAgent**判断是使用camera1还是使用camera2, 默认传入参数**AUTO**,如果是L之前的版本则使用camera1,否则使用camera2

```java
        if (Build.VERSION.SDK_INT >= FIRST_SDK_WITH_API_2 || Build.VERSION.CODENAME.equals("L")) {
            return CameraApi.API_2;
        } else {
            return CameraApi.API_1;
        }
```
拿到系统支持的camera, 之后在CameraControl中边去openCamera, 并传入CameraOpenCallback.

```java
    private static void checkAndOpenCamera(CameraAgent cameraManager,
            final int cameraId, Handler handler, final CameraAgent.CameraOpenCallback cb) {
        try {
            cameraManager.openCamera(handler, cameraId, cb);
        } catch (CameraDisabledException ex) {
        }
    }
```
这里的cameraManger是*framework/ex*下的抽象类CameraAgent,  该抽象类有两个实现类,分别对应着camera1和camera2

- [SprdAndroidCameraAgentImpl.java](http://10.0.1.79:8081/source/xref/sprdroid8.1_trunk_18b/vendor/sprd/platform/frameworks/ex/camera2/portability/src/com/android/ex/camera2/portability/SprdAndroidCameraAgentImpl.java)
- [AndroidCamera2AgentImpl.java](http://10.0.1.79:8081/source/xref/sprdroid8.1_trunk_18b/frameworks/ex/camera2/portability/src/com/android/ex/camera2/portability/AndroidCamera2AgentImpl.java)

CameraAgent内部最终会执行cameraOpen的操作

```java
public void openCamera(final Handler handler, final int cameraId,
                       final CameraOpenCallback callback) {
    try {
        //往dispatchThread插入一个runnable
        getDispatchThread().runJob(new Runnable() {
            @Override
            public void run() {
                //通过handle, message的方式open camera
                getCameraHandler().obtainMessage(CameraActions.OPEN_CAMERA, cameraId, 0,
                        CameraOpenCallbackForward.getNewInstance(handler, callback)).sendToTarget();
            }
        });
    } catch (final RuntimeException ex) {
        getCameraExceptionHandler().onDispatchThreadException(ex);
    }
}
```
DispatchThread线程内部有一个队列, 在run方法中不停的poll队列, 如果队列为空,或者队列满了就会阻塞队列.最终是通过CameraHandle进行消息的传递, CameraHandler尤其子类进行实现.



最终camera打开成功会回调我们之前传入的callback-->CameraOpenCallback.onCameraOpened, 并得到一个Camera对象 **CameraAgent.CameraProxy**

之后便回调CameraActivity进行下一步操作

##### camera参数设置

```java
CameraActivity.java:702
    @Override
    public void onCameraOpened(CameraAgent.CameraProxy camera) {
    	...
        resetExposureCompensationToDefault(camera); //设置曝光补偿
    	...
        mDataModuleManager.getCurrentDataModule().initializeStaticParams(camera);
        // 在对应Module中做一些cameraOpen的准备工作
        mCurrentModule.onCameraAvailable(camera);
}

DataModulePhoto.java:820
    @Override
    public void initializeStaticParams(CameraProxy proxy) {
    	...
    	setEVSPicturesize...
    	setExposureCompensationValue...
        setFrontFlash...
        setEVSAIDetect...
        ...
   }
```
在PhotoModule中的onCameraAvailable中会执行startPreView, 而onCameraAvailable和startPreView中都会设置了camera参数.

```java
In onCameraAvailable()
        // Set a default flash mode and focus mode
        if (mCameraSettings.getCurrentFlashMode() == null) {
            mCameraSettings.setFlashMode(CameraCapabilities.FlashMode.NO_FLASH);
        }
        if (mCameraSettings.getCurrentFocusMode() == null) {
            mCameraSettings.setFocusMode(CameraCapabilities.FocusMode.AUTO);
        }

In startPreView()
	private void updateSettingsBeforeStartPreview() {
        mActivity.waitUpdateSettingsCounter();
        updateParametersPictureSize(); //更新图片的大小
        updateCameraParametersInitialize(); //设置预览的帧速
        mCameraSettings.setDefault(mCameraId, mCameraCapabilities);
        if (!mIsImageCaptureIntent) {
            updateParametersZsl();
            updateParametersBurstMode(); //设置连拍模式
            updateParametersMirror(); //设置前置摄像头镜像
            updateParametersEOIS();
            updateParameters3DNR(); //3D数字降噪
            updateParametersThumbCallBack(); 
            updateParametersColorEffect(); //色彩效果
        }
    }

    private void updateLeftSettings() {
        if (!mBurstWorking) {
            setDisplayOrientation(); //设置显示方向
        }
        updateCameraParametersZoom();
        setAutoExposureLockIfSupported();
        setAutoWhiteBalanceLockIfSupported();  //自动白平衡
        setFocusAreasIfSupported();  //对焦范围
        setMeteringAreasIfSupported();  //测量范围

        mFocusManager.overrideFocusMode(null);
        mCameraSettings.setFocusMode(mFocusManager.getFocusMode(mCameraSettings
                .getCurrentFocusMode(),mDataModule.getString(Keys.KEY_FOCUS_MODE)));  //对焦模式
        SessionStatsCollector
                .instance()
                .autofocusActive(
                        mFocusManager.getFocusMode(mCameraSettings
                                .getCurrentFocusMode(),mDataModule.getString(Keys.KEY_FOCUS_MODE)) == CameraCapabilities.FocusMode.CONTINUOUS_PICTURE);
        if (mIsImageCaptureIntent) {
            updateParametersFlashMode();  //闪光灯模式
            mDataModuleCurrent.set(Keys.KEY_CAMERA_HDR, false);
        } else {
            updateParametersAntibanding();  // 颜色过度
            updateParametersPictureQuality();  //图片质量
            updateParametersExposureCompensation();  //曝光补偿
            updateParametersSceneMode();  //场景选择
            updateParametersWhiteBalance();  //白平衡
            updateParametersBurstCount();  //连拍张数
            updateParametersFlashMode();  //闪光灯模式
            updateParametersContrast();  //对比度
            updateParametersBrightness();  //亮度
            updateParametersISO();  //感光度
            updateParametersMetering();  //测光
            updateTimeStamp();  //时间戳
            updateParametersHDR();  //HDR
            updateParametersNormalHDR();  //正常曝光HDRx
            updateParametersSensorSelfShot(); //sensor
        }
        updateParametersSaturation(); //饱和度
        updateCameraShutterSound();  //快门声音
        updateParametersGridLine();  //网格线
        if (mContinuousFocusSupported && ApiHelper.HAS_AUTO_FOCUS_MOVE_CALLBACK) {
            updateAutoFocusMoveCallback();  //自动对焦
        }
        updateMakeLevel();  //美颜
        applySettings();
    }


```

##### setPreviewDisplay & startPreview

StartPreView 之后会setPreviewDisplay

`                mCameraDevice.setPreviewDisplay(mActivity.getCameraAppUI().getSurfaceHolder());`

然后又会new出一个runnable和一个callback, 再将callback交给doStartPreview执行, doStartPreview里才会去正在执行camera.startPreview, 并回调StartPreview里的callback.

`mCameraDevice.startPreview();`

`startPreviewCallback.onPreviewStarted();`

该callback会设置 `mCameraDevice.setSurfaceViewPreviewCallback` 这个callback是在*vender/framework/ex/*下面, 最终会回调该callback中的onSurfaceUpdate, 该方法会起一个线程执行module的init.

```java
    @Override
    public void onCameraAvailable(CameraProxy cameraProxy) {
        ...
        startPreview(true);
        ...
            
    }
    protected void startPreview(boolean optimize) {
        ...
        mCameraDevice.setPreviewDisplay(mActivity.getCameraAppUI().getSurfaceHolder());
        
        //在 doStartPreview中回调的callback 
        CameraAgent.CameraStartPreviewCallback startPreviewCallback = new CameraAgent.CameraStartPreviewCallback() {
            @Override
            public void onPreviewStarted() {
				...
                //由CameraDevices Callback 触发的 runnable
                final Runnable hideModeCoverRunnable = new Runnable() {
                    @Override
                    public void run() {
                        if (!mPaused) {
                            final CameraAppUI cameraAppUI = mActivity.getCameraAppUI();
                            cameraAppUI.onSurfaceTextureUpdated(
                                  cameraAppUI.getSurfaceTexture());
                            cameraAppUI.pauseTextureViewRendering();
                        }
                    }
                };
                if (useNewApi()) {
                    mCameraDevice.setSurfaceViewPreviewCallback(
                            //camera devices中的callback
                            new CameraAgent.CameraSurfaceViewPreviewCallback() {
                                @Override
                                public void onSurfaceUpdate() {
									...
                                    mHandler.post(hideModeCoverRunnable);
                                }
                            });
                }
            }
        };
        doStartPreview(startPreviewCallback, mCameraDevice);
    }
    
protected void doStartPreview(CameraAgent.CameraStartPreviewCallback startPreviewCallback, CameraAgent.CameraProxy cameraDevice) {
            mCameraDevice.startPreview();
            startPreviewCallback.onPreviewStarted();
    		...
}

```
##### camera 预览流程图

camera 预览流程图如下, framework部分的逻辑暂略

![](https://raw.githubusercontent.com/kevinkissum/picture/master/camera_open.png)





### Camera Capture

根据之前了解的android 原生camera capture的逻辑, 是先camera.autofocus, 然后再camera.takeCapture. 

##### camera.autofocus

拍摄的button为

```xml
            <com.android.camera.ShutterButton
                android:id="@+id/shutter_button"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_gravity="center"
                android:clickable="true"
                android:contentDescription="@string/accessibility_shutter_button"
                android:focusable="false"
                android:scaleType="center"
                android:src="@drawable/ic_camera" />
```
ShutterButton的初始化在CameraAppUI,  当点击时,点击事情交给moduleControl处理,即 PhotoModule or VideoModule

```java
        if (mShutterButton == null) {
            mShutterButton = (ShutterButton) mCameraRootView
                    .findViewById(R.id.shutter_button);
            addShutterListener(mController.getCurrentModuleController()); //设置点击监听
            addShutterListener(this);

        }

    @Override
    public boolean performClick() {
		...
        if (getVisibility() == View.VISIBLE && isClickable()
            for (OnShutterButtonListener listener : mListeners) {
                listener.onShutterCoordinate(mTouchCoordinate);
                mTouchCoordinate = null;
                listener.onShutterButtonClick();
            }
        }
        return result;
    }
```
在ModuleControl的**onShutterButtonClick**中会进行一系列条件的满足判断, 接下来会分两种情况:

- 如果是倒计时模式就init一个cancel button, 进行CountDown计时, 最后回调onCountDownFinished 再进行focus处理.
- 非计时模式直接进行focus的处理

```java
PhotoModule.java
	public void onShutterButtonClick(boolean checkShutterOnBurstWorking) {
        if (mPaused
                || mActivity.isWaitToChangeMode()
                || (mCameraState == SWITCHING_CAMERA)
				...
            return;
        }

        // Do not take the picture if there is not enough storage.
        if (!mIsImageCaptureIntent && mActivity.getStorageSpaceBytes() <= Storage.LOW_STORAGE_THRESHOLD_BYTES) {
            return;
        }
        
		...
		
        int countDownDuration  = 0;
        if(mDataModuleCurrent.isEnableSettingConfig(Keys.KEY_COUNTDOWN_DURATION)){
            countDownDuration = mDataModuleCurrent
                    .getInt(Keys.KEY_COUNTDOWN_DURATION);
        }
        mTimerDuration = countDownDuration;
        if (countDownDuration > 0) {
            initCancelButton();
            // Start count down.
            mAppController.getCameraAppUI().transitionToCancel();
            mUI.startCountdown(countDownDuration); //倒计时模式 
            return;
        } else {
            if (!isBurstCapture()) {
                mAppController.setShutterEnabled(false);
            }
            mAppController.getCameraAppUI().setBottomPanelLeftRightClickable(false);
            focusAndCapture();
        }
    }

CountDownView.java
	//计时结束之后的回调,由CountDownView.java触发
    @Override
    public void onCountDownFinished() {
		...
        focusAndCapture();
    }

```
DreamCamera用一个FocusManager去处理focus请求事件,  将逻辑从module中剥离出来, 处理完由callback返回module.

`mFocusManager.focusAndCapture(mCameraSettings.getCurrentFocusMode());`

Debug发现因为当前focusMode为continuous-picture , DreamCamera不会去执行focus, 


```java
    if (!needAutoFocusCall(currentFocusMode) && mState != STATE_FOCUSING) {
            // Focus is not needed.
            Log.i(TAG, "Focus is not needed." +  mState);
            capture();
        }
    
	private boolean needAutoFocusCall(CameraCapabilities.FocusMode focusMode) {
        return !(focusMode == CameraCapabilities.FocusMode.INFINITY
                || focusMode == CameraCapabilities.FocusMode.FIXED
                || focusMode == CameraCapabilities.FocusMode.EXTENDED_DOF
                || focusMode == CameraCapabilities.FocusMode.CONTINUOUS_PICTURE
                || focusMode == CameraCapabilities.FocusMode.CONTINUOUS_VIDEO);
    }
```


这里延伸看一下这个**focusMode**的设置

在一开始的Camera参数设置时, default模式为 AUTO

`mCameraSettings.setFocusMode(CameraCapabilities.FocusMode.AUTO)`

但在之后UpdateSettings, 依据xml中配置的值又进行了重新设置.

`mCameraSettings.setFocusMode(mFocusManager.getFocusMode(mCameraSettings.getCurrentFocusMode(),mDataModule.getString(Keys.KEY_FOCUS_MODE)));`

mDataModule是DataModuleBasic.java, 该类会在Module init时, 由ModuleManager去初始化.

 其中就有**mSupportDataMap**数据.

```java
    public String getString(String key) {
        synchronized (mLock)  {
            String value = null;
            // mSupportDataMap是从dream_camera_arrays_camera_part中加载出来的所有数组
            //其中就包含key-->pref_camera_focusmode_key_array
            if (mSupportDataMap != null && mSupportDataMap.containsKey(key)) {
                DataStorageStruct data = (DataStorageStruct) mSupportDataMap.get(key);
                value = getString(data.mStorePosition, key, data.getDefaultString());
                if (!data.isContainValue(value)) {
                    value = data.mDefaultValue;
                }

            }
			...
            return value;
         }
    }
```
dream_camera_arrays_camera_part.xml

```xml
    <integer-array name="pref_camera_focusmode_key_array">
        <item>@string/pref_camera_focusmode_key</item>
        <item>@integer/storage_position_category_bf</item>
        <item>@string/pref_camera_focusmode_default</item>  //该值即为 continuous-picture
        <item>@array/pref_camera_focusmode_entryvalues</item>
        <item>@array/pref_camera_focusmode_entryvalues</item>
    </integer-array>
```
所以会跳过focus, 回调PhotoModule进行Capture, 拍照成功回调传入的**JpegPictureCallback**,

```java
    @Override
    public boolean capture() {
        ...
        mCameraDevice.takePicture(mHandler,new ShutterCallback(false),mRawPictureCallback, mPostViewPictureCallback, new JpegPictureCallback(loc));
        ...
            
    }
    
private final class JpegPictureCallback implements CameraPictureCallback {

        @Override
        public void onPictureTaken(final byte[] originalJpegData, final CameraProxy camera) {
            ...
            saveFinalPhoto(originalJpegData, name, exif, camera, burstMode);
            ...
        }
            
```

##### Camera capture 流程图

camera capture 简要流程图如下, framework部分暂略.

![](https://raw.githubusercontent.com/kevinkissum/picture/master/camera_capture.png)



##### Camera thumbnail

缩略图的开始在**JpegPictureCallback::onPictureTaken**中

```java
    @Override
    public void onPictureTaken(final byte[] originalJpegData,
            final CameraProxy camera) {
		...
        if (!mIsImageCaptureIntent
                && !isSetFreezeFrameDisplay()
                && !inBurstMode()) {
            if (getModuleTpye() != DreamModule.INTERVAL_MODULE) {
                mThumbnailHasInvalid = true;
            }
            if (mIsFirstCallback || isHdr() && isHdrNormalOn() && (mIsHdrPicture && isNeedThumbCallBack() || !isNeedThumbCallBack())) {
                //开始显示缩略图
                startPeekAnimation(originalJpegData);
            }
        }
```
startPeekAnimation主要是实例化**Exif**类,该类原始media下的,Exif存储图像额外的拍摄参数信息.

```java
public void startPeekAnimation(final byte[] jpegData) {
    ExifInterface exif = Exif.getExif(jpegData);
    Bitmap bitmap = exif.getThumbnailBitmap();
    int orientation = Exif.getOrientation(exif);
    //改变存储单位?
    if (bitmap == null) {
        long t1 = System.currentTimeMillis();
        //use for test large yuv data
        byte[] decodeBuffer = new byte[32 * 1024];
        BitmapFactory.Options opts = new BitmapFactory.Options();
        opts.inSampleSize = 8;
        opts.inTempStorage = decodeBuffer;
        bitmap = BitmapFactory.decodeByteArray(jpegData, 0, jpegData.length/*, opts*/);
        long t2 = System.currentTimeMillis();
    }
    long t3 = System.currentTimeMillis();
    //修改图片旋转角度
    if (orientation != 0 && bitmap != null) {
        Matrix matrix = new Matrix();
        matrix.setRotate(orientation);
        Bitmap rotatedBitmap = bitmap;
        bitmap = Bitmap.createBitmap(rotatedBitmap, 0, 0,
                bitmap.getWidth(), bitmap.getHeight(), matrix, true);
        rotatedBitmap.recycle();
    }
	//间隔拍摄的特殊处理
    if (getModuleTpye() == DreamModule.INTERVAL_MODULE) {
        if (bitmap != null) {
            updateIntervalThumbnail(bitmap);
        }
    } else {
        mActivity.startPeekAnimation(bitmap);
    }
}
```
在CameraActivity中主要是另起一个runner去执行缩略图的更新.

```java
private void indicateCapture(final Bitmap indicator, final int rotationDegrees) {
    if(mainUiThumbnailRunnable != null && mMainHandler != null){
        mMainHandler.removeCallbacks(mainUiThumbnailRunnable);
    }

    mainUiThumbnailRunnable = new Runnable() {
        @Override
        public void run() {
            Log.i(TAG, "indicateCapture");
            mCameraAppUI.startCaptureIndicatorRevealAnimation(mCurrentModule
                    .getPeekAccessibilityString());
            mCameraAppUI.updateCaptureIndicatorThumbnail(indicator, rotationDegrees);
        }
    };
    if (Looper.getMainLooper() != Looper.myLooper()) {
        mMainHandler.post(mainUiThumbnailRunnable);
    } else {
        mainUiThumbnailRunnable.run();
    }
}
```
通过CameraAppUI去更新缩略图,CameraAppUI拥有**mRoundedThumbnailView**的实例,主要分两步:

- mRoundedThumbnailView.startRevealThumbnailAnimation(accessibilityString); 
- mRoundedThumbnailView.setThumbnail(thumbnailBitmap, rotation);

###### TODO

RoundedThumbnailView的动画略复杂, 暂不细说.



这里延伸探讨一下camera启动时缩略图的加载

##### Thumbnail load

由CameraActivity的traceLog可以知道加载流程:

```java
09-03 20:28:25.968 24346 24346 D kk      : requestLoad  java.lang.Throwable
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.data.CameraFilmstripDataAdapter.requestLoad(CameraFilmstripDataAdapter.java:92)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.CameraActivity.loadFilmstripItems(CameraActivity.java:3994)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.CameraActivity.onPreviewStarted(CameraActivity.java:949)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.PhotoModule.onPreviewStarted(PhotoModule.java:965)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.PhotoModule$16.onPreviewStarted(PhotoModule.java:3832)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.PhotoModule.doStartPreview(PhotoModule.java:5197)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.PhotoModule.startPreview(PhotoModule.java:3876)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.PhotoModule.onCameraAvailable(PhotoModule.java:2239)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.CameraActivity.onCameraOpened(CameraActivity.java:526)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.camera.app.CameraController.onCameraOpened(CameraController.java:190)
09-03 20:28:25.968 24346 24346 D kk      : 	at com.android.ex.camera2.portability.CameraAgent$CameraOpenCallbackForward$1.run(CameraAgent.java:133)
```

CameraActivity通过dataAdapter去loading

```java
public void loadFilmstripItems() {
    if (!mSecureCamera) {
        if (!isCaptureIntent()) {
            mFilmstripLoadStartTime = SystemClock.uptimeMillis();
            //loading, 并传入callback
            mDataAdapter.requestLoad(new Callback<Void>() {
                @Override
                public void onCallback(Void result) {
                    fillTemporarySessions();
                }
            });
        }
    }
}
```
dataAdapter::requestLoad是启动一个AsyncTask执行load过程

```java
private class QueryTask extends AsyncTask<Context, Void, QueryTaskResult> {
    private final Callback<Void> mDoneCallback;

    public QueryTask(Callback<Void> doneCallback) {
        mDoneCallback = doneCallback;
    }

    /**
     * Loads all the photo and video data in the camera folder in background
     * and combine them into one single list.
     *
     * @param contexts {@link Context} to load all the data.
     * @return An {@link CameraFilmstripDataAdapter.QueryTaskResult} containing
     *  all loaded data and the highest photo id in the dataset.
     */
    @Override
    protected QueryTaskResult doInBackground(Context... contexts) {
        final Context context = contexts[0];
        FilmstripItemList l = new FilmstripItemList();
        //查找时间最新的Photo和video
        PhotoItem photoData = mPhotoItemFactory.queryLatest();
        VideoItem videoData = mVideoItemFactory.queryLatest();

        long lastPhotoId = FilmstripItemBase.QUERY_ALL_MEDIA_ID;

        if(videoData == null && photoData == null) {
            // do nothing , return directly
        }
        else {
            //获取两者创建的时间
            Date videoDate = (videoData == null) ? new Date(0L) : videoData.getData().getCreationDate();
            Date photoDate = (photoData == null) ? new Date(0L) : photoData.getData().getCreationDate();
			//进行比较, 留下最新的那一个
            if(videoDate.after(photoDate))
                l.add(videoData);
            else
                l.add(photoData);
            FilmstripItem data = l.get(0);
            lastPhotoId = data.getData().getContentId();
            MetadataLoader.loadMetadata(context, data);
        }

        return new QueryTaskResult(l, lastPhotoId);
    }

    @Override
    protected void onPostExecute(QueryTaskResult result) {
        // Since we're wiping away all of our data, we should always replace any existing last
        // photo id with the new one we just obtained so it matches the data we're showing.
        mLastPhotoId = result.mLastPhotoId;
        //替换最新的数据,并回调callback
        replaceItemList(result.mFilmstripItemList);
        if (mDoneCallback != null) {
            mDoneCallback.onCallback(null);
        }
    }
}
```
这个看一下queryLast是一个标准的查询语句,

`FilmstripContentQueries.forCameraPathLimitLatest(mContentResolver, PhotoDataQuery.CONTENT_URI, PhotoDataQuery.QUERY_PROJECTION, PhotoDataQuery.THUMBNAIL_QUERY_ORDER, this));`

`PhotoDataQuery.CONTENT_URI = MediaStore.Images.Media.EXTERNAL_CONTENT_URI;`

```javascript
public static final String[] QUERY_PROJECTION_STATIC = {
    MediaStore.Images.ImageColumns._ID,           // 0, int
    MediaStore.Images.ImageColumns.DISPLAY_NAME,  // 1, string
    MediaStore.Images.ImageColumns.MIME_TYPE,     // 2, string
    MediaStore.Images.ImageColumns.DATE_TAKEN,    // 3, int
    MediaStore.Images.ImageColumns.DATE_MODIFIED, // 4, int
    MediaStore.Images.ImageColumns.DATA,          // 5, string
    MediaStore.Images.ImageColumns.ORIENTATION,   // 6, int, 0, 90, 180, 270
    MediaStore.Images.ImageColumns.WIDTH,         // 7, int
    MediaStore.Images.ImageColumns.HEIGHT,        // 8, int
    MediaStore.Images.ImageColumns.SIZE,          // 9, long
    MediaStore.Images.ImageColumns.LATITUDE,      // 10, double
    MediaStore.Images.ImageColumns.LONGITUDE,     // 11, double
};
```
`PhotoDataQuery.THUMBNAIL_QUERY_ORDER = MediaStore.Images.ImageColumns.DATE_TAKEN + " DESC, " + MediaStore.Images.ImageColumns._ID + " DESC";`

按date_taken的顺序进行排序,并获取第一个.

Item的初始化在PhotoItemFactory中进行 --> **public PhotoItem get(Cursor c) {** , 其又通过XXXDataFactory进行ItemData初始化, 并把ItemData封装到Item中.

`PhotoItemData data = mPhotoDataFactory.fromCursor(mContentResolver, c);`

`return new PhotoItem(mContext, data, this);`



当查询结束后, 会更新数据**mFilmstripItemList**,并回调callback.

接下来在CameraActivity便进行thumbnail的加载了,

fillTemporarySessions --> runSyncThumbnail --> thumbnailRunnable --> syncThumbnail(获取Thumbnail数据) --> syncThumbnailFromData(调用indicateCapture,流程同上)

```java
public void syncThumbnail() {
    //adapter中的list即是刚刚更新过的list
    FilmstripItem data = mDataAdapter.getItemAt(0);
    if (mSecureCamera && mDataAdapter.getTotalNumber() == 1) {
        data = null;
    }
...
```
##### Thumbnail load 流程图

![](https://raw.githubusercontent.com/kevinkissum/picture/master/camera_thumbnail_load.png)





##### Camera savePhoto

主要逻辑在**saveFinalPhoto**中,其根据intent判断是否直接存储还是在UI上显示预览图片

```java
        if (!mIsImageCaptureIntent) {
			...
        } else {
            Log.i(TAG, "saveFinalPhoto mQuickCapture=" + mQuickCapture);
            mJpegImageData = jpegData;
            if (!mQuickCapture) {
                Log.v(TAG, "showing UI");
                mUI.showCapturedImageForReview(jpegData, orientation,
                        mMirror);
            } else {
                onCaptureDone();
            }
        }
```
暂分析**mIsImageCaptureIntent=false**的情况, 其通过调用**getServices().getMediaSaver().addImage(**进行图片写入数据库,而该service的实现在**CameraServicesImpl**,mediaSaver的实现是**mediaSaverImpl**, 其内部通过AsyncTaks实现ImageAdd, 最终执行是在**Storage.java**中, 插入成功会返回Uri.

Photo实现了**OnMediaSavedListener**监听,并将listener通过addImage传入MediaSaveImpl, 插入成功后mediaSave会进行回调, 传回Uri.



### Camera 前后摄切换

各UI如果有Switch会bindUI, button定义Key **BUTTON_CAMERA_DREAM**, 

```java
public void bindCameraButton() {
    ButtonManagerDream buttonManager = (ButtonManagerDream) mActivity
            .getButtonManager();
    buttonManager.initializeButton(ButtonManagerDream.BUTTON_CAMERA_DREAM,
            mBasicModule.mCameraCallback);
}
```
Button init 设置监听在ButtonManagerDream中

```java
ButtonManagerDream.java
private void initializeCameraButton(final MultiToggleImageButton button,
        final ButtonCallback cb, final ButtonCallback preCb, int resIdImages) {
    ...

    button.setOnStateChangeListener(new MultiToggleImageButton.OnStateChangeListener() {
        @Override
        public void stateChanged(View view, int state) {
            int prefCameraId = mDataModule.getInt(Keys.KEY_CAMERA_ID);
            if (state != prefCameraId) {
                button.setEnabled(false);
                if (cb != null) {
                    //该cb是bindbt时传进来的, cb为PhotoModule中的callback
                    cb.onStateChanged(state);
                }
                //实现在CameraAPPUI中, 已弃用
                mAppController.getCameraAppUI().onChangeCamera();
            }
        }
    });
}
```
回调PhotoModule callback.

```java
PhotoModule.java
public final ButtonManager.ButtonCallback mCameraCallback = new ButtonManager.ButtonCallback() {
    @Override
    public void onStateChanged(int state) {
        if (mPaused
                || mAppController.getCameraProvider().waitingForCamera()) {
            return;
        }
        ButtonManager buttonManager = mActivity.getButtonManager();
        buttonManager.disableCameraButtonAndBlock();
        mPendingSwitchCameraId = state;
        //*
        switchCamera();
    }
};
```


这里需要注意一点的是的switchCamera方法
`protected void switchCamera() {`
    `//this method has been override by DreamPhotoModule`

PhotoModule的方法已经被子类重写, 这里调用不会执行PhotoModule反而去执行子类的方法 ---------> 多态

而子类SwitchCamera就干了一件事情执行activity的 **switchFrontAndBackMode** (冻屏, module select)

```java
CameraActivity.java    
public void switchFrontAndBackMode() {

        if (!isCaptureIntent()) {
            mDataModule.set(Keys.KEY_CAMERA_SWITCH, true);
        } else {
            mDataModule.set(Keys.KEY_INTENT_CAMERA_SWITCH, true);
        }
        int module = mCurrentModule.getMode();// current module

        int cameraId = mCurrentModule.getDeviceCameraId();// will change to camera id

        DreamUtil dreamUtil = new DreamUtil();

        if (DreamUtil.BACK_CAMERA == DreamUtil.getRightCamera(cameraId)) {
            nextCameraId = DreamUtil.getRightCameraId(DreamUtil.FRONT_CAMERA);
        } else {
            nextCameraId = DreamUtil.getRightCameraId(DreamUtil.BACK_CAMERA);
        }
        int modeIndex = dreamUtil.getRightMode(mDataModule, module, nextCameraId);
        if (isCaptureIntent()) {
            modeIndex = getModeIndex();
        }

        final int moduleType = mCurrentModule.getModuleTpye();
        if ( moduleType != DreamModule.FILTER_MODULE
                && moduleType != DreamModule.QR_MODULE) {
            if (CameraUtil.isSwitchAnimationEnable()) {
                mCameraAppUI.startSwitchAnimation(null);
            }
            mCameraRequestHandler.post(mPreSwitchRunnable);
			//冻屏操作, 如果开启blur, 则虚化
            freezeScreenCommon();
        }
        final int index = modeIndex;
        mMainHandler.post(new Runnable() {
            @Override
            public void run() {
                if (!mPaused) {
                    // For BACK to FRONT, FRONT to BACK.
                    mDataModule.set(Keys.KEY_CAMERA_ID, nextCameraId);
                    onModeSelected(index);
                }
            }
        });

        mCameraAppUI.updateModeList();
    }
```





### Camera Video

录制的大致流程是先配置好`MediaRecorder`相关参数，并在预览开始时调用`MediaRecorder.start()`,正式开始录像.

录制视频依然是由ShutterButton触发, 只不过module变成VideoModule. 

```java
    @Override
    public void onShutterButtonClick() {
        ...
        //条件判断是否正在录制 or 是否有足够的空间
        boolean stop = mMediaRecorderRecording;
        if (stop) {
            mAppController.getCameraAppUI().updateExtendPanelUI(View.VISIBLE);
            onStopVideoRecording();
        } else {
            if (mActivity.getStorageSpaceBytes() <= Storage.LOW_STORAGE_THRESHOLD_BYTES) {
                return;
            }
            mAppController.cancelFlashAnimation();
            //准备录制
            VideoRecording();
        }
```

startVideoRecording会传入一个callback到CameraActivty, 通过asyncTask去查询Storage情况,

```java
private void startVideoRecording() {
    ...
    //activity异步查询storagespace情况
    mActivity.updateStorageSpaceAndHint(new CameraActivity.OnStorageUpdateDoneListener() {
            @Override
            public void onStorageUpdateDone(long bytes) {
					...
                    //设置MediaRecorder相关参数
                    initializeRecorder(); 
                    ...
                    //Recording is now started
                    mMediaRecorder.start();
     });           
}       
```
我们看一下MediaRecorder设置了哪些,

```java
	private void initializeRecorder() {
		//是否开启ShutterSound
        updateCameraShutterSound();

		//由CameraDevices获取 mediaRecorder
        mMediaRecorder = (MediaRecorder) DreamProxy.getMediaRecoder();
		//当release时,mCameraDevice上锁, 所以在里需要解锁,不然无法使用record
        mCameraDevice.unlock();

        Camera camera = mCameraDevice.getCamera();

        mMediaRecorder.setCamera(camera);
        
		...
            
        //setAudioSource
        mMediaRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA);

        mMediaRecorder.setProfile(mProfile);
        mMediaRecorder.setVideoSize(mProfile.videoFrameWidth,
                mProfile.videoFrameHeight);
        mMediaRecorder.setMaxDuration(mMaxVideoDurationInMs);

        setRecordLocation();

        // Set output file.
        // Try Uri in the intent first. If it doesn't exist, use our own
        // instead.
        if (mVideoFileDescriptor != null) {
            mMediaRecorder.setOutputFile(mVideoFileDescriptor
                    .getFileDescriptor());
        } else {
            generateVideoFilename(mProfile.fileFormat);
            mMediaRecorder.setOutputFile(mVideoFilename);
        }

        // Set maximum file size.
        try {
            mMediaRecorder.setMaxFileSize(maxFileSize);
        } catch (RuntimeException exception) {

        }
		
		//设置水平方向
        mMediaRecorder.setOrientationHint(getRecorderOrientationHint(rotation));

        try {
            mMediaRecorder.prepare();
        } catch (IOException e) {
            Log.e(TAG, "prepare failed for " + mVideoFilename, e);
            releaseMediaRecorder();

        }

    }

```
当再次点击ShutterButton时,仍然是走之前的逻辑, 只不过经过判断执行**stopVideoRecording**, 释放资源, 重新上锁, 重新Preview等

```java
    private boolean stopVideoRecording(boolean shouldSaveVideo,boolean checkStorage) {

        // 释放audioFocus
        abandonAudioPlayback();
        //重设为OriginalRingerMode
        restoreRingerMode();

        //UI更新
        mAppController.getCameraAppUI()
                .setShouldSuppressCaptureIndicator(false);
        updateMakeUpUI(View.VISIBLE);
        
        if (mMediaRecorderRecording) {
            boolean shouldAddToMediaStoreNow = false;

            try {
                mMediaRecorder.setOnErrorListener(null);
                mMediaRecorder.setOnInfoListener(null);
				//停止录制
                mMediaRecorder.stop();
				...
            } catch (RuntimeException e) {

            }
            mMediaRecorderRecording = false;
            isShutterButtonClicked = false;

			//如果被打断则释放资源并关闭camera
            if (mPaused) {
                stopPreview();
                closeCamera();
            }
            // The orientation was fixed during video recording. Now make it
            // reflect the device orientation as video recording is stopped.
            mUI.setOrientationIndicator(0, true);
            mActivity.enableKeepScreenOn(false);
            
            if (shouldAddToMediaStoreNow && !fail && shouldSaveVideo) {
                if (mVideoFileDescriptor == null) {
                    //保持video
                    saveVideo();
                } else if (shouldShowResult()) {
                    // if no file save is needed, we can show the post capture
                    // UI now
                    showCaptureResult();
                }
            }
        }
        // release media recorder
        releaseMediaRecorder();

        if (!mPaused && mCameraDevice != null) {
            //设置focus, 并讲devices上锁
            setFocusParameters();
            mCameraDevice.lock();
            if (!ApiHelper.HAS_SURFACE_TEXTURE_RECORDING) {
                stopPreview();
                // Switch back to use SurfaceTexture for preview.
                startPreview();
            }
            // Update the parameters here because the parameters might have been
            // altered
            // by MediaRecorder.
            mCameraSettings = mCameraDevice.getSettings();
        }
		...
            
        return fail;
    }
```
##### Camera video 流程图

![](https://raw.githubusercontent.com/kevinkissum/picture/master/camera_video_reocrd.png)