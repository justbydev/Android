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
- activity의 전체 life에서 딱 한번 반드시 발생해야 하는 basic application startup logic을 구성한다.
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

#### onPause()
- user가 처음 activity를 벗어난 것을 인식했을 때 system이 호출한다.(벗어난 것이 항상 destroy됐다는 것은 아니다.)
- 물론 multi-window mode에서는 visible하지만 activity가 더 이상 foreground에 위치하지 않는다는 뜻이다.
- Activity가 Paused state에 있을 때 계속 실행해서는 안되지만 잠시 후 다시 시작할 operation을 pause하거나 adjust하게 된다.
  - app execution을 interrupt하는 event
  - multi-window mode인 경우 app 중 하나만 focus를 얻기 때문에 다른 app들은 pause한다.
  - dialog와 같이 new, semi-tranparent activity가 생성됐을 경우 기존 activity가 visible하지만 focus를 잃었기 때문에 pause한다.
- activity가 paused state가 되면 lifecycle-aware component는 ON_PAUSE event를 받게 된다.
  - @OnLifecycleEvent(Lifecycle.EVENT.ON_PAUSE)
  - component가 더 이상 foreground에 있지 않을 때 lifecycle compoenent가 기능을 멈추기 위해 사용하게 된다.
- system resource를 해제할 때도 사용한다.
  - 하지만 multi-window mode인 경우 visible하기 때문에 완전히 해제거나 UI-related resource, operation을 조정하기 위해 onStop()를 고려해야 한다.
```kotlin
class CameraComponent : LifecycleObserver {

    ...

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun releaseCamera() {
        camera?.release()
        camera = null
    }

    ...
}
```
- ON_RESUME event와 반대 상황이다.
- onPause()는 굉장히 짧기 때문에 application or user data를 저장하거나 network 사용, database transactions를 실행해서는 안된다.
  - 이와 같은 작업 수행 시 작업을 다 마치기 전에 onPause()가 끝날 수 있다.
  - 이와 같은 heavy-load shutdown operations의 경우 onStop()에서 해야 한다.
- onPause() 후에 다시 activity가 재개되어 onResume() callback이 호출되거나 완전하게 invisible되어 onStop()이 호출된다.

#### OnStop()
- activity가 user에게 invisible할 때 Stopped state로 들어가고 system은 onStop() callback을 호출한다.
- activity가 stopped state가 되면 lifecycle-aware component는 ON_STOP event를 받게 된다.
  - @OnLifecycleEvent(Lifecycle.EVENT.ON_STOP)
  - component가 더 이상 screen에서 visible하지 않을 때 lifecycle component가 기능을 멈추기 위해 사용하게 된다.
- app이 user에게 보이지 않을 때 더 이상 필요하지 않은 resource를 release하거나 조정한다.
- relatively CPU-intensive shutdown operations의 경우도 이 곳에서 실행한다.
- Stopped state여도 memory에 위치하기 때문에 state, member information을 유지한다.
  - 단, window manger에는 detach한다.
  - 다시 activity가 재개되면 기존 information을 불러온다.
  - 또한, View object의 현재 상태도 system이 유지하기 때문에 저장하고 복구할 필요가 없다.
  - 만약 activity가 stop일 때 system이 process를 destroy해도 View object 상태는 system이 Bundle에 저장하고 다시 activity로 돌아오면 restore한다.

#### onDestroy()
- activity가 destroy될 때 호출된다.
  - user가 완전히 끝내거나 finish()를 호출한 경우
  - configuration change에 의해 system이 잠깐 activity가 destroy한 경우
- activity가 destroyed state가 되면 lifecycle-aware component는 ON_DESTROY event를 받게 된다.
  - @OnLifecycleEvent(Lifecycle.EVENT.ON_DESTROY)
