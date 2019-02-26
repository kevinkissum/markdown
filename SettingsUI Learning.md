# SettingsUI Learning
[TOC]



#### 1.Setting 加载及显示

CameraAppUI 中实现了手势监听类 GestureDetector, 并将 mPreviewOverlay 的Touch事件交给该类去执行, 从而完成滑动监听:

`mPreviewOverlay.setOnTouchListener(new MyTouchListener());`

```java
CameraAppUI.java
private class MyTouchListener implements View.OnTouchListener {
        private boolean mScaleStarted = false;

        @Override
        public boolean onTouch(View v, MotionEvent event) {
            if (event.getActionMasked() == MotionEvent.ACTION_DOWN) {
                mScaleStarted = false;
            } else if (event.getActionMasked() == MotionEvent.ACTION_POINTER_DOWN) {
                mScaleStarted = true;
            }
            // mGestureDetector 手势监听类
            return (!mScaleStarted) && mGestureDetector.onTouchEvent(event);
        }
    }
```
Settings的加载只有在朝左滑动时触发:
```java
		...
        @Override
        public boolean onScroll(MotionEvent e1, MotionEvent ev,
        ...
		} else if (swipeState == SWIPE_LEFT) {
            if(mModeListView != null && mModeListView.isShown()){
                return;
            }
            showSettingsUI(swipeState);
        } else if (swipeState == SWIPE_RIGHT) {
       ...
```

SettingsUI是一个自定义layout, 由一个返回的title和fragment组成, 

```xml
<com.dream.camera.settings.DreamUIPreferenceSettingLayout 
	....
     >

    <LinearLayout
        ...
        android:orientation="horizontal" >

        <ImageView
            android:id="@+id/return_image"
        	...
         />

        <TextView
			...
            android:text="@string/pref_camera_settings_category"
         />
    </LinearLayout>

    <FrameLayout
        android:id="@+id/setting_frament_container"
		...
     />

</com.dream.camera.settings.DreamUIPreferenceSettingLayout>
```
layout包含在ViewStub组件中,只有在需要时会去inflate, 

```java
public void initSettingLayout(SettingUIListener listener){
        ViewStub viewStubSettingLayout = (ViewStub) mCameraRootView
                .findViewById(R.id.dream_ui_preference_setting_layout_id);
        if (viewStubSettingLayout != null) {
            viewStubSettingLayout.inflate();
            mSettingLayout = (DreamUIPreferenceSettingLayout) mCameraRootView
                    .findViewById(R.id.dream_ui_preference_setting_layout);
        if (listener != null && listener != uiListener) {
            mSettingLayout.changeModule(listener);
            uiListener = listener;
        }
    }
```

在 `mSettingLayout.changeModule(listener)` 中去完成fragment的填充
```java
    public void changeModule(SettingUIListener listener){

        if(mCurrentFragment != null){
            mCurrentFragment.releaseResource();
        }

        mCurrentFragment = new DreamUIPreferenceSettingFragment();
        mActivity.getFragmentManager().beginTransaction()
                .replace(R.id.setting_frament_container, mCurrentFragment)
                .commitAllowingStateLoss();
        mSettingUIListener = listener;
    }
```

**DreamUIPreferenceSettingFragment** 继承的是PreferenceFragment, 它加载的xml是 

`addPreferencesFromResource(R.xml.dream_camera_preferences);`

```xml
<PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android"
    android:key="@string/preference_key_screen_camera_root" >

    <com.dream.camera.settings.DreamUISettingPartCamera
    ...
    />

    <com.dream.camera.settings.DreamUISettingPartPhoto
    ...
    />

    <com.dream.camera.settings.DreamUISettingPartVideo
    ...
    />

</PreferenceScreen>
```

可以看到 SettingsUI 一共分3块显示, 

- CameraPart
- PhotoPart
- VideoPart

其中CameraPart是一直显示的,而Photo和Video会依据当前configType选择性显示

```java
private void initialize() {
        mRoot = (PreferenceScreen) findPreference(getResIDString(R.string.preference_key_screen_camera_root));
        mCameraPart = (DreamUISettingPartBasic) findPreference(getResIDString(R.string.preference_key_category_camera_root));
        mPhotoPart = (DreamUISettingPartBasic) findPreference(getResIDString(R.string.preference_key_category_photo_root));
        mVideoPart = (DreamUISettingPartBasic) findPreference(getResIDString(R.string.preference_key_category_video_root));

        mCameraPart.changContent();
    }

    private void changeUIVisibility() {
        if (mDataSetting.mCategory
                .equals(DataConfig.CategoryType.CATEGORY_PHOTO)) {
            setPhotoModuleVisiblity();
        } else if (mDataSetting.mCategory
                .equals(DataConfig.CategoryType.CATEGORY_VIDEO)) {
            setVideoModuleVisibility();
        }

    }
    
    private void setPhotoModuleVisiblity() {
        recursiveDelete(mRoot, mVideoPart);
        mPhotoPart.changContent();
    }

    private void setVideoModuleVisibility() {
        recursiveDelete(mRoot, mPhotoPart);
        mVideoPart.changContent();
    }
```

这里着重介绍一下Part的.changConten()方法. 

```java
//子类DreamUISettingPartCamera调用changContent,
@Override
public void changContent() {
    mDataModule = DataModuleManager.getInstance(getContext())
            .getDataModuleCamera();
    //子类DreamUISettingPartPhoto Or DreamUISettingPartVideo, 注意此处module的改变
    /*mDataModule = DataModuleManager.getInstance(getContext())
				.getCurrentDataModule();*/
    mDataModule.addListener((DreamSettingResourceChangeListener)this);
    super.changContent();
}
```
```java
  	//父类DreamUISettingPartBasic
	public void changContent() {
        
        updatePreItemAccordingConfig(this);

        // 该方法为抽象方法, 子类依据自身特点进行Preference的更新
        updatePreItemsAccordingProperties();

        // 对List类型的Preference进行设置title和Summary
        fillEntriesAndSummaries(this);

        // 设置监听
        initializeData(this);
    }
```

