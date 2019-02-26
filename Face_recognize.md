# Face Recognize

[TOC]

在PreviewStart之后, 便根据当前模式ai 检测

```java
    protected void onPreviewStarted() {
        mAppController.onPreviewStarted();
		...
        updateFace();
		...
    }
    
    protected void updateFace() {
        mFace = mDataModuleCurrent.getString(Keys.KEY_CAMERA_AI_DATECT);
        Log.d(TAG, "face = " + mFace + "; hdrState = " + isHdr());
        if (!mFace.equals(Keys.CAMERA_AI_DATECT_VAL_OFF) && isCameraIdle()) {
            startFaceDetection();
        } else if (mFace.equals(Keys.CAMERA_AI_DATECT_VAL_OFF)|| !isCameraIdle()) {
            stopFaceDetection();
        }
    }   
 
    @Override
    public void startFaceDetection() {
        mCameraDevice.setFaceDetectionCallback(mHandler, mUI);
        mCameraDevice.startFaceDetection();
        mFaceDetectionStarted = true;
        mUI.onStartFaceDetection(mDisplayOrientation, isCameraFrontFacing());
        SessionStatsCollector.instance().faceScanActive(true);
    }    
```
## 准备FaceDetection
android 提供 API 去设置 face detect, 但原生只在API1中有实现, API2是空实现, 展讯自实现了face detect.
#### 1. 设置Callback
`mCameraDevice.setFaceDetectionCallback(mHandler, mUI);`