- ViewModel를 사용하여 data를 저장하면 configuration change가 발생해도 유지된다.
  - 만약 activity가 완전히 끝난 경우에는 onCleared() method를 통해 ViweModel data도 clean up된다.
  - configuration change에 의한 destroy 상황인지 finish()에 의한 destroy 상황인지는 isFinishing() method로 구분할 수 있다.
    - finish()에 의한 destroy인 경우 isFinishing()은 true를 return한다.
- onStop()에서 아직 release하지 않은 모든 resouce를 release한다.

### [Activity state and ejection from memory]
<img width="763" alt="스크린샷 2022-03-08 오후 1 36 30" src="https://user-images.githubusercontent.com/17876424/157167053-e85ae763-f9cb-462e-a7bc-ca97647e9183.png">

- system은 memory를 위해 직접 activity를 kill하지는 않는다.
  - activity가 run되는 process를 kill하며 activity 뿐 아니라 그 process에서 running되는 모든 것들을 kill한다.

### [Saving and restoring transient UI state]
- configuration change, multi-window mode, 잠깐 다른 app을 갔다가 다시 돌아오는 경우 등에서는 user는 activity UI state가 그대로 유지될 것이라 예상한다.
  - 그렇기에 Viewmodel, onSaveInstanceState(), local storage를 사용하여 이런 상황에서 UI state를 유지해준다.
#### Instance state
- finish() 하거나 Back 한 경우
  - 이런 상황에서는 user나 system이나 둘다 activity가 완전히 destroy될 것으로 예상하기 때문에 다른 작업을 할 필요가 없다.
- configuration change or memory pressure 등 system에 의해 destroy되는 경우
  - 다시 activity로 돌아오는 경우 system은 new activity instance를 생성하면서 이전 상태가 유지되어 있어야 한다.
  - 이런 상황에서 system이 복구하는 이전 상태를 instance state라고 하며 key-value Bundle object를 사용한다.
- Bundle object는 key-value pairs collection
  - system은 기본적으로 Bundle instance state를 사용해서 View object 상태를 저장한다.
    - 부가적인 코드 없이 View object의 previous state가 복구된다.
    - 단, View는 unique ID를 갖고 있어야 한다.
- Bundle object의 경우 main thread에서 serialization을 필요로 하며 system-process memory를 소비하기 때문에 더 많은 data를 유지하기 위해서는 persisten local storage, onSaveInstanceState() method, ViewModel을 combine해서 사용해야 한다.

