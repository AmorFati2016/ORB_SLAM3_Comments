```plantuml
@startuml
title TrackSensor

:1. 检查SLAM系统传感器类型，如果不是指定的传感器则退出;
note right
    // TrackStereo
    if(mSensor!=STEREO && mSensor!=IMU_STEREO) exit(-1);

    // TrackRGBD
    if(mSensor!=RGBD) exit(-1);
    
    // TrackMonocular
    if(mSensor!=MONOCULAR && mSensor!=IMU_MONOCULAR)  exit(-1);
end note
:2. 检查定位模式状态;
note right
    unique_lock<mutex> lock(mMutexMode);
    if(mbActivateLocalizationMode) {

        // 如果是定位模式，请求LocalMapper线程停止建图
        mpLocalMapper->RequestStop();

        // Wait until Local Mapping has effectively stopped
        while(!mpLocalMapper->isStopped()) {
            usleep(1000);
        }

        // Track线程模式切换为只做Tracking
        mpTracker->InformOnlyTracking(true);
        
        // 将默认定位状态置为false，默认同时做Tracking和建图
        mbActivateLocalizationMode = false;
    }

    // 如果关闭定位模式，则打开同时定位和建图
    if(mbDeactivateLocalizationMode) {
        mpTracker->InformOnlyTracking(false);
        mpLocalMapper->Release();
        mbDeactivateLocalizationMode = false;
    }
end note

:3. 检查重置状态，也就是重置Track线程或者Activate Map;
note right
    unique_lock<mutex> lock(mMutexReset);
    if(mbReset) {
        mpTracker->Reset();
        mbReset = false;
        mbResetActiveMap = false;
    } else if(mbResetActiveMap) {
        mpTracker->ResetActiveMap();
        mbResetActiveMap = false;
    }
end note
:4. 如果是IMU 传感器，需要加载IMU数据;
note right
// TrackStereo
if (mSensor == System::IMU_STEREO)
    for(size_t i_imu = 0; i_imu < vImuMeas.size(); i_imu++)
        mpTracker->GrabImuData(vImuMeas[i_imu]);
// TrackMonocular
if (mSensor == System::IMU_MONOCULAR)
    for(size_t i_imu = 0; i_imu < vImuMeas.size(); i_imu++)
        mpTracker->GrabImuData(vImuMeas[i_imu]);
end note
:5. Track 线程计算当前帧姿态参数;
note right
// TrackStereo
cv::Mat Tcw = mpTracker->GrabImageStereo(imLeft,imRight,timestamp,filename);

// TrackRGBD
cv::Mat Tcw = mpTracker->GrabImageRGBD(im,depthmap,timestamp,filename);

// TrackMonocular
cv::Mat Tcw = mpTracker->GrabImageMonocular(im,timestamp,filename);
end note
:6. 更新状态参数和数据;
note right
unique_lock<mutex> lock2(mMutexState);
mTrackingState = mpTracker->mState;
mTrackedMapPoints = mpTracker->mCurrentFrame.mvpMapPoints;
mTrackedKeyPointsUn = mpTracker->mCurrentFrame.mvKeysUn;
end note
@enduml