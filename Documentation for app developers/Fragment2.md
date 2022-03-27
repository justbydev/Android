# Fragments(Saving state, Communication with fragments, DailogFragment, Debug, Test)
## Saving state with fragments
- 다양한 Android system operations는 fragment state에 영향을 준다.
- user state가 save되는 것이 보장되려면 Android framework는 fragment와 back stack을 자동으로 save하고 restore해야 한다.
<img width="777" alt="스크린샷 2022-03-16 오후 1 52 12" src="https://user-images.githubusercontent.com/17876424/158519069-e6063407-051a-4cba-b883-fb8531f9562c.png">


- Variables: fragment local variables
- View State: fragment의 하나 이상의 view가 가진 data
- SavedState: onSaveInstanceState()에 저장되어야 하는 fragment instance의 고유한 data
- NonConfig: server or local repository와 같은 외부 소스에서 가져온 data 혹은 commit된 후에 server에게 전송되는 user-created data
- 가끔 variable을 savedstate과 동일하게 취급하지만 위의 테이블에서처럼 차이를 보인다.


### [View state]
- Views는 그들의 state을 관리해야 한다.
  - 예를 들어 view가 user input을 받으면 configuration change에 대해 save하고 restore해야 한다.
- Android framework에서 제공하는 모든 views는 자체 onSaveInstanceState() 및 onRestoreInstanceState()가 구현되어 있기 때문에 fragment 내에서 view state를 관리하지 않아도 된다.
- view state을 유지하려면 ID가 있어야 한다.
  - 이 ID는 fragment와 view hierarchy에서 반드시 unique해야 하고 ID가 없는 view는 state을 유지할 수 없다.
- view는 fragment를 remove하거나 host를 destroy 하지않는 모든 operation에 대해 ViewState를 save하고 restore한다.

### [SavedState]
- fragment는 fragment 작동에 필수적인 적은 양의 dynamic state를 관리해야 한다.
- 이를 위해 Fragment.onSaveInstanceState(Bundle)을 사용할 수 있다.
  - Activity.onSaveInstanceState(Bundle)과 비슷하며 data가 bundle에 저장되어 configuration change, process death and recreation 상황에서 유지된다.
  - 이 데이터는 fragment의 onCreate(Bundle), onCreateView(LayoutInflater, ViewGroup, Bundle), onViewCreated(View, Bundle)에서 받을 수 있다.
  - 주의할 것은 fragment의 onSaveInstanceState는 host activity의 onSaveInstanceState가 호출될 때만 호출된다.
- Tip: ViewModel을 사용한다면 SavedStateHandle을 사용할 수 있다.
- 위의 표에서 보았듯 variable은 fragment가 backstack에 위치해도 유지된다. variable을 saved state으로 treat하면 어떠한 destructive operation에서도 유지된다.

