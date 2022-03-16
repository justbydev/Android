# Fragments
## Saving state with fragments
- 다양한 Android system operations는 fragment state에 영향을 준다.
- user state가 save되는 것이 보장되려면 Android framework는 fragment와 back stack을 자동으로 save하고 restore해야 한다.
<img width="777" alt="스크린샷 2022-03-16 오후 1 52 12" src="https://user-images.githubusercontent.com/17876424/158519069-e6063407-051a-4cba-b883-fb8531f9562c.png">


- Variables: fragment local variables
- View State: fragment의 하나 이상의 view가 가진 data
- SavedState: onSaveInstanceState()에 저장되어야 하는 fragment instance의 고유한 data
- NonConfig: server or local repository와 같은 외부 소스에서 가져온 data 혹은 commit된 후에 server에게 전송된 user-created data

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
  - 만약 SavedState module을 ViewModel에 추가하면 process death and recreation 상황에서 유지된다.

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
- 만약 Navigation library를 사용한다면 ViewModel을 destination의 NavBackStackEntry lifecycle에 scope하 수 있다.

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
- parent는 fragment가 STARTED일 때 result를 받는다.
```kotlin
button.setOnClickListener {
    val result = "result"
    // Use the Kotlin extension in the fragment-ktx artifact
    setFragmentResult("requestKey", bundleOf("bundleKey" to result))
    //getParentFragmenetManager().setFragmentResult()
}
```
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



## Q&A
<b id="f1">1) </b>같은 key에 대해 setFragmentResult()를 두 번 이상한 경우 + listener.state != STARTED [↩](#r1)<br>
- setFragmentResult()를 두 번 이상 했는데 listener state가 STARTED가 아니면 listener callback이 execute하지 않았을 것이다.
- 그렇게 되면 원래 갖고 있던 result, pending result는 가장 최근에 setFragmentResult()를 해서 pass하는 data, updated data로 replace된다는 뜻


