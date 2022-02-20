# Application Fundamentals

## 참고자료
[Android Developers: Application Fundamentals](https://developer.android.com/guide/components/fundamentals#ActivatingComponents)

## Android
#### [APK & Android App Bundle]
- Android SDK tools는 code, data, resource files를 합쳐 compile하여 APK or Android App Bundle를 생성한다.
  - APK(Android Package, .apk suffix)
    - runtime 때 필요한 Android app content를 포함한다.
    - APK 파일을 Android device에 설치하면 app을 설치할 수 있다.
  - Android App Bundle(.aab suffix)
    - runtime 때 필요하지 않은 additional metadata 포함 Android app project content를 포함한다.
      - AAB 파일을 Android device에 설치해도 app을 설치할 수 없다.
      - APK 생성과 서명을 나중으로 미룬다.
    - 구글에서는 구글 플레이 앱 배포를 APK 대신 AAB 형식을 올리도록 바꿨다.
      - 구글 플레이는 AAB로부터 APK를 생성하는데 각 device마다 필요로 하는 file들로만 구성된 optimized APK를 제공하게 된다.
      - 서명을 구글 플레이가 후에 진행한다.
#### [Android security features]
- Android OS는 multi-user Linux system이다.
  - 여기서 user는 각각의 app을 뜻한다.
- system은 각각의 app에 unique Linux user ID를 배정한다.
  - system은 user ID가 할당된 app에 있는 모든 파일에 대해 권한을 설정하고 이 user ID만이 파일들에 접근할 수 있도록 한다.
- 각 프로세스는 자체적인 VM이 있다.
  - 그렇기에 한 앱은 다른 앱과 격리된 상태로 실행된다.
- 각 앱은 각각의 Linux process에서 실행된다.
  - app component 중 하나가 실행되어야 한다면 Android system은 각각의 process를 시작하고 더 이상 필요하지 않거나 다른 앱을 위한 메모리 공간을 필요로 한다면 종료시킨다.
#### [principle of least privilege]
- 각 앱은 필요한 요소에만 접근을 하며 접근 권한이 주어진 시스템에만 접근할 수 있다.
  - 그래서 매우 안전한 환경
- 물론, 다른 앱과 data를 공유하거나 다른 시스템에 접근할 수는 있다.
  - Linux user ID를 공유하여 같은 Linux process, VM에서 실행할 수 있다.
  - device's location, camera, bluetooth connection에 대한 권한을 얻어 사용할 수 있다.
  
## App components
app components란 Android app을 구성하는 essential building block으로 시스템 혹은 user가 app에 접근할 수 있는 entry point가 되는 것이다.<br>
기본 4대 component로는 Activities, Services, Broadcast receivers, Content providers
#### [Activities]
- user interaction을 위한 entry point
- activity 하나는 single screen을 의미
- 여러 activities끼리 서로 함께 작업이 이루어져 결합된 user experience를 제공하지만 각각은 다른 것과 독립적
- activity는 또한 system과 app 사이의 interaction을 제공한다.
  - user가 현재 보는 screen을 추적해 system이 그 screen을 담당하는 activity를 호스팅하는 process를 계속 실행하도록 한다.
  - 지금은 stop된, 이전에 실행했던 activity를 호스팅한 process로 다시 돌아갈 것에 대해 우선순위를 높게 한다.
  - app이 process kill하는 시점을 알고 미리 이전 상태를 저장하여 그 상태를 복구시킨 activities로 돌아갈 수 있다 -> onSaveInstanceState()와 관련된 내용
  - Intent를 통해서 다른 app의 activity를 시작하는 것과 같이 서로 다른 app끼리 유기적으로 연결될 수 있고 그 흐름을 정의할 수 있다 -> Intent를 통한 app간의 흐름 관련 내용
- Activity class의 subclass로 실행

#### [Services]
- service는 여러 가지 이유로 발생하는 background operation을 위한 general-purpose entry point
  - long running operations가 background에서 실행될 수 있도록 하는 component
  - user interface를 제공하지 않는다.
  - background에서 실행된다고 해서 background thread에서 실행되는 것이 아니라 main thread에서 실행된다.
- Activity와 같은 다른 component에서 service를 시작하거나 bind할 수 있다.
- Services는 Started services, Bound services 두 종류로 나눈다.
  - Started services
    - Started services는 다시 foreground service와 regular background service로 나눈다.
    - foreground service는 현재 background 작업이 이루어지고 있음을 on-going notification과 함께 사용자에게 알린다.
      - 사용자가 현재 작업이 진행중임을 알고 있기 때문에 system은 service's process가 계속 running하도록 한다.
    - regular background service는 사용자가 running 중임을 모른다.
      - 그렇기에 foreground service와 달리 process 관리에 자유롭다
  - Bound services
    - app이나 system이 service 작업을 필요로 해서 running하는 service다.
    - 보통 다른 process가 필요로 하는 API를 제공하는 service
    - process A가 process B의 service에 bound되었다고 하자.
      - service를 포함한 process B는 process A를 위해 running한다.
      - user는 process A뿐 아니라 service 포함 process B까지 care한다.
- Service class의 subclass로 실행
- API 26이상부터는 background service가 제한되어 background인 상태에서는 foreground service, Bound service만 사용할 수 있다.
  - device의 제한된 resource를 사용하게 되면 시스템에 추가적인 부하가 걸려 user experience가 나빠질 수 있기 때문에 제한이 생겼다.
  - background가 되면 system이 자동으로 background service를 중단시킨다.
- 만약 target이 API level 21 이상이라면 JobScheduler를 사용하는 것이 좋다.
  - 작업이 언제 실행될지 타이밍을 정할 수 있어 필요할 때만 device를 깨울 수 있어 device 전력 소비에 좋다.
#### [Broadcast receivers]
- system이 app에게 event를 전달할 수 있도록 하는 app component로 app이 broadcast announcement에 반응하도록 한다.
- app이 실행하지 않아도 broadcast를 전달할 수 있다.
  - 그렇기에 broadcast가 올때까지 app을 실행시키고 있지 않아도 된다.
- 많은 broadcast는 system에서 발생시킨다.
  - screen turned off, low battery, picture captured..
  - 물론 앱이 다른 앱에게 event를 알리는 broadcast를 시작할 수도 있다.
- Broadcast receivers는 user interface를 갖고 있지 않기에 event가 발생했음을 상태 바 notification을 통해 알릴 수 있다.
- Broadcast는 미리 manifest에 등록시킨 static receiver와 필요할 때 등록해 사용하는 dynamic receiver로 나눈다.
  - Service와 마찬가지로 APi 26 이상부터 정책이 변경되어 resource limitation 이유로 대부분의 system broadcast를 static receiver로 전달할 수 없게 되었다.
- BroadcastReceiver class의 subclass로 실행
- 각각의 broadcast는 Intent object로 전달

#### [Content providers]
- app이 접근할 수 있는 file, SQLite database, on the web, persistent storage location에 저장할 수 있는 shared app data를 관리
  - Content providers가 허용해야 다른 app이 위에서 나열한 저장소에 있는 app data에 접근할 수 있다.
- 또다른 역할은 system이 URI로 식별할 수 있도록 명명된 data items를 생산할 수 있도록 한다.
  - URI(Uniform Resource Identifier)는 resource를 구별하는 식별자
  - app은 URI와 data를 mapping 시킬 수 있다.
  - 이 URI를 통해서 data에 접근하도록 한다.
- URI를 data에 할당하는 것은 app이 running 중일 필요가 없다.
  - app이 종료되어도 URI는 남아있다.
  - URI에 해당하는 app data를 가져올 때만 running중이면 된다.
- 다른 앱이 마음대로 app data에 접근할 수 없도록 한다.
  - 해당 URI를 통해서 접근하며 접근하고자 하면 system이 임시 URI 권한을 부여하게 된다.
  - 따라서 허용된 URI와 mapping된 data만 접근할 수 있고 이외는 접근할 수 없다.
- ContentProvider class의 subclass로 실행

Android system의 또 다른 특징은 다른 app의 component를 시작할 수 있다는 것이다.<br>
- 만약 app 내에서 camera app을 실행하고자 한다면 link를 통하거나 직접 개발할 필요 없이 camera app의 activity를 시작하면 된다.
- system은 component를 시작하면 그 component를 포함하는 app을 위한 process를 시작한다.
  - 다른 app의 component를 실행할 때 같은 process가 아닌 다른 app을 위한 process에서 실행시키게 된다.
  - 그렇기에 Android app은 하나의 entry point만을 가지지 않는다.
- 이렇듯 서로 다른 app은 서로 다른 process에서 작동하기 때문에 app이 직접 다른 app의 component를 activate 시킬 수는 없다.
  - Intent를 통해서 system에게 message를 보내서 activate 시킬 수 있다.
  
#### [Activating components]
- activities, services, broadcast receivers는 비동기 메시지 객체인 Intent를 통해 activate된다.
  - 같은 app 내의 component나 다른 app의 component를 activate 시키기 위한 message를 담아 system에게 제공하여 component를 activate 시킨다.
- Intent는 activate 시킬 특정 component를 지정하는 explicit intent와 특정 component type을 지정하는 implicit intent가 있다.
- Activities나 Services의 경우
  - action이나 data의 URI를 지정하여 component를 시작할 수 있다.
  - 또한, Intent를 통해서 다른 activity로부터 결과를 받거나 원하는 data의 URI를 받을 수도 있다.
  ```kotlin
  val intent = Intent(this, NextActivity::class.java)
  startActivity(Intent)
  ```
  ```kotlin
  val intent = Intent(Intent.ACTION_VIEW)
  intent.data = Uri.parse("https://www.naver.com")
  startActivity(intent)
  ```
  ```kotlin
  startActivityForResult(intent, REQUEST_CODE)
  ```
  ```kotlin
  val launcher = registerForActivityResult(ActivityResultContracts.startActivityResult){
    if (it.resultCode == 1){}
  }
  launcher.launch()
  ```
  ```kotlin
  val intent = Intent(this, NextService::class.java)
  startService(intent)
  ```
  ```kotlin
  ServiceConnection conn = ServiceConnection {
    override fun onServiceConnected(name: ComponentName?, service: IBinder?){...}
    override fun onServiceDisconnected(name: ComponentName?){...}
  }
  val intent = Intent(this, NextService::class.java)
  bindService(intent, conn, Context.BIND_AUTO_CREATE)
  ```
- Broadcast receivers의 경우
  - Intent는 broadcast할 announcement를 정의하여 activate한다.
  - receiver의 경우 정적 receiver의 경우 manifest에 등록, 동적 receiver의 경우 registerReceiver(Receiver, IntentFilter)를 통해 등록한다.
    - 이후 sendBroadcast(Intent)를 통해 실행한다.(순서 없이 등록된 receiver에게 전부 전달된다.)
    - sendOrderedBroadcast(Intent, receiverPermission: String)는 우선순위에 따라 순서대로 한번에 하나의 receiver에게만 전달되도록 한다.
    - sendStickyBroadcast(Intent)는 braodcast receiver 작업이 끝난 후 stick되어 빠르게 다른 곳에서도 결과를 얻을 수 있도록 한다.(API level 21에서 deprecated)
- Content Provider의 경우
  - Intent에 의해 activate되지 않는다.
  - ContentResolver를 통해서 activate된다.
  - ContentResolver가 content provider와의 모든 transaction을 담당하고 필요한 method를 실행하게 된다.
    - 즉, 다른 component들은 content provider를 직접 handle하는 것이 아니라 중간에 ContentResolver를 통해서 이루어지게 된다.
    - 이는 app data를 보호하는 역할도 한다.(for security)
    ```kotlin
    val cursor = ContentResolver.query(URI, ...)
    
## The manifest file
- Android system이 app component를 실행하기 전에 manifest file을 읽어 app component의 존재를 확인한다.
  - 따라서 AndroidManifest.xml file에 반드시 app component를 정의해야 한다.
- app component 정의 이외에도 다음과 같은 사항을 정의한다.
  - user permissions like INTERNET access, READ/WRITE access
  - app이 사용하는 API의 minimum API Level
  - app이 사용해야 하는 hardware, software features like camera, bluetooth service or a multitouch screen
  - app에서 사용할 API libraries(Android framework에서 제공하는 API 제외)
    ```gradle
    <meta-data
        android:name= "com.google.android.geo.API_KEY"
        android:value= "API KEY"/>
    ```
#### [Declaring components]
- manifest file에서 무엇보다 중요한 것은 app component 정의다.
```gradle
<?xml version="1.0" encoding="utf-8"?>
<manifest ... >
    <application android:icon="@drawable/app_icon.png" ... >
        <activity android:name="com.example.project.ExampleActivity"
                  android:label="@string/example_label" ... >
        </activity>
        ...
    </application>
</manifest>
```
- < application > tag : application 전체적인 설정을 하고 내부적으로 app component를 정의
- < activity >, < service >, < receiver >, < provider >는 각각의 app component를 가리킨다.
- activity, service, content provider는 manifest에 정의하지 않으면 system이 알지 못한다.
  - 하지만 broadcast receiver는 code 상에서 동적으로 등록해도 가능하다.
#### [Declaring component capabilities]
- Intent에는 explicit과 implicit 두가지가 있다고 했는데 explicit이 직접 target component를 지정하는 거라면 implicit는 component capability를 intent filter를 통해서 지정할 수 있다.
- system은 intent filter에 지정한 요소들을 비교해서 여러 component 중에서 맞는 component를 선택하게 된다.
  - 주의할 점은 Service는 explicit으로 지정하는 것이 안전하다. 어떤 Service가 실행될지 모르고 사용자는 Service의 실행을 볼 수 없기 때문이다. API level 21 부터는 bindService()를 implicit intent로 하면 exception이 발생한다.
```gradle
<manifest ... >
    ...
    <application ... >
        <activity android:name="com.example.project.ComposeEmailActivity">
            <intent-filter>
                <action android:name="android.intent.action.SEND" />
                <data android:type="*/*" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```
- action은 수행할 일반적인 작업을 나타내며 code에서는 setAction()을 사용해서 지정할 수 있다.
  - Intent.ACTION_VIEW, Intent.ACTION_SEND와 같이 지정 가능
- data는 작업을 수행할 데이터 타입의 MIME 타입과 URI를 지정하며 setType()과 setData()를 사용해서 지정할 수 있다.
  - MIME은 전달되는 콘텐츠의 포맷 식별자(text/plain, image/png와 같은 것), URI는 리소스 식별자
- category는 component에 대한 추가 정보
  - CATEGORY_DEFAULT가 있는데 implicit 사용 시 반드시 추가
#### [Declaring app requirements]
- Android device는 종류가 많고 같은 feature와 capability를 가지지 않는다.
- app을 실행하기에 부족한 feature를 가진 device에 설치되는 것을 막기 위해 device와 hardware, software requirements를 정의한다.
- hardware, software requirements의 경우 manifest file에 정의한다.
```gradle
<manifest ... >
    <uses-feature android:name="android.hardware.camera.any"
                  android:required="true" />
    ...
</manifest>
```
- device requirements의 경우 minSdkVersion, targetSdkVersion을 app module의 build.gradle file에 정의한다.
  - minSdkVersion은 app이 실행되는데 필요한 최소 API level(설정하지 않으면 기본은 1)
  - targetSdkVersion은 app이 타겟팅하는 API level로 이 version 이상에서 지원기능을 사용하기 위함이다.
```gradle
android {
  ...
  defaultConfig {
    ...
    minSdkVersion 26
    targetSdkVersion 29
  }
}
```
- 이런 설정들은 대부분 system이 읽는 것이 아니라 구글 플레이와 같은 external service가 읽어서 user의 device에 맞게 filtering해서 제공하게 된다.
## App resources
- Android app은 code 뿐 아니라 많은 resources로도 구성되어 있다.
- resource를 사용하여 code modify없이 app을 쉽게 update할 수 있도록 한다.
  - anim : view animations 정의한 XML file
  - menu : app menu 정의한 XMl file
  - values : strings.xml, colors.xml, arrays.xml 등
  - color : state에 따른 color를 정의한 XMl file
  - layout : layout 정의한 xml file
  - drawable : bitmap file(png, jpg, gif), shapes 등
  - mipmap : launcher icon(app icon)
  - font : font file
  - raw : 텍스트나 오디오 file
- SDK build tools는 모든 resource에 unique integer ID를 부여한다.
  - 이 ID를 통해서 code나 다른 resource xml file에서 정의하여 사용할 수 있다.
  - drawable에 있는 logo.png라면 R.drawable.logo라는 이름으로 사용할 수 있고 이는 unique interger ID를 나타낸다.
- resource는 다양한 device configrations에 맞게 작성될 수 있다.
  - 화면 전환, 다국어, 해상도 등
  - 이를 위해 사용하는 것이 qualifier
- qualifier는 device configuration에 맞게 사용되어야할 resource를 구분해주는 short string
  - device configuration에 맞는 resource를 사용할 수 있도록 resource directories name에 포함시킨다.
  - 예를 들어 device orientation에 맞는 layout을 지정한다고 하자.
  - portrait, landscape 각각에 대한 2개의 layout file을 정의하고 각각의 layout directory name에 알맞은 qualifier를 적용시킨다.
  - 그러면 시스템은 현재 device orientation에 맞는 layout을 자동으로 적용시키게 된다.