API2空实现, sprd仿API中的写法
```java
        @Override
        public void setFaceDetectionCallback(Handler handler, CameraFaceDetectionCallback callback) {
            mCameraHandler.obtainMessage(
                    CameraActions.SET_FACE_DETECTION_LISTENER,
                    FaceDetectionCallbackForward.getNewInstance(handler,
                            SprdAndroidCamera2ProxyImpl.this, callback)).sendToTarget();
        }
        //在handle 收到 消息后会设置callback. ???? 为什么不直接setCallback?
      SprdAndroidCamera2AgentImpl.java  
                    case CameraActions.SET_FACE_DETECTION_LISTENER: {
                        setFaceDetectionListener((FaceDetectionCallbackForward) msg.obj);
                        break;
                    }        
```
通过查看callback的引用查看流程
#### 2. StartFaceDetection
`mCameraDevice.startFaceDetection();`
这里API1 直接执行 `mCamera.startFaceDetection();`
```java
CameraAgent.java
        public void startFaceDetection() {
            try {
                getDispatchThread().runJob(new Runnable() {
                    @Override
                    public void run() {
                        getCameraHandler().sendEmptyMessage(CameraActions.START_FACE_DETECTION);
                    }});
            } catch (final RuntimeException ex) {
                getAgent().getCameraExceptionHandler().onDispatchThreadException(ex);
            }
        }
        
SprdAndroidCamera2AgentImpl.java
                    case CameraActions.START_FACE_DETECTION: {
                        boolean isFace = mPersistentSettings.set(
                                CaptureRequest.STATISTICS_FACE_DETECT_MODE,
                                CaptureRequest.STATISTICS_FACE_DETECT_MODE_SIMPLE);
                        mLastFrameNumberOfFaces = 0;
                        break;
                    }        
```
API2依然是空实现, 需要自行实现.
startFaceDetection没能看出做什么， 之前setFaceDetectionCallback到可以回溯触发时间，sprdImpl实现了AndroidImpl2中的abstract 方法 onMonitorControlStates， 所以AndroidImpl2会调用该方法，通过callback CameraResultStateCallback触发， checkAe或AfState会触发， mCameraResultStateCallback还被传入mSession中，跟踪不下去了 ><.
`mSession.setRepeatingRequest(mPersistentSettings.createRequest(mCamera,CameraDevice.TEMPLATE_PREVIEW, mPreviewSurface),/*listener*/mCameraResultStateCallback, /*handler*/this);`
```java
AndroidCamera2AgentImpl.java
private void checkAfState(CaptureResult result) {
    if (result.get(CaptureResult.CONTROL_AF_STATE) != null &&
            !mAlreadyDispatched) {
        // Now our mCameraResultStateCallback will invoke the callback
        // the first time it finds the focus motor to be locked.
        mAlreadyDispatched = true;
        mOneshotAfCallback = callback;
        // This is an optimization: check the AF state of this frame
        // instead of simply waiting for the next.
        mCameraResultStateCallback.monitorControlStates(result);
}
    private void checkAeState(CaptureResult result) {
        if (result.get(CaptureResult.CONTROL_AE_STATE) != null &&
                !mAlreadyDispatched) {
            // Now our mCameraResultStateCallback will invoke the
            // callback once the autoexposure routine has converged.
            mAlreadyDispatched = true;
            mOneshotCaptureCallback = listener;
            // This is an optimization: check the AE state of this frame
            // instead of simply waiting for the next.
            mCameraResultStateCallback.monitorControlStates(result);
        }
    }

public void monitorControlStates(CaptureResult result) {
    onMonitorControlStates(result);

SprdAndroidCamera2AgentImpl.java
        protected void onMonitorControlStates(CaptureResult result) {
            monitorControlStatesAIDetect(result, mCameraProxy, mActiveArray);
            monitorControlStatesRangeFind(result, mCameraProxy);
            monitorSurfaceViewPreviewUpdate(mCameraProxy);
//callback             
    public void monitorControlStatesAIDetect(CaptureResult result,
            AndroidCamera2ProxyImpl cameraProxy,
            Rect activeArray) {
        Integer faceState = result.get(CaptureResult.STATISTICS_FACE_DETECT_MODE);
        if (faceState != null) {
            int aeState = faceState;
            switch (aeState) {
                case CaptureResult.STATISTICS_FACE_DETECT_MODE_SIMPLE:
                    android.hardware.camera2.params.Face[] faces = result
                            .get(CaptureResult.STATISTICS_FACES);
                    Camera.Face[] cFaces = new Camera.Face[faces.length];
                    if (faces.length == 0 && mLastFrameNumberOfFaces == 0) {
                        break;
                    }
                    mLastFrameNumberOfFaces = faces.length;
                    for (int i = 0; i < faces.length; i++) {
                        Camera.Face face = new Camera.Face();
                        face.score = faces[i].getScore();
                        face.rect = faceForConvertCoordinate(activeArray, faces[i].getBounds());
                        cFaces[i] = face;
                    }
                    if (mCameraHandler instanceof SprdCamera2Handler) {
                        SprdCamera2Handler handler = (SprdCamera2Handler) mCameraHandler;
                        if (handler.getFaceDetectionListener() != null) {
                        //回调
                            handler.getFaceDetectionListener().onFaceDetection(cFaces, cameraProxy);
                        }
                    }

                    break;
            }
        }            
```
PhotoUI实现
```java
    public void onFaceDetection(Face[] faces, CameraAgent.CameraProxy camera) {
        if (isNeedClearFaceView()
                || ((mController instanceof PhotoModule) && !((PhotoModule)mController).mFaceDetectionStarted)) {//now face is not mutex with ai in UE's doc.
            if (mFaceView != null) {
                mFaceView.clear();
            }
            return;
        }

        // SPRD: Add for new feature VGesture but just empty interface
        if (isDetectView()) {
            doDectView(faces,mFaceView);
        } else if (mAIController.isChooseFace() && mFaceView != null && faces != null) {
            mFaceView.setFaces(faces);
        } else if (mAIController.isChooseSmile()) {
            if (faces != null && mFaceView != null) {
                if (isCountingDown()) {
                    return;
                }
                mFaceView.clear();
                int length = faces.length;
                int[] smileCount = new int[length];
                for (int i = 0, len = faces.length; i < len; i++) {
                    Log.i(TAG, " len=" + len + " faces[i].score=" + faces[i].score);
                    // mAIController.resetSmileScoreCount(faces[i].score >= 90);
                    // SPRD: Fix bug 536674 The smile face score is low.
                    mAIController.resetSmileScoreCount(faces[i].score > 40 && faces[i].score < 100);
                    if (faces[i].score > 40 && faces[i].score < 100) {
                        smileCount[i] = faces[i].score;
                    }
                }
                SurfaceTexture st = mActivity.getCameraAppUI().getSurfaceTexture();
                // SPRD: Fix bug 536674 The smile face score is low.
                if (smileCount.length > 0 && smileCount[0] > 40 && smileCount[0] < 100
                        && !mActivity.isPaused() && st != null && mController.isShutterEnabled()
                        && !mFaceView.isPause()) {
                    mFaceSimleCount++;
                    if (mFaceSimleCount > 5) {
                        mFaceView.setFaces(faces);
                        mFaceView.clearFacesDelayed();
                        mFaceSimleCount = 0;
                        Log.i(TAG, "smileCount=" + smileCount[0] + "Do Capture ... ...");
                        if(mActivity.getCameraAppUI().isSettingLayoutOpen()){
                            return;
                        }
                        mController.onShutterButtonClick();
                        mController.setCaptureCount(0);
                    }
                }
            }
        }
    }
```







