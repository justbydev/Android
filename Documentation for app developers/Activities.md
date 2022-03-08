# Activities
## 참고자료
[Android Developers: Activities](https://developer.android.com/guide/components/activities/intro-activities)
## Introduction to activities
### [The concept of activities]
- user interaction에 따라 항상 같은 곳에서 app이 시작되는 것이 아니다.
  - 이런 paradigm을 적용하여 design된 것이 Activity class
  - one app이 다른 app을 invoke할 때 activity를 invoke해서 app을 calling한다.
  - 그렇기 때문에 activity는 app과 user간의 interaction의 entry point다.
- activity는 app이 UI를 그리는 window를 제공한다.
  - 이 window는 full screen일 수도 있고 screen보다 작을 수도 있다.
- 대부분의 app은 여러 screen으로 구성되어 있고 이것은 여러 activities로 구성되어 있다는 뜻이다.
  - 일반적으로 app에서 하나의 activity는 main activity로 지정되며 이는 user가 app을 launch할 때 처음 나타나는 screen이다.
  - 각각의 activity는 다른 action을 하기 위해 다른 activity를 시작할 수 있다.
- activities들은 일관된 user experience 형성을 위해 함께 작동하지만 각 activity는 다른 activity와 loosely bound되어 있다.
  - 다른 activities들과 최소의 종속성을 가진다.
  - 실제로 activities들은 종종 다른 app의 activities를 시작한다.
- activities를 상요하려면 manifest에 등록해야 하고 반드시 activity lifecycles를 제대로 관리해야 한다.

### [Configuring the manifest]
#### Declare activities
```Gradle
<manifest ... >
  <application ... >
      <activity android:name=".ExampleActivity" />
      ...
  </application ... >
  ...
</manifest >
```
- 필수적으로 필요한 attribute는 activity class name을 나타내는 android:name이다.

#### Declare intent filters
```Gradle
<activity android:name=".ExampleActivity" android:icon="@drawable/app_icon">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```
- Intent filters를 지정하게 되는데 explicit request뿐 아니라 implicit request로 activity를 launch할 수 있도록 한다.
  - < action >은 수행할 일반적인 작업을 의미하고 < category >, < data >는 optional element다.<sup id="r1">[1)](#f1)</sup>
```kotlin
val sendIntent = Intent().apply {
    action = Intent.ACTION_SEND
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, textMessage)
}
startActivity(sendIntent)
```
- 만약 다른 app이 activities에 접근하지 못하게 하려면 intent filters를 지정하지 않는다.
#### Declare permissions
```Gradle
<manifest>
<activity android:name="...."
   android:permission=”com.google.socialapp.permission.SHARE_POST”

/>
```
```Gradle
<manifest>
   <uses-permission android:name="com.google.socialapp.permission.SHARE_POST" />
</manifest>
```
- 만약 manifest에서 같은 permissions을 갖고 있지 않다면 parent activity는 child activity를 launch할 수 없다.
- 만약 parent activity에서 uses-permission을 선언했다면 각각의 child activity에서도 반드시 matching해야 한다.
- activity에 permission을 지정했다면 app의 manifest에서도 반드시 uses-permission을 지정해야 한다.<sup id="r2">[2)](#f2)</sup>

## The Activity Lifecycle
- Activity class는 activity state가 어떻게 변하는지 알 수 있는 callback methods를 제공한다.
- lifecycle callback methods를 고려하면 다음과 같은 상황을 막을 수 있다.
  - user가 다른 app으로 switch하거나 phone call할 때 보고 있던 app이 crash되는 상황
  - user가 app을 actively하게 사용하지 않을 때 valuable system resource를 소비하는 상황
  - app을 나갔다가 나중에 돌아왔을 때 사용자의 진행 상태가 저장되지 않는 상황
  - rotate될 때 app이 crash되거나 사용자의 진행 상태가 저장되지 않는 상황

### [Activity-lifecycle concepts]
<img width="377" alt="스크린샷 2022-03-08 오전 10 58 27" src="https://user-images.githubusercontent.com/17876424/157150978-0fd65e01-d2fd-4189-8c48-acda3160eb0c.png">

- Activity class는 onCreate(), onStart(), onResume(), onPause(), onStop(), onDestroy()의 core set of 6 callbacks를 제공한다.
- user가 activity를 벗어나기 시작하면 system은 activity를 해체하기 위해 method를 호출한다.
  - 하지만 단지 background에 존재하며 후에 다시 foreground로 가져오기 위해 memory에 남아 있는 경우도 있다.
  - 다시 돌아오는 경우 activity는 user가 벗어났던 부분부터 재개한다.
- system이 activities를 비롯한 process를 kill할 가능성은 activity의 state에 따른다.
### [Lifecycle callbacks]
#### onCreate()
- activity가 처음 create될 때 실행되는 callback으로 반드시 구현해야 한다.
- activity는 Created state가 된다.
- activity의 전체 lifed에서 딱 한번 반드시 발생해야 하는 basic application startup logic을 구성한다.
- Bundle object인 savedInstanceState를 parameter로 받는다.
  - activity의 이전 saved state를 갖고 있으며 activity가 이전에 한번도 존재한 적 없다면 null이다.
- 만약 lifecycle-aware component를 갖고 있다면 ON_CREATE event를 받게 된다.
  - @OnLifecycleEvent(Lifecycle.EVENT.ON_CREATE)
- onCreate()에는 setContentView()가 있으며 XML layout file resource ID를 넘겨 화면을 구성한다.
- XML layout file resource ID 말고 ViewGroup을 넘길 수도 있다.
  - activity code를 통해 새로운 View object를 생성한다.
  - 생성한 Views를 ViewGroup에 넣는다.
  - root인 ViewGroup을 넘겨 새롭게 생성한 Views를 사용하여 화면을 구성하게 된다.
- onCreate()가 끝나면 Created state에서 Started state가 된다.
```kotlin
lateinit var textView: TextView

// some transient state for the activity instance
var gameState: String? = null

override fun onCreate(savedInstanceState: Bundle?) {
    // call the super class onCreate to complete the creation of activity like
    // the view hierarchy
    super.onCreate(savedInstanceState)

    // recovering the instance state
    gameState = savedInstanceState?.getString(GAME_STATE_KEY)

    // set the user interface layout for this activity
    // the layout file is defined in the project res/layout/main_activity.xml file
    setContentView(R.layout.main_activity)

    // initialize member TextView so we can manipulate it later
    textView = findViewById(R.id.text_view)
}

// This callback is called only when there is a saved instance that is previously saved by using
// onSaveInstanceState(). We restore some state in onCreate(), while we can optionally restore
// other state here, possibly usable after onStart() has completed.
// The savedInstanceState Bundle is same as the one used in onCreate().
override fun onRestoreInstanceState(savedInstanceState: Bundle?) {
    textView.text = savedInstanceState?.getString(TEXT_VIEW_KEY)
}

// invoked when the activity may be temporarily destroyed, save the instance state here
override fun onSaveInstanceState(outState: Bundle?) {
    outState?.run {
        putString(GAME_STATE_KEY, gameState)
        putString(TEXT_VIEW_KEY, textView.text.toString())
    }
    // call superclass to save any view hierarchy
    super.onSaveInstanceState(outState)
}
```
#### onStart()
- activity가 Started state가 되었을 때 system은 이 callback을 호출한다.
- onStart()은 activity가 user에게 visible하게 하고 foreground로 가져오게 한다.
- activity가 Started state가 되면 lifecycle-aware component는 ON_START event를 받게 된다.
  - @OnLifecycleEvent(Lifecycle.EVENT.ON_START)
- onStart() method는 굉장히 빨리 끝나며 Resumed state로 넘어가고 onResume()을 호출한다.

#### onResume()
- Resumed state가 되면 foreground로 오고 system은 이 callback을 호출한다.
- 이제 user와 interact하게 된다.
- app은 focus를 잃어버릴 때까지 Resumed state에 머무르게 된다.
- activity가 Resumed state가 되면 lifecycle-aware component는 ON_RESUME event를 받게 된다.
  - @OnLifecycleEvent(Lifecycle.EVENT.ON_RESUME)
  - lifecycle component는 component가 visible하고 foreground에 있을 때 해야하는 기능들을 수행하게 된다.
- interruptive event가 발생하게 되면 activity는 Paused state가 되고 onPause() callback을 호출한다.<sup id="r3">[3)](#f3)</sup>
- Paused state에서 Resumed state로 돌아오면 다시 onResume() callback을 호출한다.
  - 그렇기 때문에 onPause()에서 release한 것을 onResume()에서 다시 initialize 해줘야 한다.
```kotlin
class CameraComponent : LifecycleObserver {

    ...

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun initializeCamera() {
        if (camera == null) {
            getCamera()
        }
    }

    ...
}
```
- lifecycle-aware component가 ON_RESUME event를 받았을 때 처리해주는 코드다.
- 만약 multi-window mode인 경우 visible하지만 다른 window를 tap했기에 Paused 상태일 수 있다.
  - 그렇기에 만약 visible과 더불어 foreground에서 active한 경우에만 camera를 active 하도록 하고 싶다면 ON_RESUME event 후에 initialize해야 한다.
  - 만약 Paused이지만 visible한 상태에서 camera를 active하고 싶다면 ON_START event후에 initialize해야 한다.
    - 하지만 이런 경우 activity가 Paused일 때는 camera가 다른 Resumed 상태인 app에서 access가 거부될 수 있기 때문에 전체적인 user experience를 감소시킬 수 있다.
- 어떤 event에서 initialization operation을 하든 corresponding한 lifecycle event에서 resource를 해제해야 한다.
  - 만약 ON_START event 후에 initialize했다면 ON_STOP event 후에 해제해야 한다.
  - 만약 ON_RESUME event 후에 initialize했다면 ON_PAUSE event 후에 해제해야 한다.

### onPause()
- user가 처음 activity를 벗어난 것을 인식했을 때 system이 호출한다.(벗어난 것이 항상 destroy됐다는 것은 아니다.)
- 물론 multi-window mode에서는 visible하지만 activity가 더 이상 foreground에 위치하지 않는다는 뜻이다.
- Activity가 Paused state에 있을 때 계속 실행해서는 안되지만 잠시 후 다시 시작할 operation을 pause하거나 adjust하게 된다.
  - 

## Q&A
<b id="f1">1) </b>category.DEFAULT, category.LAUNCHER, action.MAIN[↩](#r1)<br>
- implicit intent를 받으려면 반드시 category.DEFAULT를 intent filter category에 포함시켜야 한다.
- action.MAIN은 이 activity가 main entry point이고 다른 intent data를 expect하지 않는다는 뜻이다.
- category.LAUNCHER는 activity's icon이 system's app launcher에 위치해야 한다는 것이다.
  - 이 둘을 반드시 함께 사용해야 activity가 app launcher로써 나타날 수 있다.
  - exported 속성이 있는데 이를 true로 하면 모든 app에서 이 activity에 access할 수 있다.
  - category.LAUNCHER를 포함한 경우 true로 지정하고 다른 activity는 보통 false로 지정한다.
- [Intent filters](https://developer.android.com/guide/components/intents-filters)
- [exported](https://developer.android.com/guide/topics/manifest/activity-element#exported)


<b id="f2">2) </b> 각각의 activity에 permission하면 uses-permission에도 똑같이 지정해야 한다고 하는데 그러면 굳이 activity마다 permission을 지정할 필요가 있을까?[↩](#r2)<br>
<b id="f3">3) </b>Paused state라고 되어 있는데 Lifecycle 에서 state 중에 Paused는 없었는데 그냥 onPause()이기에 Paused state라고 한 것일까?[↩](#r3)<br>