### [NonConfig]
- NonConfig data는 ViewModel과 같이 fragment 바깥에 위치해야 한다.
```kotlin
public class RandomGoodDeedViewModel : ViewModel() {
    private val seed = ... // Generate the seed

    private fun generateRandomGoodDeed(): String {
        val goodDeed = ... // Generate a random good deed using the seed
        return goodDeed
    }
}
```
- Viewmodel class는 configuration change로부터 data를 유지한다.
- fragment가 back stack에 위치한다면 memory에 유지된다.
- process death and recreation 후 ViewModel이 다시 생성되고 새로운 seed가 생성된다.
  - 위의 표에서처럼 NonConfig state는 만약 SavedState module을 ViewModel에 추가하면 process death and recreation 상황에서 유지된다.<sup id="ra">[a)](#fa)</sup>



## Communicating with fragments
- fragment를 재사용하려면 자체 layout과 behavior를 정의하는 완전히 독립적인 구성요소로 각 fragment를 build한다.
  - reusable fragment를 정의하면 activity와 application logic과 연결하여 overral composit UI를 실현할 수 있다.
- user event에 제대로 반응하고 state information을 공유하려면 activity와 fragment, 2개 이상의 fragment끼리 소통할 수 있는 channel이 필요하다.
  - fragment를 독립적인 구성요소로 유지하려면 절대 다른 fragment나 host activity와 직접적으로 소통해서는 안된다.
- Fragment library는 ViewModel share와 Fragment Result API, 2가지 option을 제공한다.

### [Share data using a ViewModel]
#### Share data with the host activity
```kotlin
class ItemViewModel : ViewModel() {
    private val mutableSelectedItem = MutableLiveData<Item>()
    val selectedItem: LiveData<Item> get() = mutableSelectedItem

    fun selectItem(item: Item) {
        mutableSelectedItem.value = item
    }
}
```
```kotlin
class MainActivity : AppCompatActivity() {
    // Using the viewModels() Kotlin property delegate from the activity-ktx
    // artifact to retrieve the ViewModel in the activity scope
    private val viewModel: ItemViewModel by viewModels()
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        viewModel.selectedItem.observe(this, Observer { item ->
            // Perform an action with the latest item data
        })
    }
}

class ListFragment : Fragment() {
    // Using the activityViewModels() Kotlin property delegate from the
    // fragment-ktx artifact to retrieve the ViewModel in the activity scope
    private val viewModel: ItemViewModel by activityViewModels()

    // Called when the item is clicked
    fun onItemClicked(item: Item) {
        // Set a new item
        viewModel.selectItem(item)
    }
}
```
- fragment끼리, 혹은 host activity와 fragment가 data를 share가 필요할 때 다음과 같이 ViewModel을 사용할 수 있다.
- fragment와 host activity는 전부 ViewModelProvider constructor에게 activity를 전달해서 activity scope을 가지는 ViewModel instance를 공유받게 된다.
  - ViewModelProvider가 Viewmodel 인스턴스화하거나 이미 존재한다면 이미 존재하는 ViewModel을 찾아온다.
  - fragment와 host activity 모두 selectItem을 observe하거나 수정할 수 있다.

#### Share data between fragments
- 같은 activity에 있는 2개 이상의 fragments가 서로 소통해야할 때가 있다.
  - 이런 경우 fragments끼리 직접 소통하면 fragments의 독립적 구성 요소가 유지되지 않는다.
- 이런 fragments는 activity scope을 사용하는 ViewModel을 통해 소통할 수 있다.
  - 이런 방식을 사용하면 fragments가 서로에 대해 알 필요가 없고 activity도 소통을 용이하게 하기 위해 아무것도 할 필요가 없다.
```kotlin
class ListViewModel : ViewModel() {
    val filters = MutableLiveData<Set<Filter>>()

    private val originalList: LiveData<List<Item>>() = ...
    val filteredList: LiveData<List<Item>> = ...

    fun addFilter(filter: Filter) { ... }

    fun removeFilter(filter: Filter) { ... }
}

class ListFragment : Fragment() {
    // Using the activityViewModels() Kotlin property delegate from the
    // fragment-ktx artifact to retrieve the ViewModel in the activity scope
    private val viewModel: ListViewModel by activityViewModels()
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewModel.filteredList.observe(viewLifecycleOwner, Observer { list ->
            // Update the list UI
        }
    }
}

class FilterFragment : Fragment() {
    private val viewModel: ListViewModel by activityViewModels()
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewModel.filters.observe(viewLifecycleOwner, Observer { set ->
            // Update the selected filters UI
        }
    }

    fun onFilterSelected(filter: Filter) = viewModel.addFilter(filter)

    fun onFilterDeselected(filter: Filter) = viewModel.removeFilter(filter)
}
```
- 두 fragment는 모두 ViewModelProvider에서 host activity를 scope으로 사용하고 있다.
  - fragments가 같은 scope을 사용하기 때문에 같은 Viewmodel instance를 받는다.
- ViewModel은 scope이 지정된 ViewModelStoreOwner가 영구적으로 사라질 때까지 메모리에 유지된다.
  - single activity architecture에서 ViewModel이 activity에 scope되었다면 그 ViewModel은 기본적으로 singleton이다.
  - ViewModel이 처음 인스턴스화된 후 activity scope을 사용하는 ViewModel을 찾는 subsequent call은 activity lifecycle이 영구적으로 끝날때까지 existing data와 함께 same existing ViewModel을 return한다.

#### Share data between a parent and child fragment
```kotlin
class ListFragment: Fragment() {
    // Using the viewModels() Kotlin property delegate from the fragment-ktx
    // artifact to retrieve the ViewModel
    private val viewModel: ListViewModel by viewModels()
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewModel.filteredList.observe(viewLifecycleOwner, Observer { list ->
            // Update the list UI
        }
    }
}

class ChildFragment: Fragment() {
    // Using the viewModels() Kotlin property delegate from the fragment-ktx
    // artifact to retrieve the ViewModel using the parent fragment's scope
    private val viewModel: ListViewModel by viewModels({requireParentFragment()})
    ...
}
```
- parent fragment와 child fragment끼리 data를 share할 때는 parent fragment를 ViewModel scope로 사용한다.

#### Scoping a ViewModel to the Navigation Graph
```kotlin
class ListFragment: Fragment() {
    // Using the navGraphViewModels() Kotlin property delegate from the fragment-ktx
    // artifact to retrieve the ViewModel using the NavBackStackEntry scope
    // R.id.list_fragment == the destination id of the ListFragment destination
    private val viewModel: ListViewModel by navGraphViewModels(R.id.list_fragment)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewModel.filteredList.observe(viewLifecycleOwner, Observer { item ->
            // Update the list UI
        }
    }
}
```
- 만약 Navigation library를 사용한다면 ViewModel을 destination의 NavBackStackEntry lifecycle에 scope할 수 있다.

### [Get results using the Fragment Result API]
- Fragment 1.3.0-alpha04부터 각각의 FragmentManager는 FragmentResultOwner를 implement한다.
  - 즉, FragmentManager가 fragment results를 저장하는 central store 역할을 할 수 있다는 뜻이다.
  - 이는 fragment results를 setting해서 component끼리 소통을 하고 서로간의 직접적인 reference 없이 results를 알 수 있게 한다.

#### Pass results between fragments
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // Use the Kotlin extension in the fragment-ktx artifact
    setFragmentResultListener("requestKey") { requestKey, bundle ->
        // We use a String here, but any type that can be put in a Bundle is supported
        val result = bundle.getString("bundleKey")
        // Do something with the result
    }
}
```
```java
@Override
public void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    getParentFragmentManager().setFragmentResultListener("requestKey", this, new FragmentResultListener() {
        @Override
        public void onFragmentResult(@NonNull String requestKey, @NonNull Bundle bundle) {
            // We use a String here, but any type that can be put in a Bundle is supported
            String result = bundle.getString("bundleKey");
            // Do something with the result
        }
    });
}
```
<img width="725" alt="스크린샷 2022-03-16 오후 8 13 33" src="https://user-images.githubusercontent.com/17876424/158577791-cbbf8973-2314-481d-8c8e-88c9d06db692.png">

- 만약 fragment A가 fragment B로부터 data를 받고 싶다면 fragment A에 result listener를 set한다.
  - fragment A의 FragmentManager에 setFragmentResultListener()를 call한다.
```kotlin
button.setOnClickListener {
    val result = "result"
    // Use the Kotlin extension in the fragment-ktx artifact
    setFragmentResult("requestKey", bundleOf("bundleKey" to result))
}
```
- fragment B에서는 result를 생산하고 같은 FragmentManager에서 같은 requestKey를 사용해서 result를 set한다.
  - setFragmentResult() API를 사용한다.
  - 그러면 fragment A는 result를 받고 fragment A가 STARTED가 되면 listener callback이 execute한다.

- 주어진 key에 대해 오직 하나의 listener와 결과를 받을 수 있다.
- 만약 같은 key에 대해 setFragmentResult()를 두 번 이상 하고 listener가 STARTED가 아니라면 system은 대기중인 result를 update된 result로 replace한다.<sup id="r1">[1)](#f1)</sup>
- 만약 result를 받을 corresponding listener 없이 result를 set하면 same key를 사용하는 listener가 set될때까지 FragmentManager가 result를 저장하고 있는다.
- listener가 결과를 수신하여 onFragmentResult() callback을 실행하면 결과는 clear된다.
- 위와 같은 behavior는 다음의 2가지 주요 영향이 있다.
  - back stack에 있는 Fragments는 pop되어 STARTED가 될 때까지 results를 받지 않는다.
  - 만약 result가 set됐을 때 그 result를 listen하는 fragment가 STARTED라면 listener callback이 바로 실행된다.
- fragment results는 FragmentManager level에 저장되기 때문에 setFragmentResultListener()나 setFragmentResult()는 parent FragmentManager에 attach되어야 한다.

#### Test fragment results
- FragmentScenario를 사용해서 setFragmentResult(), setFragmentResultListener()를 테스트한다.
- launchFragmentInContainer 또는 launchFragment를 사용하여 테스트 중인 fragment의 scenario를 만들고 테스트 중이 아닌 method를 수동으로 호출한다.
- setFragmentResultListener()를 테스트하려면 이를 호출하는 fragment로 scenario를 만들고 setFragmentResult()를 직접 호출한다.
```kotlin
@Test
fun testFragmentResultListener() {
    val scenario = launchFragmentInContainer<ResultListenerFragment>()
    scenario.onFragment { fragment ->
        val expectedResult = "result"
        fragment.parentFragmentManager.setFragmentResult("requestKey", bundleOf("bundleKey" to expectedResult))
        assertThat(fragment.result).isEqualTo(expectedResult)
    }
}

