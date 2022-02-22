# SDKVersion
## 참고자료
[Manage manifest files](https://developer.android.com/studio/build/manage-manifests#merge-manifests)<br>
[< uses-sdk >](https://developer.android.com/guide/topics/manifest/uses-sdk-element)
## sdkversion
Android에는 여러 가지 sdkversion 설정이 있고 app module의 build.gradle에 설정할 수 있고 manifest.xml의 <uses-sdk>를 통해서도 설정할 수 있다.<br>
하지만 Android developers Android Studio part의 Manage manifest files에서는 다음과 같이 경고한다.
```
Important: Build configurations from the build.gradle file override any corresponding attributes in the merged manifest file.
For example, the minSdkVersion from the build.gradle file overrides the matching attribute in the <uses-sdk> manifest element. 
To avoid confusion, you should simply leave out the <uses-sdk> element and define these properties only in the build.gradle file. 
For more details, see Configure your build.
```  
빌드 과정에서 gradle build는 여러 manifest file을 merge하게 되는데 sdkversion의 경우 build.gradle에 작성된 것이 manifest의 uses-sdk를 override하게 된다.
따라서, 혼란을 피하기 위해 그냥 build.gradle에만 정의하도록 한다.

## category of sdkversion
#### minSdkVersion
- application을 run하기 위해 필요한 device의 최소 API Level을 지정
- Android system은 API Level이 minSdkVersion보다 낮으면 app이 설치되는 것을 막는다.
- 지정하지 않으면 기본 1로 설정
- 만약 app이 모든 API level에 대응되지 않고 minSdkVersion도 설정하지 않았다면 app은 runtime때 crash된다.
  
#### targetSdkVersion
- app이 target으로 설정한 API Level
- 개발자가 systemd에게 target으로 설정한 API Level에 대해 테스트했음을 알리고 향후 호환성 유지를 위한 호환성 동작 경계 Level을 system에게 알리는 역할을 한다.
  - 물론 targetSdkVersion 이하부터 minSdkVersion까지에서도 run 할 수 있다.
- 만약 targetSdkVersion보다 API Level이 높다면 system은 호환성 동작을 실행한다.
  - targetSdkVersion에 맞게 디자인된 app의 모습으로, 예상대로 계속 실행되도록 한다.
  - 일치하게 되면 호환성 동작을 사용 중지할 수 있다.

#### maxSdkVersion
- app이 실행될 수 있는 최대 API Level을 지정
- 만약 API Level이 maxSdkVersion보다 높게 되면 system은 app이 사용자에게 보이지 않게 한다.
  - 만약 maxSdkVersion은 그대로인데 user의 device의 API Level이 과거보다 높아지게 되면 더이상 system은 app 유효성 검사를 할 수 없게 되어 app이 보이지 않게 한다.
  - 따라서 이는 호환성과 상관없이 새로운 API Level에 대한 app 배포를 차단하는 수단이 되기에 선언하는 것을 권장하지 않는다.
  
#### compileSdkVersion
