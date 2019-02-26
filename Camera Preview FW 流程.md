# Camera Preview FW 流程

[TOC]

#### CameraAgent的初始化

在CameraActivity中会进行CameraAgent的初始化, 

```java
CameraActivity.java
private Runnable mOncreateCameraController = new Runnable() {

    @Override
    public void run() {
        long startTime = System.currentTimeMillis();
        mActiveCameraDeviceTracker = ActiveCameraDeviceTracker.instance();
        try {
            mCameraController = new CameraController(mAppContext,
                    CameraActivity.this, mCameraRequestHandler,
                    //ＡＰＩ 1 或 2
                    CameraAgentFactory.getAndroidCameraAgent(mAppContext,
                            CameraAgentFactory.CameraApi.API_1),
                    CameraAgentFactory.getAndroidCameraAgent(mAppContext,
                            CameraAgentFactory.CameraApi.AUTO),
                    mActiveCameraDeviceTracker);
            mCameraController
                    .setCameraExceptionHandler(new CameraExceptionHandler(
                            mCameraExceptionCallback, mMainHandler));
        } catch (AssertionError e) {
            Log.e(TAG, "Creating camera controller failed.", e);
            mFatalErrorHandler.onGenericCameraAccessFailure();
        }
        ....
};
  
  CameraAgentFactory.java  
    public static synchronized CameraAgent getAndroidCameraAgent(Context context, CameraApi api) {
        api = validateApiChoice(api);

        if (api == CameraApi.API_1) {
            if (sAndroidCameraAgent == null) {
                sAndroidCameraAgent = new AndroidCameraAgentImpl();
                sAndroidCameraAgentClientCount = 1;
            } else {
                ++sAndroidCameraAgentClientCount;
            }
            return sAndroidCameraAgent;
        } else { // API_2
            if (highestSupportedApi() == CameraApi.API_1) {
                throw new UnsupportedOperationException("Camera API_2 unavailable on this device");
            }

            if (sAndroidCamera2Agent == null) {
                //使用的是Sprd的Impl
                sAndroidCamera2Agent = new SprdAndroidCamera2AgentImpl(context);
                sAndroidCamera2AgentClientCount = 1;
            } else {
                ++sAndroidCamera2AgentClientCount;
            }
            return sAndroidCamera2Agent;
        }
    }
```
#### camera request

之后便进行camera request操作

```java
CameraActivity.java
private Runnable mOncreateOpencamera = new Runnable() {

    @Override
    public void run() {
        mCounterOncreateOpenCamera.waitCount();
        mOnCreateTime = System.currentTimeMillis();

        if (!isKeyguardLocked()
                && mCurrentModeIndex == getResources().getInteger(
                        R.integer.camera_mode_auto_photo)
                && checkAllCameraAvailable()) {
            mIsCameraRequestedOnCreate = true;
            mCameraController.requestCamera(mDataModuleManager.getInstance(CameraActivity.this).getTempCameraModule().getInt(Keys.KEY_CAMERA_ID, 0),
                                            
            ...
     
CameraController.java     
    private static void checkAndOpenCamera(CameraAgent cameraManager,
            final int cameraId, Handler handler, final CameraAgent.CameraOpenCallback cb) {
        Log.i(TAG, "checkAndOpenCamera");
        try {
            CameraUtil.throwIfCameraDisabled();
            //API2 得到的 CameraAgent, 至此进入fw
            cameraManager.openCamera(handler, cameraId, cb);
        } catch (CameraDisabledException ex) {
            handler.post(new Runnable() {
                @Override
                public void run() {
                    cb.onCameraDisabled(cameraId);
                }
            });
        }
    }
```
#### openCamera

只有基类CameraAgent实现了openCamera

```java
CameraAgent.java
public void openCamera(final Handler handler, final int cameraId,
                       final CameraOpenCallback callback) {
    try {
        //DispatchThread实现还是在AndroidCamera2AgentImpl中
        getDispatchThread().runJob(new Runnable() {
            @Override
            public void run() {
                //CameraHandler也是在AndroidCamera2AgentImpl中
                getCameraHandler().obtainMessage(CameraActions.OPEN_CAMERA, cameraId, 0,
                        CameraOpenCallbackForward.getNewInstance(handler, callback)).sendToTarget();
            }
        });
    } catch (final RuntimeException ex) {
        getCameraExceptionHandler().onDispatchThreadException(ex);
    }
}
```
CameraAgent获取其实现类AndroidCamera2AgentImp里的Handle进行发送OPEN_CAMERA的消息

