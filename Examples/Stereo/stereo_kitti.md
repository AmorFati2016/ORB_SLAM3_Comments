``` plantuml
@startuml
|stereo_kitti.cc|
:main;
:LoadImages;
:Create SLAM system;
|system.cc|
:TrackStereo;
|system.cc|
|LocalMapping.cc|

if (打开定位模式) then (yes)
|LocalMapping.cc|
:mpLocalMapper->RequestStop();
:mpLocalMapper->isStopped();
|LocalMapping.cc|
|tracking.cc|
:mpTracker->InformOnlyTracking(true);
|tracking.cc|
else (no)
|tracking.cc|
:mpTracker->InformOnlyTracking(false);
|tracking.cc|
|LocalMapping.cc|
:mpLocalMapper->Release();
endif
|LocalMapping.cc|
|tracking.cc|
:mpTracker->Reset();
note left
mbReset = true;
end note
:mpTracker->ResetActiveMap();
note left
mbResetActiveMap = true;
end note
:GrabImageStereo;
:mpTracker->GrabImuData(vImuMeas[i_imu]);
note left
IMU_STEREO
end note
|tracking.cc|
|Frame.cc|
:Frame();
:ExtractORB();
:UndistortKeyPoints();
:ComputeStereoMatches();
:AssignFeaturesToGrid();
|Frame.cc|
|tracking.cc|
:Track();
|Atlas.cc|
:mpAtlas->GetCurrentMap();
|Atlas.cc|
|tracking.cc|
if (NOT_INITIALIZED) then (no)
|tracking.cc|
:track pose;
:NeedNewKeyFrame();
:CreateNewKeyFrame();
note left
if need
end note
|tracking.cc|
else (yes)
|tracking.cc|
:StereoInitialization();
|tracking.cc|
|FrameDrawer.cc|
:mpFrameDrawer->Update(this);
|FrameDrawer.cc|
|Atlas.cc|
:mpAtlas->GetAllMaps();
|Atlas.cc|
endif
|tracking.cc|
|stereo_kitti.cc|
:Shutdown;
:SaveTrajectoryKITTI;
|stereo_kitti.cc|

@enduml