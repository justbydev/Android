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
<b id="f3">3) </b> 만약 A-B-C에서 B에 singletask를 했다면?[↩](#r3)<br>
- B 위의 C가 destroy되고 B가 onNewIntent()로 실행되면서 A-B가 된다.
<b id="f4">4) </b> 만약 singletask에서 taskAffinity가 없다면?[↩](#r4)<br>
- affinity는 task name이며 따로 지정하지 않으면 package name을 사용한다.
- affinity를 지정하지 않는다면 singletask여도 같은 task에 쌓이게 된다.
- 물론 다시 같은 activity를 호출하게 되면 재사용된다.
- 즉, singletask는 하나의 task에 activity instance는 무조건 한 개 있어야 한다는 것을 의미한다.
- 이는 singleInstance도 같은 현상이 나타날 수 있다.