```java
AndroidCamera2AgentImp.java
				case CameraActions.OPEN_CAMERA:
                case CameraActions.RECONNECT: {
                    CameraOpenCallback openCallback = (CameraOpenCallback) msg.obj;
                    int cameraIndex = msg.arg1;
					//Camera通过State值来保存camera的状态
   					//CAMERA_UNOPENED --> CAMERA_UNCONFIGURED --> CAMERA_CONFIGURED --> CAMERA_PREVIEW_READY --> CAMERA_PREVIEW_ACTIVE --> CAMERA_FOCUS_LOCKED         
                    if (mCameraState.getState() > AndroidCamera2StateHolder.CAMERA_UNOPENED) {
                        openCallback.onDeviceOpenedAlready(cameraIndex,
                                generateHistoryString(cameraIndex));
                        break;
                    }

                    mOpenCallback = openCallback;
                    mCameraIndex = cameraIndex;
					//Sprd 增加的需求, 夜景, 景深等模式camera id 就不是 0 or 1 了.
                    if (mCameraIndex == 5 || mCameraIndex == 6 || mCameraIndex == 7 || mCameraIndex == 12
                            || mCameraIndex == 13 || mCameraIndex == 15) {
                        mCameraId = "" + mCameraIndex;
                    } else {
                        mCameraId = mCameraDevices.get(mCameraIndex);
                    }
               
                    Log.i(TAG, String.format("Opening camera index %d (id %s) with camera2 API",
                            cameraIndex, mCameraId));

                    if (mCameraId == null) {
                        mOpenCallback.onCameraDisabled(msg.arg1);
                        break;
                    }
                    //这里的Manager是CameraManager : mCameraManager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);
                    mCameraManager.openCamera(mCameraId, mCameraDeviceStateCallback, this);

                    break;
                }
```
在CameraManager中最终调用到openCameraDeviceUserAsync进行openCamera
```java
    private CameraDevice openCameraDeviceUserAsync(String cameraId,
            CameraDevice.StateCallback callback, Handler handler, final int uid)
            throws CameraAccessException {
            //在getCameraCharacteristics中会进行CameraService的Binder操作，
        CameraCharacteristics characteristics = getCameraCharacteristics(cameraId);
        CameraDevice device = null;
		...
           //TODO 不明所以
	        ICameraDeviceUser cameraUser = null;
           //初始化devices实现类，该类提供Capture等操作
            android.hardware.camera2.impl.CameraDeviceImpl deviceImpl =
                    new android.hardware.camera2.impl.CameraDeviceImpl(
                        cameraId,
                        //上层的传下的callback
                        callback,
                        handler,
                        characteristics,
                        mContext.getApplicationInfo().targetSdkVersion);

            ICameraDeviceCallbacks callbacks = deviceImpl.getCallbacks();

                if (supportsCamera2ApiLocked(cameraId)) {
                    // Use cameraservice's cameradeviceclient implementation for HAL3.2+ devices
                    ICameraService cameraService = CameraManagerGlobal.get().getCameraService();
                    cameraUser = cameraService.connectDevice(callbacks, cameraId,
                            mContext.getOpPackageName(), uid);
                } else {
                    // Use legacy camera implementation for HAL1 devices
                    Log.i(TAG, "Using legacy camera HAL.");
                    cameraUser = CameraDeviceUserShim.connectBinderShim(callbacks, id);
                }
                ...
            //TODO 
            //回调之前impl传入的CameraDevicesStateCallback
            //setRemoteDevice --> mDeviceHandler.post(mCallOnOpened);
            deviceImpl.setRemoteDevice(cameraUser);
            device = deviceImpl;

        return device;
    }
```
这里跟踪CameraDevicesStateCallback， 该callback由Agent2Impl传入， 当impl进行setRemote时该callback会产生回调，
```java
    public void setRemoteDevice(ICameraDeviceUser remoteDevice) throws CameraAccessException {
            mRemoteDevice = new ICameraDeviceUserWrapper(remoteDevice);
                    remoteDeviceBinder.linkToDeath(this, /*flag*/ 0);
            mDeviceHandler.post(mCallOnOpened);//回调callback
            mDeviceHandler.post(mCallOnUnconfigured);
        }
    }
    private final Runnable mCallOnOpened = new Runnable() {
        @Override
        public void run() {
            if (sessionCallback != null) {
                sessionCallback.onOpened(CameraDeviceImpl.this);
            }
            //DeviceCallback即为AndroidCamera2AgentImp一路传下的callback
            mDeviceCallback.onOpened(CameraDeviceImpl.this);
        }
    };
   
```

