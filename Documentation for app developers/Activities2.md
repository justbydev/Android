# Activities
## Tasks and the back stack
- task는 user가 interact하는 activities의 collection
- 이렇게 모인 activities는 back stack에 arrange되어 관리된다.
  - 새로운 activity는 back stack에 추가되고 finish되면 stack으로부터 pop된다.

### [Lifecycle of a task and its back stack]
- user가 app launcher의 icon이나 홈 화면의 바로가기를 클릭하면 app의 task가 foreground로 나오게 된다.
  - 최근에 app을 사용한 적이 없어 task가 없다면 task를 생성하고 main activity가 stack의 root activity로써 app이 실행된다.
- new activity는 stack의 top에 push되고 이미 있던 activity는 stopped된 채로 stack에 남아 있는다.
- back action을 취하면 activity가 destroy되면서 stack에서 pop되고 이전 activity가 resume된다.
- stack의 activities는 절대 재정렬되지 않고 오직 push, pop만 된다.
- 계속 back action을 취하면 홈 스크린이 나올 때까지 pop 되고 stack에서 모든 activity가 remove되면 task는 남아있지 않다.

#### Back press behavior for root launcher activities
- Intent filter에서 ACTION_MAIN, CATEGORY_LAUNCHER로 정의한 activity가 Root launcher activity다.
  - app launcher에서 app을 시작할 때 entry point가 되는 unique한 activity로 task를 시작할 때 사용된다.
