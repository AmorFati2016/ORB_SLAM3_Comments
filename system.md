```plantuml
@startuml
:system(strVocFile, strSettingsFile, sensor, bUseViewer, initFr);
:1. 检查SLAM系统传感器初始化类型, 需要用到 sensor;
note right
if(mSensor==MONOCULAR)
    cout << "Monocular" << endl;
...
end note
:2. 检查传感器配置文件,也就是相机标定参数等 , 需要用到 strSettingsFile;
note right
    cv::FileStorage fsSettings(strSettingsFile.c_str(),cv::FileStorage::READ);
    // opencv xml文件读取
end note
:3. 实例化ORB词典对象,并加载ORB词典;
note right
    mpVocabulary = new ORBVocabulary();
    bool bVocLoad = mpVocabulary->loadFromTextFile(strVocFile);
end note
:4. 根据ORB词典实例对象,创建关键帧DataBase;
note right
// 需要用到词典 mpVocabulary
mpKeyFrameDatabase = new KeyFrameDatabase(*mpVocabulary);
end note
:5. 创建Altas,也就是多地图系统;
note right
    // 0 表示 initKFid
    mpAtlas = new Atlas(0); 
end note
:6. IMU传感器初始化设置,如果需要的话. 
    也就是将Map.cc中的mbIsInertial变量设置为true;
note right
if (mSensor==IMU_STEREO ||
mSensor==IMU_MONOCULAR)
    mpAtlas->SetInertialSensor();

void Map::SetInertialSensor() {
    unique_lock<mutex> lock(mMutexMap);
    mbIsInertial = true;
}
end note
:7. 利用Atlas创建Drawer. 包括FrameDrawer和MapDrawer;
note right
    mpFrameDrawer = new FrameDrawer(mpAtlas);
    mpMapDrawer = new MapDrawer(mpAtlas, strSettingsFile);
end note
:8. 初始化Tracking线程;
note right
mpTracker = new Tracking(this, mpVocabulary, mpFrameDrawer, mpMapDrawer,
                mpAtlas, mpKeyFrameDatabase, strSettingsFile, mSensor, strSequence);
end note
:9. 初始化局部地图LocalMapping线程，并加载;
note right
mpLocalMapper = new LocalMapping(this, mpAtlas,mSensor==MONOCULAR 
                || mSensor==IMU_MONOCULAR, mSensor==IMU_MONOCULAR
                || mSensor==IMU_STEREO, strSequence);
mptLocalMapping = new thread(&ORB_SLAM3::LocalMapping::Run,mpLocalMapper);
// 这里用到了传入的参数 initFr
mpLocalMapper->mInitFr = initFr;
mpLocalMapper->mThFarPoints = fsSettings["thFarPoints"];
end note
:10. 初始化闭环LoopClosing线程，并加载;
note right
mpLoopCloser = new LoopClosing(mpAtlas, mpKeyFrameDatabase, mpVocabulary, mSensor!=MONOCULAR); 
// mSensor!=MONOCULAR);
mptLoopClosing = new thread(&ORB_SLAM3::LoopClosing::Run, mpLoopCloser);
end note
:11. 初始化用户可视化线程, 并加载;
note right
// 参数 bUseViewer 在这里用到
if(bUseViewer) {
    mpViewer = new Viewer(this, mpFrameDrawer,mpMapDrawer,mpTracker,strSettingsFile);
    mptViewer = new thread(&Viewer::Run, mpViewer);
    mpTracker->SetViewer(mpViewer);
    mpLoopCloser->mpViewer = mpViewer;
    mpViewer->both = mpFrameDrawer->both;
}
end note
:12. 将上述实例化的对象指针, 传入需要用到的线程, 方便进行数据共享;
note right
// Tracking 需要用到 mpLocalMapper 和闭环 mpLoopCloser
mpTracker->SetLocalMapper(mpLocalMapper);
mpTracker->SetLoopClosing(mpLoopCloser);

// LocalMapper 需要用到 Tracking 和闭环 mpLoopCloser
mpLocalMapper->SetTracker(mpTracker);
mpLocalMapper->SetLoopCloser(mpLoopCloser);

// mpLoopCloser 需要用到 Tracking 和 LocalMapper
mpLoopCloser->SetTracker(mpTracker);
mpLoopCloser->SetLocalMapper(mpLocalMapper);
end note
@enduml