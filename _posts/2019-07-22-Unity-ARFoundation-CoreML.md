---
layout:     post
title:      "Unity AR Foundation and CoreML: Hand detection and tracking"
subtitle:   ""
date:       2019-07-22 12:00:00
author:     "chenjd"
header-img: "img/2019-07-23/handdetection_1.png"
tags:
    - Unity
    - AR Foundation
    - XR
---

>This post was first published at [Medium](https://medium.com/@chen_jd/unity-ar-foundation-and-coreml-hand-detection-and-tracking-b74c592206c5)



## 0x00 Description

The [AR Foundation](https://docs.unity3d.com/Packages/com.unity.xr.arfoundation@1.5/manual/index.html) package in Unity wraps the low-level API such as ARKit, ARCore into a cohesive whole.

The CoreML is a framework that can be harnessed to integrate machine learning models into your app on iOS platform.

This article and the demo project at the end of the article show how to enable the CoreML to work with AR Foundation in Unity. With AR Foundation in Unity and CoreML on iOS, we can interact with virtual objects with our hands. 

This article refers to Gil Nakache's article and uses the mlmodel used in his article. In his article, he describes how to implement these on the native iOS platform with Swift.

#### Version

Unity Version: 2018.3.13f1

Xcode Version: 10.2.1

The ARFoundation Plugin: 1.5.0-preview.5

iPhone 7: 12.3.1

## 0x01 Implementation

#### Import AR Foundation Plugin

For convenience, I use the local package import. This is very simple, just modify the manifest.json file in the package folder and add the local package in the project manifest.

```
    "com.unity.xr.arfoundation": "file:../ARPackages/com.unity.xr.arfoundation",
    "com.unity.xr.arkit": "file:../ARPackages/com.unity.xr.arkit
```

After importing the AR Foundation plugin, we can create some related components in the scene, such as AR Session, AR Session Origin.
![](/img/2019-07-23/handdetection_2.png)

Then in our script, listen to the `frameReceived` event to get the data for each frame.

        if (m_CameraManager != null)
        {
            m_CameraManager.frameReceived += OnCameraFrameReceived;
        }

#### Create a Swift plugin for Unity

In order for C# to communicate with Swift, we need to create an object-c file as a bridge.

In this way, C# can call the method in Object-C by `[DllImport("__Internal")]`. Then Object-C will call Swift via `@objc`. After importing `UnityInterface.h`, Swift can call the `UnitySendMessage` method to pass data to C#.

There is a [sample](https://github.com/chenjd/Unity-Hello-Swift). This project demonstrates how to create a Swift plugin for Unity and print "Hello, I'm Swift" in Unity.

In the Unity-ARFoundation-HandDetection project, the structure of the plugins folder is as follows:

```none
<Plugins>
  └── iOS
      ├── HandDetector
      │   ├── Native
      │   │  ├──HandDetector.swift
      │   │  └──HandDetectorBridge.mm
      │   └── Managed
      │      └──HandDetector.cs
      └── Unity
```



However, it should be noted that the Xcode project exported by Unity does not specify the version of Swift.
![](/img/2019-07-23/SwiftVersion.png)

So you can manually specify a version, or create a script in Unity to automatically set its version.

#### Import mlmodel

Add the HandModel to our Xcode project, then it will generate an Objective-C model class automatically. But I want the mlmodel to generate a Swift class. We can set it at **Build Settings/CoreML Model Compiler - Code Generation Language** from Auto to Swift.

Then we get an automatically generated Swift model class called HandModel.
![](/img/2019-07-23/handmodel.png)
Of course, if you don't want to add it manually, you can also add it automatically through a build post processing script in Unity.

#### How to get the ARFrame ptr from AR Foundation

After completing the above steps, the basic framework is built. Next, we will use CoreML to implement hand detection and tracking.

    @objc func startDetection(buffer: CVPixelBuffer) -> Bool {
        //TODO
        self.retainedBuffer = buffer
        let imageRequestHandler = VNImageRequestHandler(cvPixelBuffer: self.retainedBuffer!, orientation: .right)
        
        visionQueue.async {
            do {
                defer { self.retainedBuffer = nil }
                try imageRequestHandler.perform([self.predictionRequest])
            } catch {
                fatalError("Perform Failed:\"\(error)\"")
            }
        }
        
        return true
    }

In Swift, we need a `CVPixelBuffer` to create a `VNImageRequestHandler` to perform the hand detection. Usually we can get it from ARFrame.

    CVPixelBufferRef buffer = frame.capturedImage;

Therefore, the next question is how to get the ARFrame pointer of ARKit on iOS from the AR Foundation in C#, then pass the pointer to the Hand Detection plugin in Swift.

In AR Foundation, you can get a `nativePtr` from a `XRCameraFrame`,  which points to a struct on ARKit that looks like this:

    typedef struct UnityXRNativeFrame_1
    {
        int version;
        void* framePtr;
    } UnityXRNativeFrame_1;

and the `framePtr` points to the latest `ARFrame`.

Specifically, we can call `TryGetLatestFrame` defined in XRCamera​Subsystem to get a XRCameraFrame instance.

    cameraManager.subsystem.TryGetLatestFrame(cameraParams, out frame)

Then pass the nativePtr from C# to Object-C.

    m_HandDetector.StartDetect(frame.nativePtr);

In Object-C, we will get a `UnityXRNativeFrame_1` pointer and we can get the `ARFrame` pointer from UnityXRNativeFrame_1.

        UnityXRNativeFrame_1* unityXRFrame = (UnityXRNativeFrame_1*) ptr;
        ARFrame* frame = (__bridge ARFrame*)unityXRFrame->framePtr;
        
        CVPixelBufferRef buffer = frame.capturedImage

Once the ARFrame is acquired, it comes to the iOS development domain. Create a VNImageRequestHandler object and start performing the detection. Once the detection is complete, the detectionCompleteHandler callback is invoked and passes the result of the detection to Unity via `UnitySendMessage`.

    private func detectionCompleteHandler(request: VNRequest, error: Error?) {
        
        DispatchQueue.main.async {
            
            if(error != nil) {
                UnitySendMessage(self.callbackTarget, "OnHandDetecedFromNative", "")
                fatalError("error\(error)")
            }
            
            guard let observation = self.predictionRequest.results?.first as? VNPixelBufferObservation else {
                UnitySendMessage(self.callbackTarget, "OnHandDetecedFromNative", "")
                fatalError("Unexpected result type from VNCoreMLRequest")
            }
            
            let outBuffer = observation.pixelBuffer
            
            guard let point = outBuffer.searchTopPoint() else{
                UnitySendMessage(self.callbackTarget, "OnHandDetecedFromNative", "")
                return
            }
            
            UnitySendMessage(self.callbackTarget, "OnHandDetecedFromNative", "\(point.x),\(point.y)")
        }
    }

Then we will get the position data in viewport space.
![](/img/2019-07-23/handdetection_3.png)
Viewport space is normalized and relative to the camera. The bottom-left of the viewport is (0,0); the top-right is (1,1). The z position is in world units from the camera. 

Once we get the position in viewport space, we transform it from viewport space to world space via `ViewportToWorldPoint` function in Unity. Provide the function with a vector where the x-y components of the vector come from Hand Detection and the z component is the distance of the resulting plane from the camera. 

       var handPos = new Vector3();
       handPos.x = pos.x;
       handPos.y = 1 - pos.y;
       handPos.z = 4;//m_Cam.nearClipPlane;
       var handWorldPos = m_Cam.ViewportToWorldPoint(handPos);

We can create a new object in Unity with the world space position or move the old object to the world space position. In other words, the position of the object is controlled according to the position of the hand.
![](/img/2019-07-23/handdetection_4.png)



#### Post Process Build
As I said above, we can write a C# script in Unity to automatically set properties of the generated xcode project. For example, we can set the Swift version property in the Build Setting of a Xcode project. We can even add mlmodel file to the Build Phase, such as the Compile Sources Phase. We can use the PBXProject class defined in `UnityEditor.iOS.Xcode` namespace. PBXProject class defines many useful functions such as `AddBuildProperty`, `SetBuildProperty`, `AddSourcesBuildPhase`.

    [PostProcessBuild]
    public static void OnPostProcessBuild(BuildTarget buildTarget, string path)
    {
        if(buildTarget != BuildTarget.iOS)
        {
            return;
        }

        string projPath = path + "/Unity-iPhone.xcodeproj/project.pbxproj";
        
        var proj = new PBXProject();
        proj.ReadFromFile(projPath);
        var targetGUID = proj.TargetGuidByName("Unity-iPhone");

        //set xcode proj properties
        proj.AddBuildProperty(targetGUID, "SWIFT_VERSION", "4.0");
        proj.SetBuildProperty(targetGUID, "SWIFT_OBJC_BRIDGING_HEADER", "Libraries/Plugins/iOS/HandDetector/Native/HandDetector.h");
        proj.SetBuildProperty(targetGUID, "SWIFT_OBJC_INTERFACE_HEADER_NAME","HandDetector-Swift.h");
        proj.SetBuildProperty(targetGUID, "COREML_CODEGEN_LANGUAGE", "Swift");
        
        
        //add handmodel to xcode proj build phase.
        var buildPhaseGUID = proj.AddSourcesBuildPhase(targetGUID);
        var handModelPath = Application.dataPath + "/../CoreML/HandModel.mlmodel";
        var fileGUID = proj.AddFile(handModelPath, "/HandModel.mlmodel");
        proj.AddFileToBuildSection(targetGUID, buildPhaseGUID, fileGUID);
        
        proj.WriteToFile(projPath);

    }

![](/img/2019-07-23/handdetection_6.png)

## 0x02 Conclusion

With the AR Foundation in Unity and the CoreML, we can let Unity Chan stand on our fingers. 
![](/img/2019-07-23/handdetection_5.png)
This article is a brief description of the process for integrating CoreML and AR Foundation. I think that you can use them to make more interesting content.

Here is the demo project used in the article.

[Unity-ARFoundation-HandDetection](https://github.com/chenjd/Unity-ARFoundation-HandDetection)






#### Useful Links

https://heartbeat.fritz.ai/hand-detection-with-core-ml-and-arkit-f4c8da98e88e

https://medium.com/@kevinhuyskens/implementing-swift-in-unity-53e0b668f895