class ResultListenerFragment : Fragment() {
    var result : String? = null
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Use the Kotlin extension in the fragment-ktx artifact
        setFragmentResultListener("requestKey") { requestKey, bundle ->
            result = bundle.getString("bundleKey")
        }
    }
}
```
- setFragmentResult()를 테스트하려면 이를 호출하는 fragment에서 scenario를 만들고 setFragmentResultListener()를 직접 호출한다.
```kotlin
@Test
fun testFragmentResult() {
    val scenario = launchFragmentInContainer<ResultFragment>()
    lateinit var actualResult: String?
    scenario.onFragment { fragment ->
        fragment.parentFragmentManager
                .setFragmentResultListener("requestKey") { requestKey, bundle ->
            actualResult = bundle.getString("bundleKey")
        }
    }
    onView(withId(R.id.result_button)).perform(click())
    assertThat(actualResult).isEqualTo("result")
}

class ResultFragment : Fragment(R.layout.fragment_result) {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        view.findViewById(R.id.result_button).setOnClickListener {
            val result = "result"
            // Use the Kotlin extension in the fragment-ktx artifact
            setFragmentResult("requestKey", bundleOf("bundleKey" to result))
        }
    }
}
```
#### Pass results between parent and child fragments
- child fragment에서 parent로 result를 전달하려면 parent fragment에서 getChildFragmentManager()를 사용해서 setFragmentResultListener()를 호출한다.
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // We set the listener on the child fragmentManager
    childFragmentManager.setFragmentResultListener("requestKey") { key, bundle ->
        val result = bundle.getString("bundleKey")
        // Do something with the result
    }
}
```
- child fragment에서는 FragmentManager를 통해 result를 set한다.
- parent는 fragment가 STARTED가 되면 result를 받는다.
```kotlin
button.setOnClickListener {
    val result = "result"
    // Use the Kotlin extension in the fragment-ktx artifact
    setFragmentResult("requestKey", bundleOf("bundleKey" to result))
    //getParentFragmenetManager().setFragmentResult()
}
```
<img width="607" alt="스크린샷 2022-03-20 오후 4 46 13" src="https://user-images.githubusercontent.com/17876424/159153136-2d9808ea-748e-450d-be75-0ba74cab6ca5.png">

#### Receive results in the host activity
```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        supportFragmentManager
                .setFragmentResultListener("requestKey", this) { requestKey, bundle ->
            // We use a String here, but any type that can be put in a Bundle is supported
            val result = bundle.getString("bundleKey")
            // Do something with the result
        }
    }
}
```
- host activity가 fragment result를 받으려면 getSupportFragmentManage()를 사용해서 result listener를 set한다.

## Displaying dialogs with DialogFragment
- DialogFragment는 dialog를 create하고 host하도록 design된 special fragment subclass다.
  - 엄밀히 말하면 fragment 내에 dialog를 host할 필요는 없지만 그렇게 한다면 FragmentManager가 dialog state를 관리하고 configuration change가 발생했을 때 자동으로 dialog를 복원하게 한다.