#### Save simple, lightweight UI state using onSaveInstanceState()
```kotlin
override fun onSaveInstanceState(outState: Bundle?) {
    // Save the user's current game state
    outState?.run {
        putInt(STATE_SCORE, currentScore)
        putInt(STATE_LEVEL, currentLevel)
    }

    // Always call the superclass so it can save the view hierarchy state
    super.onSaveInstanceState(outState)
}

companion object {
    val STATE_SCORE = "playerScore"
    val STATE_LEVEL = "playerLevel"
}
```
- activity가 멈추기 시작하면 onSaveInstanceState()<sup id="r4">[4)](#f4)</sup>가 호출되어 instance state bundle에 state information을 저장한다.
  - 기본적으로 activity's view hierarchy state를 저장한다.
- 예상치 못하게 activity가 destroy된 경우 onSaveInstanceState()를 override하여 필요한 데이터를 Bundle object를 사용하여 저장한다.
  - override 할 때 view hierarchy state 저장을 위해 superclass implementation을 call해야 한다.
- user가 finish()를 통해 명시적으로 activity를 끝낸 경우 호출되지 않는다.

#### Restore activity UI state using saved instance state
- Bundle 통해 저장한 saved instance state는 onCreate()와 onRestoreInstanceState()에서 받게 된다.
  - 둘은 같은 Bundle을 받게 된다.
- onCreate()가 new instance, recreate 두 상황에서 모두 호출되어 Bundle을 받기 때문에 null 체크를 해줘야 한다.
- onRestoreInstanceState()를 사용할 수도 있다.
  - onStart() 다음에 system이 호출한다.<sup id="r5">[5)](#f5)</sup>
  - 복구할 saved state가 있는 경우에만 호출하기 때문에 Bundle이 null인지 체크할 필요가 없다.
```kotlin
override fun onRestoreInstanceState(savedInstanceState: Bundle?) {
    // Always call the superclass so it can restore the view hierarchy
    super.onRestoreInstanceState(savedInstanceState)

    // Restore state members from saved instance
    savedInstanceState?.run {
        currentScore = getInt(STATE_SCORE)
        currentLevel = getInt(STATE_LEVEL)
    }
}
```
### [Navigating between activities]
#### Starting one activity from another
- 하나의 activity에서 다른 activity로의 이동이 필요한 경우가 있다.
- startActivity() or startActivityForResult() 사용
  - 둘다 Intent object를 전달한다.
  - Intent object에는 정확히 어떤 activity를 시작할 것인지 명시하거나 어떤 action을 실행하고 싶은지 나타낼 수 있다.
    - type of action을 표시한 경우 system이 그것에 맞는 activity를 선택한다.
  - Intent object를 통해 시작하게 될 activity에 소량의 데이터도 전달할 수 있다.
#### startActivity()
- result를 return할 필요 없는 activity를 시작할 때 사용한다.
```kotlin
val intent = Intent(this, SignInActivity::class.java)
startActivity(intent)
```
- 위와 같이 알고 있는 activity를 launch할 수 있다.'
```kotlin
val intent = Intent(Intent.ACTION_SEND).apply {
    putExtra(Intent.EXTRA_EMAIL, recipientArray)
}
startActivity(intent)
```
- app이 email sending, status update와 같은 다른 action을 필요로 할 수도 있다.
  - 이런 경우 이런 action을 수행할 activities를 직접 가지기 보다는 다른 app에서 제공하는 activities를 사용할 수 있다.
  - 이때, Intent를 사용하여 수행하고자 하는 action을 정의하고 system이 action에 맞는 다른 app의 activity를 launch한다.
  - 만약 해당하는 activity가 여러개라면 user가 선택할 수 있다.<sup id="r6">[6)](#f6)</sup>

#### startActivityForResult()
```kotlin
class MyActivity : Activity() {
    // ...

    override fun onKeyDown(keyCode: Int, event: KeyEvent?): Boolean {
        if (keyCode == KeyEvent.KEYCODE_DPAD_CENTER) {
            // When the user center presses, let them pick a contact.
            startActivityForResult(
                    Intent(Intent.ACTION_PICK,Uri.parse("content://contacts")),
                    PICK_CONTACT_REQUEST)
            return true
        }
        return false
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, intent: Intent?) {
        when (requestCode) {
            PICK_CONTACT_REQUEST ->
                if (resultCode == RESULT_OK) {
                    startActivity(Intent(Intent.ACTION_VIEW, intent?.data))
                }
        }
    }

    companion object {
        internal val PICK_CONTACT_REQUEST = 0
    }
}
```
- 다른 activity가 끝날 때 결과를 받고 싶은 경우 startActivityForResult(Intent, int)를 사용할 수 있다.<sup id="r7">[7)](#f7)</sup>
  - Integer의 경우 call를 구별해주는 request code
  - result는 onActivityResult(int, int, Intent)를 통해 받게 된다.
- child activity는 exit할 때 setResult(int)를 호출하게 된다.
  - Integer는 result code로 RESULT_CANCELED(-1), RESULT_OK(0), or RESULT_FIRST_USER(1)로 시작하는 custom value를 사용할 수 있다.
  - 또한 Intent object에 additional data를 담아서 return할 수 있다.
    -  setResult(int, Intent) 사용
- 만약 crash와 같은 이유로 child activity가 fail되면 RESULT_CANCELED를 parent activity에게 보낸다.
#### Coordinating activities
- one activity가 다른 activity를 시작할 때 둘은 각자의 lifecycle를 갖게 된다.
- 이런 상황에서 다음과 같이 lifecycle callback이 order를 가진다.(A가 B를 시작한 경우)
1) Activity A의 onPause()가 실행된다.
2) Activity B의 onCreate(), onStart(), onResume()가 순서대로 실행되고 이제 Activity B가 user focus를 가진다.
3) Activity A가 더 이상 visible하지 않으면 onStop()이 실행된다. 