- root launcher activity에서 back action을 취하면 Android version에 따라 다르게 event가 발생한다.
  - Android 11 이하인 경우
    - system이 activity를 finish한다.
  - Android 12 이상인 경우
    - system은 activity를 finish하기 보다는 activity와 그것의 task를 background로 옮긴다.
    - 이는 user가 warm state에서 app을 빠르게 resume할 수 있도록 한다.
    - 만약 custom back navigation를 필요로 한다면 onBackPressed() override 보다는 AndroidX Activity APIs<sup id="r1">[1)](#f1)</sup>를 추천한다.
      - system Back press를 intercepting하는 component가 없다면 맞는 system behavior에게 연기한다.
    - 만약 onBackPressed()를 override해서 activity를 finish한다면 finishing하기보다 super.onBackPressed()에 호출하도록 구현을 update해야 한다.
      - super.onBackPressed()는 activity, task를 background로 이동시킨다.<sup id="r2">[2)](#f2)</sup>

#### Background and foreground tasks
- task는 user가 새로운 task를 시작하거나 홈 스크린으로 갈 때 background로 가는 cohesive unit이다.
- background에 있을 떄 task의 모든 activity는 stop 되지만 back stack은 온전하게 남아 있다.
- 만약 task가 A, B 2개가 있다고 하자.
  - Task A가 foreground에 있었는데 홈 버튼을 눌러서 Task A가 background로 가고 새로운 app을 시작하면 그에 해당하는 Task B가 activities stack과 함께 시작한다.
  - 다시 홈 버튼을 누르고 원래 보던 app을 시작하면 Task A가 foreground로 올라오고 stack top에 있는 activity가 resume된다.

#### Multiple activity instances
- back stack은 절대 재정렬되지 않기 때문에 같은 activity가 여러 개 back stack에 존재할 수 있다.
  - back action으로 pop하면 순서대로 navigate하여 같은 activity가 드러나게 된다.

#### multi-window environments
- multi-windowed 환경에서 여러 app을 동시에 실행하게 되면 system은 각각의 window에서 task들을 분리해서 관리하게 되고 각 window는 여러 개의 task를 가질 수 있다.

### [Manage tasks]
- 대부분의 app에서 task나 back stack을 잘 관리해주지만 원하는 다른 동작이 있을 수 있다.
- task나 back stack 관련 원하는 동작을 하기 위해 manifest file < activity >의 attribute를 사용하거나 startActivity에서 flag와 함께 Intent를 전달할 수 있다.
#### Defining launch modes
- launch mode는 새로운 activity instance가 current task과 어떻게 연결될지 정의하며 manifest file과 Intent flag를 사용할 수 있다.
#### Define launch modes using the manifest file
- < activity >의 launchMode attribute를 사용한다.
- "standard"(the default mode)
- "singleTop"
  - 만약 current task에 같은 activity instance가 top에 있다면 새로운 activity instance를 생성하지 않고 onNewIntent() method를 통해 top에 있는 activity instance를 사용한다.
  - onCreate()가 불리지 않고 onNewIntent()가 불린다.
  - A-B-C-D인 상황에서 singleTop이고 D가 불린다면 onNewIntent()를 통해 Intent를 받으면서 그대로 A-B-C-D가 된다.
    - 하지만 B를 호출하면 새로운 B instance가 stack에 쌓인다.
    - 즉, 오직 top에 존재하는 instance에 대해서만 중복을 피한다.
    - back action을 하게 되면 기존에 있던 D의 상태로 돌아가지 않는다.
- "singleTask"<sup id="r3">[3)](#f3)</sup>
  - singleTask로 지정한 activity는 새로운 task의 root로만 존재할 수 있다.<sup id="r4">[4)](#f4)</sup>
  - 하지만 만약 이미 어떤 task에 activity instance가 있다면 onNewIntent()를 통해 그 task의 activity instance를 사용한다.
  - activity가 새로운 task에서 시작하지만 back action을 하면 이전 activity로 돌아간다.
- "singleInstance"
  - singleTask와 다 같지만 singleInstance로 지정한 activity를 가진 task에는 다른 activity를 launch하지 않는다.
  - 즉, 그 task의 member는 오직 해당 activity 하나다.
- 만약 singleTask를 사용한 activity instance가 있는 task가 background로 갔다가 foreground로 오게 되면 그 task에 있는 모든 activity가 foreground로 오게 되고 back stack은 모든 activity를 갖고 있게 된다.

#### Define launch modes using Intent flags
- startActivity와 함께 Intent flag를 통해 설정한다.
- FLAG_ACTIVITY_NEW_TASK
  - launchMode의 "singleTask"와 같다.
  - 이미 task에 activity가 있다면 onNewIntent()와 함께 그 activity를 실행한다.
- FLAG_ACTIVITY_SINGLE_TOP
  - launchMode의 "singleTop"과 같다.
  - back stack의 top에 같은 activity가 있다면 onNewIntent()와 함께 그 activity를 실행한다.
- FLAG_ACTIVITY_CLEAR_TOP
  - launchMode에 존재하지 않는다.
  - 실행하고자 하는 activity가 current task에 있다면 이미 존재하고 있는 activity 위의 다른 activities는 전부 destroy 시키고 onNewIntent()와 함께 그 activity를 실행한다.

#### Handle affinities
- affinity는 activity가 어떤 task에 포함될 지를 나타낸다.
- 기본적으로 같은 app의 모든 activity는 같은 affinity를 가지기 때문에 같은 task에 포함된다.
- 하지만 activity의 affinity는 바꿀 수 있으며 다른 app의 activity가 같은 affinity를 가질 수 있고 같은 app의 다른 activity가 다른 affinity를 가질 수도 있다.
- manifest의 < activity >의 taskAffinity attribute를 통해 바꿀 수 있다.
  - 지정하지 않으면 기본적으로 package name을 기본 affinity로 사용한다.
  - system은 app의 default task affinity를 확인하는데 사용한다.
- FLAT_ACTIVITY_NEW_TASK flag와 함께 사용하는 경우
  - startActivity()를 하면 새로운 task로 시작한다.
  - 하지만 같은 affinity를 가진 task에 이미 activity가 존재한다면 activity는 그 task에서 launch된다.
  - 만약 이 flag를 사용해서 new task를 만든 뒤 홈 버튼을 눌렀다면 반드시 그 task로 다시 돌아갈 방법이 있어야 한다.
  - 또한, 어떤 entity는 외부 task에서 activities를 시작해서 항상 FLAG_ACTIVITY_NEW_TASK와 함께 사용한다.
    - 이런 상황에서도 프로그램 icon 사용과 같이 다시 task로 돌아갈 수 있는 독립적인 방법이 있는지 살펴봐야 한다.
- allowTaskReparenting attribute를 true로 한 경우
  - 그 activity를 시작한 task에 있다가 같은 affinity를 가진 task가 foreground로 왔을 때 그 task로 옮겨지게 된다.

#### Clear the back stack
- 만약 user가 task를 오랫동안 leave하면 system은 root activity를 제외한 모든 activities를 clear한다.
- 그럼 user가 다시 task를 시작했을 때 root activity만 복구된다.
- 다음과 같은 activity attribute 설정으로 이런 동작을 바꿀 수 있다.
- alwaysRetainTaskState
  - task의 root activity에서 true로 하면 위에서 말한 동작이 일어나지 않고 오랜 시간이 지나도 모든 activities가 task에 남아있는다.
- clearTaskOnLaunch
  - task의 root activity에서 true로 하면 user가 task를 떠났다가 돌아올 때마다 root activity만 남기고 전부 clear된다.
  - alwaysRetainTaskState와 반대이며 항상 user는 task를 initial state로 받는다.
- finishOnTaskLaunch
  - clearTaskOnLaunch와 같지만 오직 하나의 activity에 대해서만 작동한다.
  - root activity를 제외하고 이 attribute를 true로 하면 user가 task를 나가면 이 activity는 존재하지 않게 된다.

#### Start a task
- intent filter에서 android.intent.action.MAIN, android.intent.category.lAUNCHER를 지정하여 entry point로 지정한다.
  - 이런 intent filter는 app launcher에 그 activity가 icon과 label로 나타나게 한다.
  - 또한, 언제나 activity를 launch하여 task로 돌아갈 수 있게 한다.
    - 이는 user가 activity launcher를 사용하여 task를 떠나고 나중에 다시 돌아올 수 있게 한다.
    - 만약 singleTask, singleInstance를 사용하여 new task를 만들고 홈 버튼으로 나가게 되었다면 task는 background에 있고 invisible하게 되는데 ACTION_MAIN, CATEGORY_LAUNCHER를 지정하지 않았으면 task로 돌아갈 수 없게 된다.


## Process and Application Lifecycle
- 대부분의 Android application은 각자의 Linux process에서 작동한다.
- Android의 특이한 특징은 application이 직접 스스로 application process의 lifetime을 control하지 않는다는 것이다.
  - system이 실행 중인 것으로 판단되는 application 요소, 이 요소가 user에게 중요한 정도, overall memory를 조합하여 system에 의해 control된다.
  - application developers는 application components들이 application's process lifetime에 어떻게 영향을 주는지 이해해야 한다.
  - 그렇지 않으면 중요한 작업 도중 application's process가 killing될 수 있다.
- memory가 부족할 때 어떤 process를 kill할 지는 Android가 각각의 process를 "important hierarchy"를 기준으로 정하게 된다.
- foreground process
  - user가 현재 사용하고 있는 process
    - onResume()가 call되어 screen top에서 running 되어 user interaction이 일어나고 있는 Activity
    - BroadcastReceiver.onReceive()가 실행되고 있는 BroadcastReceiver
    - Service.onCreate(), Service.onStart(), Service.onDestroy() 중 하나가 실행되고 있는 Service
  - 이런 process는 몇 개 없으며 정말 memory가 부족한 경우에만 kill된다.
- visible process
  - 작업이 이루어지고 있는 것을 user가 알고 있는 process로 kill하게 되면 user experience에 부정적인 영향을 주게 되는 process
    - onPause()가 call되어 foreground에 있지는 않지만 visible한 Activity
    - Service.startForeground()를 통해 실행된 foreground service
    - 라이브 배경화면, 입력 방법 같이 user가 인지하는 특정 기능에 대해 system이 사용하고 있는 서비스
  - foreground process보다는 낮지만 그래도 여전히 controll된다.
  - 모든 foreground process가 계속 running되도록 하는 것을 필요로 하지 않는 이상 kill되지 않는다.
- service process
  - startService()로 시작한 Service
  - user에게 직접적으로 visible하지는 않지만 user가 신경쓰고 있는 작업을 수행한다.
  - 30분 이상 지속된 Service의 경우 cached list로 중요도를 낮추어 excessive resource를 사용하지 않도록 막는다.
- cached process
  - 현재 필요로 하지 않으며 원하면 언제나 system이 kill할 수 있는 process
  - well running system은 여러 개의 cached process를 갖고 있고 보통 가장 오래된 것을 kill한다.
  - onStop()가 호출되어 user에게 더 이상 visible하지 않는 Activity도 포함된다.
    - kill된 후 다시 복구되면 user experience에 영향을 주지 않도록 saved state를 사용하여 복구한다.
  - 이 process들은 list로 관리되며 정확한 순서는 구현 세부 사항이지만 다른 process보다 유용한 process를 유지하려고 한다.


## Parcelables and Bundles
### [Sending data between activities]
```kotlin
val intent = Intent(this, MyActivity::class.java).apply {
    putExtra("media_id", "a1b2c3")
    // ...
}
startActivity(intent)
```
- 새로운 Activity 시작을 위해 startActivity(Intent) 사용을 위해 Intent object를 생성할 때 putExtra를 통해 parameter를 전달할 수 있다.
- OS는 intent의 Bundle을 parcel하고 새로운 activity를 시작하면 data를 un-parcel해서 새로운 activity한테 전달한다.
- Intent object를 보낼 때 Bundle class 사용을 추천한다.
  - Bundle class는 parcel을 사용하여 최적화된 marshalling, unmarshalling<sup id="r5">[5)](#f5)</sup>을 제공한다.
- 어떤 object의 경우 복잡해서 직접 Parceable을 implemen해야 한다.
  - writeToParcel(android.os.Parcel, int) method를 사용하여 class field를 Parcel에 담는다.
  - Parceable.Creator interface를 implement한 non-null field CREATOR를 제공해야 하며 createFromParcel() method를 통해 Parcel를 다시 원래 object로 변환한다.
- intent를 통해 data 전송 시 few KB limit이 있어서 너무 많은 data를 전송하면 TransactionTooLargeException<sup id="r6">[6)](#f6)</sup>이 발생하게 된다.

### [Sending data between processes]
- process끼리 전송 시 custom Parceable object를 사용하지 않는 것을 권장한다.
  - 서로 주고 받는 app끼리 정확히 같은 version의 custom class를 주고받아야 한다.
  - app이 system에게 custom parceable를 전송하려고 하면 system은 knowledge가 없는 class를 unmarshall하지 못해서 error가 발생한다.
- Parcel object는 Binder transaction buffer를 사용하며 Binder transaction buffer는 1MB의 고정된 size를 가진다.
  - 이 limit은 activity level이 아닌 process level이다.
  - 이런 transaction은 onSaveInstanceState, startActivity, any interaction with the system 등을 전부 포함한다.
  - size limit을 넘게 되면 TransactionTooLargeException이 발생한다.
  - 또한, savedInstanceState의 경우 데이터를 user가 다시 해당 activity로 돌아오기 전까지 hold하고 있어야 하기 때문에 50k 이하로 유지하기를 권장한다.

## Recents screen
- Recents screen(Overview screen, recent task list, recent apps)는 system level UI로 최근에 access한 activities, tasks의 list다.
- Recents screen은 document-centric model로 다른 document를 가진 같은 activity의 여러 instance도 Recents screen에 보여준다.
- 보통 Recents screen에 activities, tasks가 어떻게 보이는지는 system이 정하지만 app이 어떻게 보일지 정할 수 있다.
  - ActivityManager.AppTask class는 task를 관리할 수 있게 하고 activity Intent flag나 manifest의 < activity > attribute를 통해 관리할 수 있다.

### [Add tasks to the Recents screen]
#### Use the Intent flag to add a task
- activity의 새로운 document를 만들 때 startActivity()를 call하고 Intent를 pass한다.
  - Intent flag를 사용하게 된다.
- FLAG_ACTIVITY_NEW_DOCUMENT를 사용하면 activity를 새로운 task로 treat하게 만들 수 있다.
  - Recents screen에 activity가 new task로 나타난다.
  - 반드시 launchMode는 "stardard"여야 한다.
- FLAG_ACTIVITY_MULTIPLE_TASK를 사용하면 항상 new document를 create한다.<sup id="r7">[7)](#f7)</sup>
  - target activity를 root로 항상 새로운 task를 생성한다.
- main activity가 새로운 activity를 launch할 때 existing task를 탐색한다.
  - intent component name, intent data가 일치하는 task가 있는지 탐색한다.
  - 만약 그런 task가 없거나 FLAG_ACTIVITY_MULTIPLE_TASK를 사용했다면 그 activity를 root로 한 새로운 task가 생성된다.
  - 만약 있다면 onNewIntent()를 통해 존재하는 task가 front로 온다.

#### Use the activity attribute to add a task
- manifest에서는 < activity > attribute의 android:documentLaunchMode를 설정한다.
- intoExisting<sup id="r8">[8)](#f8)</sup>
  - 이미 존재하는 task를 재사용한다.
  - FLAG_ACTIVITY_MULTIPLE_TASK를 세팅하지 않은 FLAG_ACTIVITY_NEW_DOCUMENT와 같다.
- always
  - document가 open되어 있어도 항상 new task를 만든다.
  - FLAG_ACTIVITY_MULTIPLE_TASK, FLAG_ACTIVITY_NEW_DOCUMENT를 같이 사용한 것과 같다.<sup id="r9">[9)](#f9)</sup>
- none
  - document를 위해 새로운 task를 만들지 않는다.
  - 아무것도 설정하지 않았을 경우의 default attribute
- never
  - document를 위해 새로운 task를 만들지 않는다.
  - FLAG_ACTIVITY_MULTIPLE_TASK, FLAG_ACTIVITY_NEW_DOCUMENT를 override한다.
    - 둘 중 하나가 설정되어도 Recents screen에는 app을 위한 한 개의 task를 나타낸다.

### [Remove tasks]
- 기본적으로 activity가 finish하면 document task는 Recents screen에서 자동으로 제거된다.
- < activity > attribute, android:excludeFromRecents를 true로 하면 항상 Recents screen에서 제외될 수 있다.
- < activity > attribute, android:maxRecents에 integer value를 지정하면 Recents screen에 나타낼 수 있는 app의 최대 task 개수를 지정할 수 있다.
  - 기본 개수는 16개이다.
  - 만약 maximum 개수에 도달하게 되면 가장 오래 전에 사용된 task부터 제거된다.
  - maximum으로 지정할 수 있는 개수는 50개이다.(low memory device에서는 25개)
  - 1개보다 적은 수는 invalid하다.
#### Use the AppTask class to remove tasks
```kotlin
fun onRemoveFromOverview(view: View) {
    // It is good pratice to remove a document from the overview stack if not needed anymore.
    finishAndRemoveTask()
}
```
- finishAndRemoveTask()를 호출해서 언제 task와 관련 activities를 remove하는지 정할 수 있다.
#### Retain finished tasks
```kotlin
private fun newDocumentIntent() =
        Intent(this, NewDocumentActivity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_DOCUMENT or
                    android.content.Intent.FLAG_ACTIVITY_RETAIN_IN_RECENTS)
            putExtra(KEY_EXTRA_NEW_DOCUMENT_COUNTER, getAndIncrement())
        }
```
- activity가 finished 됐어도 Recents screen에 task로 유지하고 싶으면 FLAG_ACTIVITY_RETAIN_IN_RECENTS flag를 사용한다.
- android:autoRemoveFromRecents를 false로 두어도 같다.

### [Enable recents URL sharing(Pixel only)]
- Android 12 이상의 pixel에서는 Recents screen에서 직접 web link를 공유할 수 있다.

## Restrictions on starting activities from the background
- API level 29 이상부터 app이 background에서 실행될 때 Activity를 시작할 수 있는 시점에 제한이 있다.
- background에서 Activity를 시작하는 대신 notification을 사용하는 guide를 제시한다.
### [Display notifications instead]
- background에서 실행되는 app은 직접 Activity를 시작하지 않는 대신 time-sensitive notification을 표시하여 긴급한 정보를 user에게 전달한다.
- 이런 알림 시스템은 다음과 같은 장점을 제공한다.
  - device 사용 시 사용자에게 응답할 수 있는 notification을 제공한다.
    - 사용자는 현재 상태를 유지하고 screen에 보이는 content를 control할 수 있다.
  - time-sensitive notification은 방해 금지 모드를 지킨다.
  - device's screen이 꺼져 있으면 full-screen intent가 바로 시작된다.
  - device's Settings screen을 통해 최근에 어떤 app이 noticiation을ㄹ 보냈는지 확인할 수 있다.
### [Exceptions to the restriction]
- Android 10 이상의 app에서는 다음 중 하나 이상의 조건을 만족할 때 activities를 시작할 수 있다.
  - app이 visible window를 가질 때
  - app이 foreground task의 back stack에 있는 activity를 가진 경우
  - app의 Recents screen의 기존 작업 back stack에 있는 activity를 가진 경우
  - 가장 최근에 시작한 activity를 가진 경우
  - app이 가장 최근에 activity에서 finish()를 호출한 경우
    - finish()가 호출된 시점에 activity가 foreground에 있거나 foreground task의 back stack에 있는 경우 해당
  - system에 binding된 service가 있는 경우
  - 다른 visible app에 binding된 service가 있는 경우
  - app이 다른 visible app에서 전송된 PendingIntent를 받은 경우
  - app이 UI를 실행해야 하는 system Broadcast를  경우
    - Broadcast가 전송되고 몇 초 후에 app이 activity를 시작할 수 있다
  - app이 CompanionDeviceManager API를 통해 부속 하드웨어 기기와 연결되는 경우
    - app은 사용자가 페어링된 기기에서 수행하는 작업에 대한 응답으로 작업을 시작할 수 있다.

## Q&A
<b id="f1">1) </b>onBackPressedDispatcher를 사용하라는 뜻일까?[↩](#r1)<br>
- OnBackPressedCallback abstract method인 handleOnBackPressed() 사용
- Dispatcher에 Callback을 등록
- 만약 onBackPressed()를 override한다면 반드시 super.onBackPressed()를 사용해야 한다.
- [OnBackPressedDispatcher](https://developer.android.com/reference/androidx/activity/OnBackPressedDispatcher)
```kotlin
public class FormEntryFragment extends Fragment {
    @Override
    public void onAttach(@NonNull Context context) {
        super.onAttach(context);
        OnBackPressedCallback callback = new OnBackPressedCallback(
            true // default to enabled
        ) {
            @Override
            public void handleOnBackPressed() {
                showAreYouSureDialog();
            }
        };
        requireActivity().getOnBackPressedDispatcher().addCallback(
            this, // LifecycleOwner
            callback);
    }
}
```

<b id="f2">2) </b>back button을 해도 root launcher activity는 finish되지 않는다면 destroy되지 않고 이전 상태를 그대로 유지하고 있다는 뜻인가? 그러면 destroy시키기 위해 finish()를 사용하는 것을 생각했는데 finishing 하지 말고 super.onBackPressed() 호출하도록 하라는 것은 root launcher activity를 destroy시키지 말라는 뜻일까?[↩](#r2)<br>
- However, if your app overrides onBackPressed() to handle Back navigation and finish the activity, update your implementation to call through to super.onBackPressed() instead of finishing
```kotlin
override fun onBackPressed() {
    onBackPressedDispatcher.addCallback {
        finish()
    }
    super.onBackPressed()
}
```
```kotlin
override fun onBackPressed() {
    finish()
    super.onBackPressed()
}
```


<b id="f3">3) </b> 만약 A-B-C에서 B에 singletask를 했다면?[↩](#r3)<br>
- B 위의 C가 destroy되고 B가 onNewIntent()로 실행되면서 A-B가 된다.


<b id="f4">4) </b> 만약 singletask에서 taskAffinity가 없다면?[↩](#r4)<br>
- affinity는 task name이며 따로 지정하지 않으면 package name을 사용한다.
- affinity를 지정하지 않는다면 singletask여도 같은 task에 쌓이게 된다.
- 물론 다시 같은 activity를 호출하게 되면 재사용된다.
- 즉, singletask는 하나의 task에 activity instance는 무조건 한 개 있어야 한다는 것을 의미한다.
- 이는 singleInstance도 같은 현상이 나타날 수 있다.


<b id="f5">5) </b> marshalling, unmarshalling이란[↩](#r5)<br>
- Serialization은 객체 데이터를 일련의 byte stream으로 변환하는 작업
- Marshalling은 객체의 메모리 구조를 저장이나 전송을 위해 필요로 하는 자료 형태로 변환하는 작업
  - Bundle은 parcel을 이용해서 marshalling한다고 되어 있는데 그러면 marshalling은 전송을 위해 사용하는 변화 과정 일련을 말하는 것일까?
- Serialization과 Parceable, Marshalling의 정확한 차이는 무엇이며 분명 Bundle은 parcelable를 사용하여 비용이 드는 작업이라고 했었는데 잘못 이해한 것인가?

<b id="f6">6) </b> TransactionTooLargeException [↩](#r6)<br>
- Parcel objects는 Binder transaction buffer를 사용한다.
- Binder transaction buffer는 1MB의 고정된 size를 가진다.
- [TransactionTooLargeException](https://developer.android.com/reference/android/os/TransactionTooLargeException)

<b id="f7">7) </b> FLAG_ACTIVITY_NEW_DOCUMENT와 FLAG_ACTIVITY_MUTIPLE_TASK의 차이[↩](#r7)<br>
- NEW_DOCUMENT의 경우 이미 activity가 seperate task에 있다면 새롭게 생성되지 않고 존재하는 것을 onNewIntent()를 통해 사용한다.
- MULTIPLE_TASK의 경우 이미 존재해도 항상 새로운 task를 만들어낸다.
  - 단, NEW_TASK와 함께 사용할 경우에만 작동한다.
- 둘다 back action에 대해서 이전 activity로 돌아간다.
  - 단, NEW_DOCUMENT는 중복된 것은 task가 새롭게 생성되지 않기 때문에 중복 제외 호출된 순서대로 돌아간다.
  - NEW_DOCUMENT의 경우 root activity로 new task가 생성되기 때문에 back action에 의해서 Recents screen에서 사라지지 않는다.
  - NEW_TASK의 경우 root activity만 남는다.

<b id="f8">8) </b> documentLaunchMode=intoExisting과 FLAG_ACTIVITY_NEW_DOCUMENT의 차이[↩](#r8)<br>
- 기본적으로 Recents screen에 나타나는 구조는 같다.
- back action 후 Recents screen에 남아있는 것이 다르다.
  - intoExisting의 경우 생성된 task가 전부 Recents screen에 남아 있다.

<b id="f9">9) </b>FLAG_ACTIVITY_MULTIPLE_TASK, FLAG_ACTIVITY_NEW_DOCUMENT를 같이 사용[↩](#r9)<br>
- FLAG_ACTIVITY_MULTIPLE_TASK, FLAG_ACTIVITY_NEW_TASK를 같이 사용한 것과 같은 결과를 나타낸다.