### [Create a DialogFragment]
```kotlin
class PurchaseConfirmationDialogFragment : DialogFragment() {
    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog =
            AlertDialog.Builder(requireContext())
                .setMessage(getString(R.string.order_confirmation))
                .setPositiveButton(getString(R.string.ok)) { _,_ -> }
                .create()

    companion object {
        const val TAG = "PurchaseConfirmationDialog"
    }
}
```
- DialogFragment를 create하기 위해 DialogFragment를 extend하고 onCreateDialog()를 override한다.
- onCreateView()에서 root view를 create했듯이 onCreateDailog()에서는 diplay할 Dialog를 create한다.
  - DialogFragment는 fragment lifecycle state에 맞게 Dialog를 display하는 것을 관리한다.
  - onCreateView()과 비슷하게 any Dialog subclass를 return하고 AlertDialog에만 제한되지 않는다.
- DialogFragment는 Dialog.setOnCancelListener(), Dialog.setOnDismissListener() callback을 갖고 있기 때문에 set하지 않아도 된다.
  - 이 event를 조작하려면 onCancel(), onDismiss()를 override한다.

### [Showing the DialogFragment]
```kotlin
// From another Fragment or Activity where you wish to show this
// PurchaseConfirmationDialogFragment.
PurchaseConfirmationDialogFragment().show(
     childFragmentManager, PurchaseConfirmationDialog.TAG)
```
- DialogFragment를 display할 때 FragmentTransaction을 create할 필요 없이 show() method를 사용한다.
  - FragmentManager reference와 FragmentTransaction tag로 사용할 String을 넘길 수 있다.