## Handle Activity State Changes
### [Configuration change occurs]
- configuration change(rotation, language changes, etc)가 발생했을 때 activity는 destroy된 후 recreate된다.
- onPause(), onStart(), onDestroy() 후에 새로운 activity instance가 생성되면서 onCreate(), onStart(), onResume()이 호출된다.
#### Handle multi-window cases
- API level 24 이상부터 가능한 multi-window mode에 진입하면 system은 configuration change되었다고 알리고 위와 같이 lifecycle이 변한다.
- multi-window mode가 resize 되어도 같은 결과가 나타난다.
- 2개의 app이 동시에 visible 해도 오직 하나만 focus를 얻고 foreground에서 user와 interact한다.
  - focus를 얻은 activity는 Resumed state가 되고 다른 app은 Paused state가 된다.
- 만약 focus하는 app을 바꾸게 된다면(app A to app B)
  - app A에서는 onPause()가 호출되고 app B에서는 onResume()가 호출된다.

### [Activity or dialog appears in foreground]
- 만약 새로운 activity나 dialog가 foreground로 와서 focus를 가지고 진행 중인 activity를 부분적으로 가리면 가려진 activity는 focus를 잃고 Paused state가 되면서 onPause()를 호출한다.
  - 가려진 activity가 다시 foreground로 오고 focus를 얻으면 onResume()이 호출된다.
- 만약 새로운 activity나 dialog가 foreground로 와서 focus를 가지고 진행 중인 activity를 완전히 가리면 가려진 activity는 focus를 잃고 Stopped state가 되면서 system은 빠르게 onPause(), onStop()을 호출한다.
  - 가려진 activity가 다시 foreground로 오고 focus를 얻으면 onRestart(), onStart(), onResume()이 호출된다.
  - 만약 다시 foreground로 온 activity가 new instance라면 onRestart() 없이 onStart(), onResume()이 호출된다.
  - 홈이나 최근 목록 버튼을 눌렀을 때도 system은 activity가 완전히 가려진 것처럼 처리한다.

### [User presses or gestures Back]
- 만약 Back을 하게 되면 onPause(), onStop(), onDestroy() 되고 destroy되면 back stack에서 사라진다.
  - 만약 그 activity가 root launcher activity라면 Android version에 따라 다르게 처리한다.(Tasks and back stack에서 정리)
- 이 상황에서는 onSaveInstanceState()가 호출되지 않는다.
  - user가 다시 이 activity로 오지 않는다고 expect하기 때문이다.
- onBackPressed()를 override할 수 있으며 이때 super.onBackPressed()를 호출하기를 권장한다.

### [System kills app process]
- 만약 app이 background에 있고 system이 foreground app을 위해 additional memory를 필요로 한다면 background app을 kill할 수 있다.

## Test your activities
- activity는 모든 user interaction을 처리하고 lifecycle event에 따라 제대로 된 반응을 하는 것이 중요하기 때문에 device-level events 동안 app's activities를 test하는 것이 중요하다.
### [Drive an activity'sstate]
- activities test에 있어 중요한 측면은 activities를 특정 state에 placing하는 것이다.
- 여기서는 AndroidX Test library의 ActivityScenario를 사용한다.
  - ActivityScenario는 local unit test와 on-device integration test 모두에서 사용할 수 있는 cross-platform API다.
  - thread safety
  - test의 instrumentation thread와 테스트 중인 activity를 실행하는 thread 간에 event를 동기화한다.