因为此时CameraDevices已经生成，在**Agent2Impl**中则又会进行一个代理的装饰。
```java
            public void onOpened(CameraDevice camera) {
                mCamera = camera;
                if (mOpenCallback != null) {
                        /*
                         * SPRD @{
                         * Original Code
                         *
                        mCameraProxy = new AndroidCamera2ProxyImpl(AndroidCamera2AgentImpl.this,
                                mCameraIndex, mCamera, characteristics, props);
                         */
                        if (mSprdAgentImpl == null) {
                            mCameraProxy = new AndroidCamera2ProxyImpl(AndroidCamera2AgentImpl.this,
                                    mCameraIndex, mCamera, characteristics, props);
                        } else {
                            mCameraProxy = mSprdAgentImpl.new SprdAndroidCamera2ProxyImpl(
                                    mSprdAgentImpl, mCameraIndex, mCamera, characteristics, props);
                        }
						...
                        //回调上层app传入的callback
                        mOpenCallback.onCameraOpened(mCameraProxy); 
                }
            }
```
#### Preview

上层得到的最终是AndroidCamera2ProxyImpl实现类。之后顺序是从CameraControl-->CamraAcitvity-->PhotoModule-->CamreaAppUI,其中PhotoModule会执行Preview的操作。
SetPriview --> SurfaceView 展讯注释了原生，自实现其逻辑，在SprdAndroidCamera2AgentImpl中

```java
        protected void setPreviewDisplay(SurfaceHolder surfaceHolder) {
            // TODO: Must be called after providing a .*Settings populated with sizes
            if (mSession != null) {
                closePreviewSession();
            }
            mSurfaceHolder = surfaceHolder;
            if (mPreviewSurface != null) {
                mPreviewSurface.release();
            }
            mPreviewSurface = mSurfaceHolder.getSurface();
            if (mCaptureReader != null) {
                mCaptureReader.close();
            }
            //PhotoSize 在此前通过devices.applySettings已经实例化
            mCaptureReader = ImageReader.newInstance(
                    mPhotoSize.width(), mPhotoSize.height(), ImageFormat.JPEG, 1);
            if (mThumbnailReader != null) {
                mThumbnailReader.close();
            }
            List<OutputConfiguration> outConfigurations = new ArrayList<>();
            //PreviewSize 在此前通过devices.applySettings已经实例化
            OutputConfiguration surfaceViewOutputConfiguration= new OutputConfiguration(new Size(mPreviewSize.width(), mPreviewSize.height()), SurfaceHolder.class);
            surfaceViewOutputConfiguration.addSurface(mPreviewSurface);
            surfaceViewOutputConfiguration.enableSprd();
            outConfigurations.add(surfaceViewOutputConfiguration);
            outConfigurations.add(new OutputConfiguration(mCaptureReader.getSurface()));
            //生成缩略图的ImageRender
            if (mNeedThumb) {
                mThumbnailReader = ImageReader.newInstance(
                        mThumbnailSize.width(), mThumbnailSize.height(), ImageFormat.YUV_420_888, 1);
                outConfigurations.add(new OutputConfiguration(mThumbnailReader.getSurface()));
            }
            //createCaptureSession
            //CameraConstrainedHighSpeedCaptureSessionImpl
            //CameraCaptureSessionImpl
                mCamera.createCaptureSessionByOutputConfigurations(
                        outConfigurations,
                        mCameraPreviewStateCallback, this);

        }
```
Sprd中StartPreView会走两次，第一次会因为camera未open而return， 第一次是SurfaceView创建成功的时候，第二次是Camera Open成功的时候。
StartPreview中通过CameraAgent.CameraProxy驱动设备进行startPreview， cameraFW传递消息， 最终到msession处，

```java
                            mSession.setRepeatingRequest(
                                    mPersistentSettings.createRequest(mCamera,
                                            CameraDevice.TEMPLATE_PREVIEW, mPreviewSurface),
                                    /*listener*/mCameraResultStateCallback, /*handler*/this);
```
mSession最终还是由CameraDevicesImpl去实现。