- 만약, fragment안에서 DialogFragment를 create하면 Fragment의 child FragmentManager를 사용해야 한다.<sup id="r2">[2)](#f2)</sup>
  - configuration change 이후 state가 제대로 restore되기 위해서
- non-null tag를 사용하면 findFragmentByTag()를 통해 DialogFragment를 받을 수 있다.
- DialogFragment는 configuration change 이후 자동으로 restore되기 때문에 user action에 의해 dialog를 호출하 때 show() 호출만을 고려하거나 findFragmentByTag()가 null을 return한다면 dialog가 더 이상 존재하지 않는다는 것만 신경쓰면 된다.

### [DialogFragment lifecycle]
- DialogFragment는 a few additional lifecycle callbacks를 갖고 있다.



  - onCreateDialog()
    - fragment가 관리하고 display할 수 있도록 Dialog를 제공하는 callback
  - onDismiss()
    - Dialog가 dismiss될 때 custom logic을 수행해야 할 때 사용하는 callback
    - resource release, observable resource unsubscribe 등의 custom logic
  - onCancle()
    - Dialog가 cancel될 때 custom logic을 수행해야 할 때 사용하는 callback
- 또한 DialogFragment는 dismiss나 cancellability를 설정할 수 있는 method를 갖고 있다.
  - dismiss()<sup id="rb">[b)](#fb)</sup>
    - fragment와 그 dialog를 dismiss한다.
    - 만약 fragment가 back stack에 add되었다면 이 entry를 포함하여 그 위의 모든 back stack state가 pop된다.
    - 아니라면 fragment를 remove하기 위한 new transaction이 commit된다.
  - setCancellable()
    - 나타난 Dialog가 cancelable한지 control한다.
    - Dialog.setCancelable(boolean)을 직접 호출하는 대신 사용해야 한다.
- DialogFragment를 Dialog와 함께 사용할 때 onCreateView() 또는 onViewCreated()를 override하지 않는다.<sup id="r3">[3)](#f3)</sup>
  - Dialogs are not only views—they have their own window
    - 따라서 onCreateView()를 override하기에 insufficient하다.
  - onCreateView()를 override하고 non-null view를 제공하지 않는 한 custom DialogFragment에서 onViewCreated()는 호출되지 않는다.

### [Using custom views]
- onCreateView()를 override해서 DialogFragment를 create하고 dialog를 display할 수 있다.
  - 일반 fragment와 마찬가지로 onCreateView에게 layoutId을 지정한다.
  - 혹은 Fragment 1.3.0-alpha02에 도입된 DialogFragment construstor를 사용한다.
  ```kotlin
  class MyDialogFragment extends DialogFragment {
    public MyDialogFragment() {
      super(R.layout.dialog_fragment_main);
    }
  }
  ```
- onCreateView()에 return된 View는 자동으로 dialog에 add된다.
  - default empty dialog가 직접 만든 custom view로 채워지기 때문에 onCreateDialog()를 override할 필요 없다.
  - BottomSheetDialogFragment의 경우 bottom sheet로 지정된 dialog에 view를 포함한다.
  
## Debug your fragments
### [FragmentManager logging]
- FragmentManager는 다양한 메시지를 Logcat에 내보낼 수 있다.
- Logcat에 noise를 보내지 않도록 사용을 중지할 수 있지만 때로는 log message는 fragment와 관련된 troubleshoot issue에 대해 도움을 준다.
- FragmentManager는 Debug and Verbose log levels에 의미있는 output을 내보낸다.
```
adb shell setprop log.tag.FragmentManager DEBUG
```
```
adb shell setprop log.tag.FragmentManager VERBOSE
```
- adb shell을 사용하여 logging을 사용할 수 있다.
- 만약 verbose logging을 사용할 수 있으면 Logcat window에서 log level filter를 적용할 수 있다.
  - 하지만 FragmentManager log뿐 아니라 모든 log를 filter하기 때문에 log level에서 FragmentManager logging만 사용하는 것이 좋다.
#### DEBUG logging
- DEBUG level에서 FragmentManager는 lifecycle state 변경과 관련된 log message를 보낸다.
- 각 log 항목에는 Fragment의 toString() dump가 포함된다.
- log 항목은 다음 정보로 구성된다.
  - Fragment instance의 간단한 class name
  - Fragment instance의 ID hash code
  - Fragment instance의 FragmentManager unique ID. configuration change 및 process death, recreation에 있어 stable하다.
  - Fragment가 add된 container ID(단, setting되었다면)
  - Fragment tag(단, setting되었다면)
```
D/FragmentManager: moveto ATTACHED: NavHostFragment{92d8f1d} (fd92599e-c349-4660-b2d6-0ece9ec72f7b id=0x7f080116)
```
  - Fragment class name: NavHostFragment
  - ID hash hode: 92d8f1d
  - unique ID: fd92599e-c349-4660-b2d6-0ece9ec72f7b
  - container ID: 0x7f080116
  - 현재 tag는 없는데 있다면 tag=tag_value 형식으로 보여준다.
- 또한 간결성과 가독성을 위해 unique ID가 UUID로 축약되고 모든 fragment instance에 lifecycle state가 접미사로 추가된다.
```
D/FragmentManager: movefrom RESUMED: FirstFragment{ccd2189} (<UUID> id=0x7f080116)
D/FragmentManager: movefrom STARTED: FirstFragment{ccd2189} (<UUID> id=0x7f080116)
D/FragmentManager: movefrom ACTIVITY_CREATED: FirstFragment{ccd2189} (<UUID> id=0x7f080116)
D/FragmentManager: moveto ATTACHED: SecondFragment{84132db} (<UUID> id=0x7f080116)
D/FragmentManager: moveto CREATED: SecondFragment{84132db} (<UUID> id=0x7f080116)
```
#### VERBOSE logging
- VERBOSE level에서 FragmentManager는 주로 internal state를 log message로 보낸다.
```
V/FragmentManager: Run: BackStackEntry{f9d3ff3}
V/FragmentManager: add: NavHostFragment{86274b0} (<UUID> id=0x7f080130)
V/FragmentManager: Added fragment to active set NavHostFragment{86274b0} (<UUID> id=0x7f080130)
V/FragmentManager: computeExpectedState() of 1 for NavHostFragment{86274b0} (<UUID> id=0x7f080130)
D/FragmentManager: moveto ATTACHED: NavHostFragment{86274b0} (<UUID> id=0x7f080130)
V/FragmentManager: Commit: BackStackEntry{5cfd2ae}
D/FragmentManager:   mName=null mIndex=-1 mCommitted=false
D/FragmentManager:   Operations:
D/FragmentManager:     Op #0: SET_PRIMARY_NAV NavHostFragment{86274b0} (<UUID> id=0x7f080130)
V/FragmentManager: computeExpectedState() of 1 for NavHostFragment{86274b0} (<UUID> id=0x7f080130)
D/FragmentManager: moveto CREATED: NavHostFragment{86274b0} (<UUID> id=0x7f080130)
V/FragmentManager: Commit: BackStackEntry{e93833f}
D/FragmentManager:   mName=null mIndex=-1 mCommitted=false
D/FragmentManager:   Operations:
D/FragmentManager:     Op #0: REPLACE FirstFragment{886440c} (<UUID> id=0x7f080130)
D/FragmentManager:     Op #1: SET_PRIMARY_NAV FirstFragment{886440c} (<UUID> id=0x7f080130)
V/FragmentManager: Run: BackStackEntry{e93833f}
V/FragmentManager: add: FirstFragment{886440c} (<UUID> id=0x7f080130)
V/FragmentManager: Added fragment to active set FirstFragment{886440c} (<UUID> id=0x7f080130)
V/FragmentManager: computeExpectedState() of 1 for FirstFragment{886440c} (<UUID> id=0x7f080130)
D/FragmentManager: moveto ATTACHED: FirstFragment{886440c} (<UUID> id=0x7f080130)
```
- VERBOSE level의 log message는 상당수는 앱 개발자에게 거의 쓸모가 없지만 back stack이 변경되는 시점을 확인하면 일부 문제를 debug하는데 도움이 될 수도 있다.

### [StrictMode for fragments]
- Jetpack Fragment library 1.4.0-alpha01 version부터 fragments StrictMode가 소개되었으며 이는 app에 unexpected하게 동작할 수 있는 common issue를 잡을 수 있다.
- 기본적으로 Fragment StrictMode에는 아무것도 catch하지 않는 LAX policy가 있지만 custom policy를 만들 수도 있다.
  - custom policy는 감지할 위반을 감지하고 감지하면 어떤 penalty를 줄지 명시한다.
  - FragmentManager DEBUG logging을 사용하면 어떤 policy를 사용하든 StrictMode violation을 log할 수 있다.
- custom StrictMode policy를 적용하려면 FragmentManager에 할당한다.
  - 할 수 있는 한 빨리 해야 한다.
```kotlin
class ExampleActivity : AppCompatActivity() {

    init {
        supportFragmentManager.strictModePolicy =
            FragmentStrictMode.Policy.Builder()
                .penaltyDeath()
                .detectFragmentReuse()
                .allowViolation(FirstFragment::class.java,
                                FragmentReuseViolation::class.java)
                .build()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val binding = ActivityExampleBinding.inflate(layoutInflater)
        setContentView(binding.root)
        ...
   }
}
```
- 만약 strict mode의 사용 가능 여부를 Context를 사용해야 한다면 onContextAvailableListener를 사용한다.
```kotlin
class ExampleActivity : AppCompatActivity() {

    init {
        addOnContextAvailableListener { context ->
            if(context.resources.getBoolean(R.bool.enable_strict_mode)) {
                supportFragmentManager.strictModePolicy = FragmentStrictMode.Policy.Builder()
                    .penaltyDeath()
                    .detectFragmentReuse()
                    .allowViolation(FirstFragment::class.java, FragmentReuseViolation::class.java)
                    .build()
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val binding = ActivityExampleBinding.inflate(layoutInflater)
        setContentView(binding.root)
        ...
   }
}
```
- 가능한 모든 violations을 catch하도록 strict mode를 구성해야 하는 가장 마지막 지점은 onCreate()로 super.onCreate() 이전에 해야 한다.<sup id="rc">[c)](#fc)</sup>
```kotlin
class ExampleActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        supportFragmentManager.strictModePolicy = FragmentStrictMode.Policy.Builder()
            .penaltyDeath()
            .detectFragmentReuse()
            .allowViolation(FirstFragment::class.java, FragmentReuseViolation::class.java)
            .build()

        super.onCreate(savedInstanceState)

        val binding = ActivityExampleBinding.inflate(layoutInflater)
        setContentView(binding.root)
        ...
   }
}
```
- 위에서는 FirstFragment에 대한 fragment reuse violation에 대해서만 detect하고 이를 감지하면 app이 종료된다.
- 특정 violation을 선택적으로 허용할 수도 있다.
  - 하지만 위 예에서는 이 policy는 다른 모든 fragment type에 대해 violation을 강제한다.
  - 이는 third-party library component가 StrictMode violations을 포함할 때 유용하다.
    - 이런 경우 library에서 violation을 수정할 때까지, third-party component의 StrictMode 허용 목록에 이 violation을 일시적으로 추가할 수 있다.<sup id="rd">[d)](#fd)</sup>
- 다음은 3가지 penalty type이다.
  - penaltyLog()는 violation 세부정보를 LogCat에 dump한다.
  - penaltyDeath()는 violation을 감지하면 앱을 종료한다.
  - penaltyListener()는 violation 감지할 때마다 호출되는 custom listener를 추가할 수 있다.
- policy에서 penalty를 원하는 대로 조합해 적용할 수 있다.
  - 명시하지 않으면 기본값 penaltyLog()가 적용된다.
  - penaltyLog() 이외의 penalty를 적용하는 경우 penaltyLog()를 명시적으로 설정하지 않으면 penaltyLog() 사용이 중지된다.
- penaltyListener()는 violation을 logging할 third-party logging library가 있으면 유용하다.
- 또는, release builds에서 non-critical violation catching을 설정하고 crash reporting library에 logging할 수도 있다.

```kotlin
class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        FragmentStrictMode.defaultPolicy =
            FragmentStrictMode.Policy.Builder()
                .detectFragmentReuse()
                .detectFragmentTagUsage()
                .detectRetainInstanceUsage()
                .detectSetUserVisibleHint()
                .detectTargetFragmentUsage()
                .detectWrongFragmentContainer()
                .apply {
                    if (BuildConfig.DEBUG) {
                        // Fail early on DEBUG builds
                        penaltyDeath()
                    } else {
                        // Log to Crashlytics on RELEASE builds
                        penaltyListener {
                            FirebaseCrashlytics.getInstance().recordException(it)
                        }
                    }
                }
                .build()
    }
}
```
- 위 예시는 global StrictMode policy를 설정한 예시
  - FragmentStrictMode.setDefaultPolicy()를 통해 모든 FragmentManager instance에 default policy를 설정한다.
#### Fragment reuse
- fragment reuse violation은 detectFragmentReuse()를 사용할 수 있고 FragmentReuseViolation을 던진다.
- 이는 FragmentManager에서 제거된 Fragment instance의 reuse를 나타낸다.
- 이런 reuse는 Fragment가 이전 state을 유지하며 동작이 일관되지 않을 수 있기 때문에 문제가 발생한다.
- 매번 새로운 instance를 생성하는 경우 FragmentManager에 instance가 추가될 때 항상 초기 상태가 된다.

#### Fragment tag usage
- fragment tag usage violation은 detectFragmentTagUsage()를 사용할 수 있고 FragmentTagUsageViolation을 던진다.
- 이는 XMl layout에서 < fragment > tag를 사용해 Fragment를 inflate할 때를 나타내며 FragmentContainerView를 사용하여 해결할 수 있다.
  - FragmentContainerView를 사용하면 Fragment transactions, configuration change를 안정적으로 관리할 수 있다.

#### Retain instance usage
- retain instance usage violation은 detectRetainInstanceUsage()를 사용할 수 있고 RetainInstanceUsageViolation을 던진다.
- retained Fragment 사용을 나타내며 특히 deprecated된 setRetainInstance(), getRetainInstance()를 호출할 때 나타난다.
- retained Fragment instance를 관리할 때에는 ViewModel을 사용하여 state을 저장해야 한다.

#### Set user visible hint
- user visible hint violation은 detectSetUserVisibleHint()를 사용할 수 있고 SetUserVisibleHintViolation를 던진다.
- 이는 deprecated된 setUserVisibleHint()가 call될 때를 나타낸다.
  - 만약 이 method를 call하려면 대신 setMaxLifecycle()를 call해야 한다.
- 만약 setMaxLifecycle()을 override했다면<sup id="re">[e)](#fe)</sup>
  - 만약 setUserVisibleHint()가 true면 behavior를 onResume()으로 옮겨야 한다.
  - 만약 setUserVisibleHint()를 false면 behavior를 onPause()으로 옮겨야 한다.

#### Target fragment usage
- target fragment usage violation은 detectTargetFragmentUsage()를 사용할 수 있고 TargetFragmentUsageViolation를 던진다.
- deprecated된 setTargetFragment(), getTargetFragment(), getTargetRequestCode()를 사용할 때 나타나고 대신 FragmentResultListener를 register해야 한다.

#### Wrong fragment container
- wrong fragment container violation은 detectWrongFragmentContainer()를 사용할 수 있고 WrongFragmentContainerViolation를 던진다.
- 이는 Fragment를 FragmentContainerView가 아닌 다른 container에 add할 때 나타난다.
  - Fragment transaction은 FragmentContainerView 내에서 hosting하지 않으면 제대로 동작하지 않을 수도 있다.
  - 또한, exit animation을 사용하는 fragment를 다른 모든 fragment 위에 그리는 View API 문제 해결에 도움이 된다.

## Test your fragments
- fragments는 reusable container로써 다양한 activities와 layout configurations에서 같은 user interface layout을 제공할 수 있다.
- 용도가 다양한 fragment를 고려하면 fragment가 일관되고 resource-efficient한 환경을 제공하는지 검증하는 것은 중요하다.
  - fragment는 특정 parent activity or fragment에 의존적이면 안된다.
  - fragment가 user에게 visible하지 않으면 fragment의 view hierarhy를 create해서는 안된다.
- 위와 같은 조건을 테스트하기 위해 AndroidX fragment-testing library는 FragmentScenario class를 제공하여 fragment를 create하고 Lifecycle.State를 변경한다.

### [Declaring dependencies]
```Gradle
dependencies {
    val fragment_version = "1.4.1"

    debugImplementation("androidx.fragment:fragment-testing:$fragment_version")
}
```
- FragmentScenario를 사용하기 위해 다음과 같이 debugImplementation을 사용하여 정의한다.

### [Create a fragment]
- FragmentScenario는 다음의 2가지 방법으로 test에서 fragment를 launch한다.
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEventFragment() {
        // The "fragmentArgs" argument is optional.
        val fragmentArgs = bundleOf(“selectedListItem” to 0)
        val scenario = launchFragmentInContainer<EventFragment>(fragmentArgs)
        ...
    }
}
```
- launchInContainer는 fragment의 user interface를 테스트한다.
- FragmentScenario는 activity의 root view controller한테 fragment를 attach한다.
  - 이 activity는 그 외의 경우는 empty다.
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEventFragment() {
        // The "fragmentArgs" arguments are optional.
        val fragmentArgs = bundleOf("numElements" to 0)
        val scenario = launchFragment<EventFragment>(fragmentArgs)
        ...
    }
}
```
- launch()는 fragment의 user interface 없이 테스트한다.
- FragmentScenario는 fragment를 root view가 없는 empty activity에 attach한다.<br><br>
- 이렇게 2가지 방법으로 launch한 후 FragmentScenario는 기본적으로 fragment를 RESUMED state으로 만든다.
  - initialState argument를 통해 이를 override할 수도 있다.