**updatePreItemAccordingConfig**

这个方法主要是动态加载不同module下的设置项, 

```java
private void updatePreItemAccordingConfig(PreferenceGroup group) {
    	//获得该Part所有的Prefernce的Key
        ArrayList<String> keyList = getAllPreKeyList(group);
    	//ShowItems存的是该module需要显示的Preference的key.
        //通过减法 remove 就可以得到不需要的Preference
        keyList.removeAll(mDataModule.getShowItemsSet());
    	//将不需要的Preference移除掉
        for (int i = 0; i < keyList.size(); i++) {
            Log.e(TAG, "remove key = " + keyList.get(i));
            Preference pref = group.findPreference(keyList.get(i));
            if (pref != null) {
                group.removePreference(pref);
            }
        }
    }
```

这里再深究一下 `mDataModule.getShowItemsSet()`

当我们切换module时候, 无论是Photo Or Video 类型的module, 最终都会由所有module的父类,PhotoModule 或者 VideoModule, 通过DataModuleManager更新当前module的属性.

**PhotoModule** or **VideoModule**

```java
@Override
    public void init(CameraActivity activity, boolean isSecureCamera,
                     boolean isCaptureIntent) {
		...
        DataStructSetting dataSetting = new DataStructSetting(
                DreamUtil.intToString(getMode()), DreamUtil.isFrontCamera(
                mAppController.getAndroidContext(), mCameraId),
                mActivity.getCurrentModuleIndex(), mCameraId);

        // change the data storage module
        DataModuleManager.getInstance(mAppController.getAndroidContext())
                .changeModuleStatus(dataSetting);
        ...
```

**状态类** **DataStructSetting**,

```java
    public DataStructSetting(String category, boolean isFront, String mode,
            int cameraID) {
        mCategory = category; //类型 Photo or Video
        mIsFront = isFront;   //前置 or 后置
        mMode = mode;		  //module值, 之后靠它获取对应的设置项数组
        mCameraID = cameraID; 
    }
```

最后在 **DataModuleBasic** 进行 initializeData, 通过状态类 DataStructSetting 获取到不同module下的array id, 然后转换成data map.

```java
    public void initializeData(DataStructSetting dataSetting) {
        mDataSetting = dataSetting;

        // generate support configuration data resourceID
        //该数组显示的是ListPreference中的内容
        int supportdataResourceID = DreamSettingUtil
                .getSupportDataResourceID(dataSetting);

        // generate support data map
        if (supportdataResourceID != -1) {
            generateSupportDataList(supportdataResourceID);
        }

        // initialize show item resourceID
        int showItemSetID = DreamSettingUtil
                .getPreferenceUIConfigureID(dataSetting);

        // generate show item data
        if (showItemSetID != -1) {
            generateShowItemList(showItemSetID);
        }
        // setEntryAndEntryValues for list
        fillEntriesAndSummaries();
    }
```

获取对应module对应的数组ID

```java
public static int getPreferenceUIConfigureID(DataStructSetting dataSetting) {
    int resourceID = -1;
    // camera configuration resource
    if (dataSetting.mCategory
            .equals(DataConfig.CategoryType.CATEGORY_CAMERA)) {
        resourceID = R.array.camera_public_setting_display;//camera Part的设置项
        return resourceID;
    }
    // photo configuration resource
    if (dataSetting.mCategory
            .equals(DataConfig.CategoryType.CATEGORY_PHOTO)) {
        if (dataSetting.mIsFront) {
            resourceID = getPrefUIFrontPhoto(dataSetting.mMode , resourceID);//PhotoPart前置的设置项
        } else {
            resourceID = getPrefUIBackPhoto(dataSetting.mMode , resourceID);//PhotoPart后置的设置项
        }
        return resourceID;
    }

    // video configuration resource
    if (dataSetting.mCategory
            .equals(DataConfig.CategoryType.CATEGORY_VIDEO)) {
        if (dataSetting.mIsFront) {
            resourceID = getPrefUIFrontVideo(dataSetting.mMode , resourceID);//VideoPart前置的设置项
        } else {
            resourceID = getPrefUIBackVideo(dataSetting.mMode , resourceID);//VideoPart后置的设置项
        }
        return resourceID;
    }

    return resourceID;
}
```
举个栗子--> CameraPart 数组

```xml
<integer-array name="camera_public_setting_display">
    <item>@string/pref_camera_storage_path</item>
    <item>@string/pref_camera_volume_key_function</item>
    <item>@string/pref_shutter_sound_key</item>
    <item>@string/pref_camera_recordlocation_key</item>
    <item>@string/pref_camera_quick_capture_key</item>
</integer-array>
```
至此, SettingsUI的加载和显示已经完成.

PhotoPart和VidoPart切换会根据其module对应的数组进行动态的修改.

#### 2,数据存储

Settings数据以sharePreference形式存储在 `/data/data/com.android.camera2/shared_prefs` 中

依据Camera, Photo , Video, 前置, 后置 一共分为7个表

- camera_camera_setting.xml
- camera_category_photo_setting.xml
- camera_category_photo_front_setting.xml
- camera_category_photo_back_setting.xml
- camera_category_video_setting.xml
- camera_category_video_front_setting.xml
- camera_category_video_back_setting.xml

`R.xml.dream_camera_preferences` 中的Key 和 xml 中的key 一一对应