#### Create an activity
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEvent() {
       launchActivity<MyActivity>().use {
       }
    }
}
```
- test 중인 activity는 다음과 같이 create한다.
- create 후 ActivityScenario는 activity를 RESUMED state로 만든다.
- 이와 같은 state에서는 activity의 View elements와 자유롭게 interact 할 수 있다.
- test가 끝나면 close하는 것이 좋다.
  - use extension을 사용하여 activity가 자동으로 close되도록 한다.
- 다른 방법으로 ActivityScenarioRule를 사용하여 each test 전에 자동으로 ActivityScenario.launch하도록 할 수도 있다.
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @get:Rule var activityScenarioRule = activityScenarioRule<MyActivity>()

    @Test fun testEvent() {
        val scenario = activityScenarioRule.scenario
    }
}
```
#### Drive the activity to a new state
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEvent() {
        launchActivity<MyActivity>().use { scenario ->
            scenario.moveToState(State.CREATED)
        }
    }
}
```
- moveToState()를 통해서 CREATED, STARTED 등 다른 state으로 이동할 수 있다.
  - app이나 system에 의해 interrupt될 수 있기 때문에 각각 stopped, paused인 상황을 simulate한다.

#### Determine the current activity state
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEvent() {
        launchActivity<MyActivity>().use { scenario ->
            scenario.onActivity { activity ->
              startActivity(Intent(activity, MyOtherActivity::class.java))
            }

            val originalActivityState = scenario.state
        }
    }
}
```
- test하고 있는 activity의 현재 상태를 결정하고 싶다면 ActivityScenario object의 state field 값을 받아온다.

#### Recreate the activity
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEvent() {
        launchActivity<MyActivity>().use { scenario ->
            scenario.recreate()
        }
    }
}
```
- system이 resource가 모자라 activity를 destroy하는 상황을 simulate하고 싶다면 recreate()를 사용한다.
- ActivityScenario class는 @NonConfigurationInstance를 사용하여 activity의 saved instance state와 object는 유지한다.

#### Retrieve activity results
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testResult() {
        launchActivity<MyActivity>().use {
            onView(withId(R.id.finish_button)).perform(click())

            // Activity under test is now finished.

            val resultCode = scenario.result.resultCode
            val resultData = scenario.result.resultData
        }
    }
}
```
- finished activity로부터 result code나 data를 받고 싶다면 ActivityScenario object의 result field를 사용한다.
#### Trigger actions in the activity
- ActivityScenario안에서의 모든 method는 blocking calls이기 때문에 instrumentation thread에서 실행해야 한다.
- activity의 action을 trigger하려면 Espresso view matchers를 사용한다.
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEvent() {
        launchActivity<MyActivity>().use {
            onView(withId(R.id.refresh)).perform(click())
        }
    }
}
```
- activity 자체에서 method를 호출해야 한다면 다음과 같이 ActivityAction을 구현한다.
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEvent() {
        launchActivity<MyActivity>().use { scenario ->
            scenario.onActivity { activity ->
              activity.handleSwipeToRefresh()
            }
        }
    }
}
```


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
<b id="f4">4) </b> onSaveInstanceState() 호출되는 경우와 호출되지 않는 경우는? [↩](#r4)<br>
- activity가 finish되어 완전히 destroy되어 user가 다시 돌아온다고 여겨지지 않는 경우 호출되지 않는다.
  - finish()에 의한 종료나 back button에 의한 수동 종료인 경우 호출되지 않는다.
  - system에 의해 configuration change, memory pressure에 의해 activity가 destroy된 경우 호출된다.
- 만약 finish될 때 데이터를 저장하기 위해서는 persistent data로 저장해야 하고 이는 onStop()에서 행한다.


<b id="f5">5) </b> onRestoreInstanceState()가 onStart() 다음에 호출된다면 onSaveInstanceState()는 어느 시점에 호출될까?[↩](#r5)<br>
<b id="f6">6) </b> Intent action에 해당하는 activity 중에서 보여주는/선택하는 과정은?[↩](#r6)<br>
<b id="f7">7) </b> startActivityForResult()가 deprecated 됐다는데 이유와 대체 방법은?[↩](#r7)<br>