### [Provide dependencies]
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEventFragment() {
        val someDependency = TestDependency()
        launchFragmentInContainer {
            EventFragment(someDependency)
        }
        ...
    }
}
```
- 만약 fragment가 dependencies를 갖고 있다면 launchInContainer() or launch() method에 custom FragmentFactory를 제공하여 이 dependencies의 test version을 제공할 수 있다.

### [Drive the fragment to a new state]
- app의 UI test를 할 때는 RESUMED state에서 해도 충분하지만 lifecycle state이 다른 state으로 transition되는 동작에 대해서 평가를 해야 할수도 있다.
  - initialState argument를 모든 launchFragment*() method에 전달하여 초기 상태를 지정할 수 있다.
- fragment를 다른 lifecycle state으로 이동시키려면 moveToState()를 호출한다.
  - CREATED, STARTED, RESUMED, DESTORYED argument를 지원하며 fragment를 contain하는 fragment or activity가 해당 state으로 변하는 상황을 simulate한다.
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEventFragment() {
        val scenario = launchFragmentInContainer<EventFragment>(
            initialState = Lifecycle.State.INITIALIZED
        )
        // EventFragment has gone through onAttach(), but not onCreate().
        // Verify the initial state.
        scenario.moveToState(Lifecycle.State.RESUMED)
        // EventFragment moves to CREATED -> STARTED -> RESUMED.
        ...
    }
}
```
### [Recreate the fragment]
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEventFragment() {
        val scenario = launchFragmentInContainer<EventFragment>()
        scenario.recreate()
        ...
    }
}
```
- 만약 app을 실행중인 device가 resource가 부족하게 되면 fragment를 contain하는 activity를 destroy시킬 수도 있다.
  - 이런 상황에서 app은 fragment를 recreate하여 return하고 이런 상황을 recreate()를 통해 simulate할 수 있다.
  - FragmentScenario.recreate()는 fragment를 destroy하고 host와 fragment를 recreate한다.
  - recreate하면 destroy전의 lifecycle state으로 return된다.

### [Interacting with UI fragments]
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEventFragment() {
        val scenario = launchFragmentInContainer<EventFragment>()
        onView(withId(R.id.refresh)).perform(click())
        // Assert some expected behavior
        ...
    }
}
```
- test하의 fragment에서 UI action을 일으키려면 Espresso view matchers를 이용한다.
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testEventFragment() {
        val scenario = launchFragmentInContainer<EventFragment>()
        scenario.onFragment { fragment ->
            fragment.myInstanceMethod()
        }
    }
}
```
- 만약 fragment 자체 method를 호출하려면 FragmentScenario.onFragment()를 사용하여 FragmentAction을 전달한다.

### [Test dialog actions]
```kotlin
@RunWith(AndroidJUnit4::class)
class MyTestSuite {
    @Test fun testDismissDialogFragment() {
        // Assumes that "MyDialogFragment" extends the DialogFragment class.
        with(launchFragment<MyDialogFragment>()) {
            onFragment { fragment ->
                assertThat(fragment.dialog).isNotNull()
                assertThat(fragment.requireDialog().isShowing).isTrue()
                fragment.dismiss()
                fragment.parentFragmentManager.executePendingTransactions()
                assertThat(fragment.dialog).isNull()
            }
        }

        // Assumes that the dialog had a button
        // containing the text "Cancel".
        onView(withText("Cancel")).check(doesNotExist())
    }
}
```
- FragmentScenario는 dialog fragments testing도 지원한다.
- dialog fragments는 UI elements를 갖고 있지만 그 layout은 layout 자체가 아닌 별도의 window에 채워지기 때문에 FragmentScenario.launch()를 사용하여 테스트한다.









## Q&A
<b id="f1">1) </b>같은 key에 대해 setFragmentResult()를 두 번 이상한 경우 + listener.state != STARTED [↩](#r1)<br>
- setFragmentResult()를 두 번 이상 했는데 listener state가 STARTED가 아니면 listener callback이 execute하지 않았을 것이다.
- 그렇게 되면 원래 갖고 있던 result, pending result는 가장 최근에 setFragmentResult()를 해서 pass하는 data, updated data로 replace된다는 뜻


<b id="f2">2) </b>fragment에서 DialogFragment 사용 시 왜 child FragmentManager를 사용해야 할까?[↩](#r2)<br>
- DialogFragment도 결국 Fragment
- activity에서는 supportFragmentManager를 사용
- fragment에서 DialogFragment를 생성하는 것은 결국 child fragment를 생성하는 것
  - 그래서 child FragmentManager 사용

<b id="f3">3) </b> DialogFragment에서의 onCreateView(), onViewCreated()[↩](#r3)<br>
- DialogFragment를 Dialog와 사용한다는 것은 onCreateDialog()를 통해 Dialog를 return 하는 경우
  - Dialog는 views 뿐 아니라 own window를 가진다.
    - 그렇기에 view를 return하고 그 view를 받는 onCreateView(), onViewCreated()를 override하지 않는다.
  - 게다가 만약 custom DialogFragment를 사용한다는 것은 layout file를 사용하여 view를 inflate시킨다는 것이고 이때는 onCreateView()를 override하고 non-null view를 제공해서 onViewCreated()가 받도록 해야 한다.
    - onCreateView()를 override하고 non-null view를 제공하지 않으면 onViewCreated()가 호출되지 않는다.



## 추가 Q&A
<b id="fa">a) </b> NonConfig의 경우 ViewModel의 saved state module을 사용하면 process death and recreation에서도 유지된다.[↩](#ra)<br>
- ViewModel saved state module을 사용한다는 것은 SavedStateHandle을 사용하는 것
- 기존 ViewModel의 경우 state을 memory에 저장
  - 그렇기에 process death되면 전부 destroy된다.
    - process는 memory에 load되어 실행중인 program이고 모든 요소들을 갖고 있기에 process death 된다는 것은 memory에서 제거되는 것이고 ViewModel은 memory에 저장되어 있기에 destroy
- SavedStatehandle을 사용하는 것은 Saved Instance state을 사용하는 것으로 disk에 저장
  - 그래서 process death and recreation에서도 restore된다.
- 또한 NonConfig는 server or local respository와 같은 외부 소스에서 가져오는 데이터인데 이는 결국 ViewModel-Repository-Datasource를 통해서 받아와서 UI state에 저장하는 데이터들이기에 NonConfig를 ViewModel을 사용

<b id="fb">b) </b> Dialog의 dismiss와 cancle [↩](#rb)<br>
- dismiss는 Dialog를 안전하게 종료할 때 사용하는 것으로 Dialog button에 의해서 Dialog를 종료시키는 경우
- cancel은 back button을 눌렀을 때 Dialog를 종료시키는 경우
  - back button에 의해 종료되지 않으려면 setCancelable(false)를 사용
  - onCancleListener가 register 되어 있다면 call된 후 Dailog dismiss 호출

<b id="fc">c) </b> fragmentmanager strictmode를 super.onCreate() 이전에 설정[↩](#rc)<br>
- fragmentmanager strictmode를 설정하는 가장 마지마 지점은 onCreate() 내의 super.onCreate() 이전
- process recreate 될 때 super.onCreate(SavedInstanceState)를 통해 fragmentmanager가 fragment를 restore
  - 그렇기 때문에 그 이전에 fragmentmanager strictmode를 설정해야 한다.

<b id="fd">d) </b> allowViolation과 StrictMode violation을 contain한 third-party library[↩](#rd)<br>
- allowViolation()의 역할
  - Allow the specified [Fragment] class to bypass penalties for the specified * [Violation], if detected.
  - 즉, 지정한 Fragment는 해당 Violation을 허용하는 것
- 그래서 위와 같은 기능을 사용하면 StrictMode violation을 contain한 third-party library에서 violation을 고치기 전까지 해당 violtion에 대해 allow list에 임시적으로 추가하여 사용할 수 있다.

<b id="fe">e) </b> setUserVisibleHint와 setMaxLifecycle[↩](#re)<br>
- setUserVisibleHint(true) 대신 setMaxLifecycle(fragment, Lifecycle.State.RESUMED) 사용
  - onResume()은 visible fragment일 때만 호출
- setuserVisibleHint(false) 대신 setMaxLifecycle(fragment, Lifecycle.State.STARTED) 사용


