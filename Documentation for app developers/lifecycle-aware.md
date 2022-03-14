# Lifecycle-aware components(Lifecycles, ViewModel, LiveData)
## 참고 자료
[Android Developers: Lifecycle-aware components](https://developer.android.com/topic/libraries/architecture/lifecycle)

## Handle lifecycles
- Lifecycle-aware components는 activities, fragments와 같은 component의 lifecycle status 변화에 반응하여 동작한다.
- Lifecycle-aware components를 사용하면 lifecycle method에 종속된 code를 components 그자체 코드로 옮길 수 있다.
- androidx.lifecycle package에서 제공하는 classes, interfaces를 사용하여 lifecycle-aware components를 build하면 자동으로 현재 lifecycle status에 맞게 behavior를 조정할 수 있다.
- Android Framework에 정의된 대부분의 app component는 lifecycle를 갖고 있다.
  - 이 lifecycle을 고려하지 않으면 memory leak, application crash도 발생할 수 있다.
```kotlin
class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this) { location ->
            // update UI
        }
    }

    public override fun onStart() {
        super.onStart()
        Util.checkUserStatus { result ->
            // what if this callback is invoked AFTER activity is stopped?
            if (result) {
                myLocationListener.start()
            }
        }
    }

    public override fun onStop() {
        super.onStop()
        myLocationListener.stop()
    }

}
```
- onStart()에서 locationlistener 설정 전 long-running operation이 이루어져서 onStart() 작업이 끝나기 전에 onStop()이 끝날 수 있다.
  - androidx.lifecycle package에서 제공하는 classes, interfaces가 이런 문제를 해결해 줄 수 있다.

### [Lifecycle]
<img width="734" alt="스크린샷 2022-03-07 오후 1 32 03" src="https://user-images.githubusercontent.com/17876424/156968455-372457c3-9566-4355-ade5-089eaa8cf323.png">

- Lifecycle은 activity, fragment와 같은 component의 lifecycle state information을 가진 class
- Event와 State를 가진다.
  - Event는 Framework 및 Lifecycle class에서 전달되는 lifecycle events로 activities, fragments의 callback events와 mapping된다.
  - State는 Lifecycle object에 의해 track되는 component의 current state
  - Lifecycle class 내에 Event, State enum class로 정의되어 있다.
- DefaultLifecycleObserver를 implement한 class가 component's lifecycle status를 monitor하고 corresponding method를 override한다.
  - DefaultLifecycleObserver에는 events에 맞는 method(onCreate, onStart 등)이 overried되어 있고 해당하는 event가 발생했음을 notify한다.
- Lifecycle class의 method인 addObserver()를 통해 observer instance를 passing한다.
  - Lifecycle class는 LifecycleOwner를 implement한 object 통해 Lifecycle class를 얻는다.
```kotlin
class MyObserver : DefaultLifecycleObserver {
    override fun onResume(owner: LifecycleOwner) {
        connect()
    }

    override fun onPause(owner: LifecycleOwner) {
        disconnect()
    }
}

myLifecycleOwner.getLifecycle().addObserver(MyObserver())
```
### [LifecycleOwner]
- Lifecycle를 받을 수 있는 getLifecycle() method를 가진 single method interface
  - 만약 whole applicationn process의 lifecycle를 manage하고 싶으면 ProcessLifecycleOwner를 사용할 수 있다.
- LifecycleOwner는 Fragment, AppcompatActivity와 같은 class가 Lifecycle을 갖고 있음을 나타낸다.
  - custom application class가 LifecycleOwner interface를 implement할 수 있다.
- DefaultLifecycleObserver를 implement한 component는 LifecycleOwner를 implement한 component와 원활하게 작동한다.
  - LifecycleObserver가 관찰을 위해 등록하는 Lifecycle를 LifecycleOwner가 제공하기 때문이다.
- 위에서 나왔던 locationlistener의 경우 DefaultLifecycleObserver를 implement하고 이 listener를 사용하는 component가 lifecycle를 전달한다.
  - 이 lifecycle의 state를 감지하여 자체적으로 감지하여 작동하 수 있도록 하여 lifecycle-aware하게 만든다.
```kotlin
internal class MyLocationListener(
        private val context: Context,
        private val lifecycle: Lifecycle,
        private val callback: (Location) -> Unit
): DefaultLifecycleObserver {

    private var enabled = false

    override fun onStart(owner: LifecycleOwner) {
        if (enabled) {
            // connect
        }
    }

    fun enable() {
        enabled = true
        if (lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
            // connect if not connected
        }
    }

    override fun onStop(owner: LifecycleOwner) {
        // disconnect if connected
    }
}
```
```kotlin
class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this, lifecycle) { location ->
            // update UI
        }
        Util.checkUserStatus { result ->
            if (result) {
                myLocationListener.enable()
            }
        }
    }
}
```
#### Implementing a custom LifecycleOwner
- Support Library 26.1.0 이후부터는 Fragments, Activities는 이미 LifecycleOwner를 implement한다.<sup id="r1">[1)](#f1)</sup>
  - Activities는 ComponentActivity가, Fragments는 Fragment가 LifecycleOwner를 implement
  - getLifecycle()에 대한 실제 구현도 되어 있고 Lifecycle instance를 얻을 수 있다.
  - LifecycleRegistry<sup id="r2">[2)](#f2)</sup>를 return
- LifecycleOwner를 직접 implement한 custom class를 갖고 있다면 LifecycleRegistry class를 사용할 수 있다.
  - 단, 이 class에 event를 전달해야 한다.
```kotlin
class MyActivity : Activity(), LifecycleOwner {

    private lateinit var lifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleRegistry = LifecycleRegistry(this)
        lifecycleRegistry.markState(Lifecycle.State.CREATED)
    }

    public override fun onStart() {
        super.onStart()
        lifecycleRegistry.markState(Lifecycle.State.STARTED)
    }

    override fun getLifecycle(): Lifecycle {
        return lifecycleRegistry
    }
}
```
### [Best practices for lifecycle-aware components]
- UI controller(activities and fragments)를 가볍게 유지해야 한다. 자체 데이터를 확보하기 보다는 ViewModel를 사용하고 LiveData 객체를 observe하여 변화를 views에 적용해야 한다.
- data-driven UI를 작성해야 한다. 데이터 변경에 따라 View를 업데이트하거나 user actions를 Viewmodel에게 전달하는 것은 UI controller의 역할이다.
- data logic은 Viewmodel class에 배치한다. Viewmodel은 UI controller와 rest of app의 connector 역할이다.
  - 하지만 Viewmodel에 fetch data logic을 갖고 있으면 안되며 다른 component에서 data를 fetch하면 그 데이터를 받아 다시 UI controller에게 제공하게 된다.
- Data binding을 사용하여 view와 UI controller 사이의 인터페이스를 깨끗하게 하자. view를 declarative하게 하고 activities, fragments에 작성하는 view update code를 minimize 시킬 수 있다.
- UI가 복잡하다면 UI modifications를 처리할 presenter class를 생성하는 것을 고려하자.
- Viewmodel에서 View, Activity의 context를 reference하는 것을 avoid해라.
  - 만약 Viewmodel이 activity보다 outlive하면(configuration change와 같은 경우) activity가 leak되고 garbage collector에 의해 제대로 관리가 되지 않는다.
  - long-running task나 비동기 작업의 경우 Kotlin coroutines를 사용하자.

### [Use cases for lifecycle-aware components]
- 위치 기반 어플리케이션 : app이 visible하면 fine-grained location을 사용하고 background로 가면 coarse-grained로 switch된다.
  - LiveData를 통해 user location 변경에 따른 UI update가 자동으로 이루어진다.
- video buffering이 최대한 빨리 시작하지만 app이 완전히 시작될때까지 연기한다.
  - app이 destroy되면 buffering을 종료한다.
- app이 foreground에 있으면 네트워크 데이터를 실시간 update하고 background가 되면 자동으로 중지할 수 있다.
- app background에 있으면 animated drawables를 중지하고 foreground로 오면 다시 시작한다.

## Viewmodel
- Viewmodel class는 lifecycle를 고려하여 UI-related data를 store and manage하며 configuration change에도 data가 살아남도록 해준다.
- user actions나 device events에 의해 UI controllers(activities and fragments)가 destroy되거나 re-create되고 그 과정에서 transient UI-related data가 lost된다.
  - simple data의 경우 onSaveInstanceState()를 통해 저장하고 onCreate의 bundle을 통해 restore할 수 있지만 large data는 할 수 없다.
- 시간을 소모하는 asynchronous calls를 필요로 하는 경우도 있는데 UI controller destroy될 때 memory leak을 막기 위해 clean up되어야 한다.
- 이와 같은 작업들을 비롯하여 data-related logic을 UI controllers에서 다루게 되면 excessive responsibility가 부과되어 test하기 힘들어진다.
  - 따라서 이런 logic이나 view data ownership을 UI controller로부터 seperate시키는 것이 좋다.

### [Implement a ViewModel]
```kotlin
class MyViewModel : ViewModel() {
    private val users: MutableLiveData<List<User>> by lazy {
        MutableLiveData<List<User>>().also {
            loadUsers()
        }
    }

    fun getUsers(): LiveData<List<User>> {
        return users
    }

    private fun loadUsers() {
        // Do an asynchronous operation to fetch users.
    }
}
```
```kotlin
class MyActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.

        // Use the 'by viewModels()' Kotlin property delegate
        // from the activity-ktx artifact
        val model: MyViewModel by viewModels()
        // val model = ViewModelProvider(this@MyActivity).get(MyViewModel::class.java)
        model.getUsers().observe(this, Observer<List<User>>{ users ->
            // update UI
        })
    }
}
```
- UI를 위한 data를 ViewModel object에게 맡기고 configuration changes가 일어나도 data를 자동으로 유지한다.
- 만약 activity가 re-created되면 처음 activity가 create한 같은 MyViewModel instance를 받게 된다.
- owner activity가 finished되면 ViewModel object의 onCleared()<sup id="r4">[4)](#f4)</sup> method가 호출되어 resources가 clean up된다.
- ViewModel objects는 views나 LifecycleOwners를 outlive하도록 design되어 있다.
- 또한 LiveData를 contain할 수 있다.
  - 단, Viewmodel object는 LiveData를 observe하면 안된다.

### [The lifecycle of a ViewModel]
<img width="434" alt="스크린샷 2022-03-07 오후 5 34 53" src="https://user-images.githubusercontent.com/17876424/156996040-83604c8b-16ff-40c2-acd3-36500f31ef6a.png">


- Viewmodel를 얻을 때 ViewModelProvider<sup id="r5">[5)](#f5)</sup>에게 전달되는 Lifecycle에 Viewmodel object가 scope 된다.
  - scope된 Lifecycle이 끝나기 전까지 ViewModel은 memory에 남아있게 된다.
  - Activity라면 finished 될때까지, Fragment라면 detached 될때까지
- Viewmodel은 처음에 activity object의 onCreate()에서 request한다.
  - 만약 configuration change에 의해 onCreate()가 여러번 호출되면 activity가 finished되고 destroyed되기 전까 처음 request했던 ViewModel이 계속 존재하게 된다.


### [Share data between fragments]
- 같은 activity에 attach된 여러 fragment들의 communicate는 Viewmodel을 통해 할 수 있다.
- activity scope<sup id="r6">[6)](#f6)</sup>를 사용하는 ViewModel를 share하여 communication을 handle할 수 있다.
```kotlin
class SharedViewModel : ViewModel() {
    val selected = MutableLiveData<Item>()

    fun select(item: Item) {
        selected.value = item
    }
}

class ListFragment : Fragment() {

    private lateinit var itemSelector: Selector

    // Use the 'by activityViewModels()' Kotlin property delegate
    // from the fragment-ktx artifact
    private val model: SharedViewModel by activityViewModels()
    //val model = ViewModelProvider(requireActivity()).get(SharedViewModel::java.class)
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        itemSelector.setOnClickListener { item ->
            // Update the UI
        }
    }
}

class DetailFragment : Fragment() {

    // Use the 'by activityViewModels()' Kotlin property delegate
    // from the fragment-ktx artifact
    private val model: SharedViewModel by activityViewModels()
    //val model = ViewModelProvider(requireActivity()).get(SharedViewModel::java.class)
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        model.selected.observe(viewLifecycleOwner, Observer<Item> { item ->
            // Update the UI
        })
    }
}
```
- 위와 같이 작성하면 ViewModelProvider를 통해서 activity scope를 사용하는 같은 SharedViewModel instance를 받게 된다.
- 이와 같이 사용하면 다음과 같은 장점이 있다.
  - activity에서 부가적인 작업을 할 필요 없다.
  - Fragments는 서로에 대해 알 필요 없이 ViewModel를 통해서만 communicate 하면 된다.
    - 하나의 Fragment가 사라져도 다른 Fragment는 정상적으로 작동된다.
  - 각각의 Fragment는 각자의 lifecycle를 갖고 있는데 서로의 lifecycle에 영향을 받지 않는다.


### [Replacing Loaders with ViewModel]
- CursorLoader와 같은 Loader class는 app UI data와 database 간의 동기화를 유지하는데 자주 사용된다.
- 이를 ViewModel이 대체할 수 있다.
  - UI controller가 data-loading operation과 분리된다.
- CursorLoader를 사용하면 CursorLoader가 database content를 observe하고 database가 변하면 data를 reload하고 UI를 update한다.
- Viewmodel은 Room, LiveData를 통해 대체한다.
  - database가 바뀌면 Room이 LiveData에게 알리고 LiveData를 observe하여 UI를 update한다.


## LiveData
- LiveData는 observable data holder class다.
- LiveData는 app component의 lifecycle를 인식하는 lifecycle-aware다.
  - LiveData는 active lifecycle state일 때만 update 되도록 한다.
- LiveData는 Observer class를 통해 구현되는 observer를 고려하여 lifecycle이 STARTED, RESUMED state일 때 observer가 active하도록 한다.
  - inactive observer일 때는 LiveData objects의 변화를 감지하지 못한다.
- observer는 LifecycleOwner interface를 implement한 object와 함께 등록할 수 있다.
  - 따라서 Lifecycle object가 DESTROYED일 때 observer도 remove된다.<sup id="r7">[7)](#f7)</sup>
  - activities, fragments는 LiveData object를 안전하게 observe하고 그들의 lifecycles가 destroyed되면 그 즉시 unsubscribed된다.

### [The advantages of using LiveData]
#### UI와 data state의 일치 보장
- Observer pattern을 따른다.
  - data가 변경될 때 Observer object에게 알린다.
  - app data가 변할 때 UI update를 observer가 해준다.
#### No memory leaks
- Lifecycle objects에 bound되어 lifecycle이 destroyed될 때 clean up된다.
#### stopped activities에 의한 crash가 없다.
- observer's lifeclce이 inactive하면 LiveData events를 받지 않는다.
#### lifecycle을 더 이상 수동으로 handling하지 않는다.
- Livedata가 lifecycle status를 알고 자동으로 관리한다.
#### 항상 최신 데이터를 가진다.
- lifecycle이 inactive였다가 active되면 바로 최신 데이터를 받는다.
#### configuration changes
- configuration changes가 발생하여 activity, fragment가 recreate되면 바로 최신 데이터를 받는다.
#### resources 공유
- system service를 공유할 수 있도록 singleton pattern 사용하는 LiveData object extend하여 사용할 수 있다.
  - LiveData가 system service에 한번 연결되면 resource를 필요로 하는 모든 observer가 LiveData object를 관찰할 수 있다.

### [Work with LiveData objects]
- 다음의 과정으로 LiveData object를 생성한다.
1) LiveData는 certain type of data를 가지며 주로 ViewModel class에 있다.
2) Observer object를 생성하고 LiveData가 가진 data가 변할 때 어떤 작업을 할지 정의하는 onChanged() method를 정의한다. 주로 activity, fragment의 UI controller에 생성한다.
3) observe() method를 통해 LiveData object에 Observer object를 attach한다. 또한, LifecycleOwner object도 take한다. 이를 통해 LiveData object를 Observer object가 subscribe하고 변화를 감지한다.

- 이와 같이 구성하면 attach된 LifecycleOwner가 active state라면 LiveData object가 hold한 data가 변하면 register된 모든 observers가 trigger된다.
- LifecycleOwner object 없이 항상 active 상태이고 싶다면 observeForever(Observer)를 사용하고 remove 시 removeObserver(Observer)를 사용한다.
#### Create LiveData objects
```kotlin
class NameViewModel : ViewModel() {

    // Create a LiveData with a String
    val currentName: MutableLiveData<String> by lazy {
        MutableLiveData<String>()
    }

    // Rest of the ViewModel...
}
```
- LiveData는 Collections를 비롯한 어떤 data로 hold할 수 있는 wrapper다.
- LiveData를 activity, fragment가 아닌 ViewModel object에 두는 이유는 다음과 같다.
  - UI controllers는 오직 data를 display하는 책임만 가지도록 한다.
  - LiveData가 특정 activity, fragment에서만 coupling되도록 하지 않고 configuration change에도 살아남을 수 있도록 한다.

#### Observe LiveData objects
```kotlin
class NameActivity : AppCompatActivity() {

    // Use the 'by viewModels()' Kotlin property delegate
    // from the activity-ktx artifact
    private val model: NameViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Other code to setup the activity...

        // Create the observer which updates the UI.
        val nameObserver = Observer<String> { newName ->
            // onChanged()
            // Update the UI, in this case, a TextView.
            nameTextView.text = newName
        }

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        model.currentName.observe(this, nameObserver)
    }
}
```
- 대부분의 경우 LiveData observing은 app component의 onCreate()에서 시작된다.
  - onCreate()에서 시작하기 때문에 onResume()에서의 불필요한 call을 만들지 않는다.
  - app component가 STARTED state가 되자마자, 즉, active 되자마자 바로 LiveData를 observe할 수 있도록 한다.
- LiveData는 data change 시 오직 active observer에게 update를 전달한다.
- inactive에서 active state이 될 때 가장 마지막 값과 다를 때 update를 전달한다.
  - update되면 Observer의 onChanged()를 invoke한다.

#### Update LiveData objects
- LiveData는 stored data를 update하는 method가 없고 MutableLiveData class를 사용해야 한다.
  - 보통 MutableLiveData는 ViewModel에서 사용하고 immutable LiveData를 observer에게 expose한다.
- setValue(T), postValue(T)를 사용한다.<sup id="r8">[8)](#f8)</sup>
  - setValue(T)는 main thread에서, postValue(T)는 worker thread에서 사용한다.
  - update하면 observer가 onChanged() method를 call한다.

#### Use LiveData with Room
- Room library는 observable queries를 지원하고 LiveData objects를 return한다.
- Room은 database가 update될 때 LiveData object를 update하는 코드를 생성한다.
- 필요할 때 background thread에서 비동기적으로 query를 run한다.

#### Use coroutines with Livedata
- LiveData는 Kotlin coroutines도 support한다.

### [LiveData in an app's architecture]
- LiveData는 lifecycle owner와 ViewModel objects lifespan 사이에서 communicate한다.
- LiveData는 ViewModel class에 정의하고 activities, fragments가 hold하도록 하지 않는다.
  - activities, fragments는 오직 data를 display하는 역할만 해야 한다.
- LiveData를 data layer classes에서 hold하지 않아야 한다.
  - LiveData는 비동기 데이터 흐름을 handle하기 위해 design되지 않았다.
  - data stream을 combine할 능력이 제한적이다.
  - 모든 LiveData objects는 main thread에서 observe 된다.<sup id="r9">[9)](#f9)</sup>
  - 만약 ViewModel class가 아닌 다른 layer에서 사용하고 싶다면 Kotlin Flows를 사용하고 ViewModel에서 asLiveData()를 사용하여 변환하는 것을 고려하자.

### [Extend LiveData]
```kotlin
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager = StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }
}
```
```kotlin
public class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        val myPriceListener: LiveData<BigDecimal> = ...
        myPriceListener.observe(viewLifecycleOwner, Observer<BigDecimal> { price: BigDecimal? ->
            // Update the UI.
        })
    }
}
```
- onActive()의 경우 LiveData object가 active observer를 가질 때 호출된다.
  - active state이기에 여기서 stock price를 update한다.
- onInactive()는 LiveData object가 active observer르 가지지 않을 때 호출된다.
  - observer가 inactive하기 때문에 StockManager service를 remove한다.
- setValue(T)를 통해 update하고 active observer가 변화를 감지한다.
- fragment's view의 Lifecycleowner를 pass하여 fragment view lifecycle이 active state일 때만 observe하도록 한다.
- LiveData가 lifecycle-aware하다는 것은 multiple activities, fragments, services끼리 공유할 수 있다는 뜻이므로 LiveData를 singleton으로 사용할 수도 있다.
```kotlin
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager: StockManager = StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }

    companion object {
        private lateinit var sInstance: StockLiveData

        @MainThread
        fun get(symbol: String): StockLiveData {
            sInstance = if (::sInstance.isInitialized) sInstance else StockLiveData(symbol)
            return sInstance
        }
    }
}
```
### [Transform LiveData]
- observer에게 보내기 전에 객체에 저장된 값을 변경하고 싶거나 다른 객체의 값에 따라 다른 LiveData instance를 반환해야 하는 경우가 있다.
  - 이때, Transformations class를 사용한다.
```kotlin
val userLiveData: LiveData<User> = UserLiveData()
val userName: LiveData<String> = Transformations.map(userLiveData) {
    user -> "${user.name} ${user.lastName}"
}
```
```kotlin
private fun getUser(id: String): LiveData<User> {
  ...
}
val userId: LiveData<String> = ...
val user = Transformations.switchMap(userId) { id -> getUser(id) }
```
- 다른 LiveData의 변화를 감지하게 된다.
- switchMap의 경우 map과 달리 람다함수가 LiveData object를 return한다.
- transformation method를 사용하여 observer's lifecycle 전반에 걸쳐 정보를 전달할 수 있다.<sup id="r10">[10)](#f10)</sup>
- return된 LiveData object를 observe하지 않다면 transformations은 calculate되지 않는다.
  - transformations은 늦게 calculate되기 때문에 lifecycle-related behavior는 implicitly하게 pass down된다.

- 만약 ViewModel object안에서 Lifecycle object를 필요로 한다면 transformations를 사용하는 것이 좋다.
```kotlin
class MyViewModel(private val repository: PostalCodeRepository) : ViewModel() {

    private fun getPostalCode(address: String): LiveData<String> {
        // DON'T DO THIS
        return repository.getPostCode(address)
    }
}
```
- 위와 같이 작성하면 UI component가 address를 받을 때마다 postal code를 return하게 된다.
  - getPostalCode()를 호출할 때마다 이전 LiveData object를 unregister하고 새로운 LiveData object를 register한다.<sup id="r11">[11)](#f11)</sup>
```kotlin
class MyViewModel(private val repository: PostalCodeRepository) : ViewModel() {
    private val addressInput = MutableLiveData<String>()
    val postalCode: LiveData<String> = Transformations.switchMap(addressInput) {
            address -> repository.getPostCode(address) }


    private fun setInput(address: String) {
        addressInput.value = address
    }
}
```
- postalCode와 연관된 observer가 active라면 addressInput이 변할 때마다 postalCode가 다시 계산된다.
- 이런 machanism을 사용하면 앱의 하위 수준(RoomDB 등)에서 요청이 있을 때 lazily 계산되는 LiveData object를 생성할 수 있다.
  - ViewModel object는 LiveData object 참조를 쉽게 얻어 그 위에 transformation rule을 쉽게 적용할 수 있다.

#### Create new transformations
- own transformation을 implement하고 싶다면 MediatorLiveData class를 사용할 수 있다.
  - MediatorLivedata class는 다른 LiveData object를 수신하고 그 객체의 event를 처리한다.

### [Merge multiple LiveData sources]
- MediatorLiveData는 LiveData의 subclass로 여러 LiveData source를 merge 할 수 있도록 한다.
  - MediatorLiveData object observer는 어느 original LiveData가 변하든 trigger된다.
  - database, network를 통해 받아오는 LiveData object가 있을 때 이를 MediatorLiveData로 merge 후 MediatorLiveData를 observe하면 database, network LiveData object 모두의 update를 observe한다.



## Q&A
#### [Lifecycle]
<b id="f1">1) </b>그럼 이전 버전에서는 lifecycle을 어떻게 처리하고 observe 했을까? [↩](#r1)<br>
<b id="f2">2) </b>Lifecycle class에 대한 구현체로 여러 observer를 handle할 수 있다. [↩](#r2)<br>
- Lifecycle class를 extend했다.
- custom LifecycleOwner를 사용할 때 직접 사용할 수 있다.
- LifecycleOwner에 getLifecycle()이 있고 getLifecycle()를 통해 Lifecycleregistry를 얻는다.
  - Lifecycleregistry에 실제 구현되어 있는 addObserver에 observer를 register하여 여러 observer가 lifecycle를 observe할 수 있도로 할 수 있다.
  - LifecycleRegistry.java에는 "An implementation of Lifecycle that can handle multiple observers" 라고 되어 있다.

<b id="f3">3) </b>Lifecycle/LifecycleOwner/LifecycleObserver/LifecycleRegistry 구성 정리 <br>
#### [ViewModel]
<b id="f4">4) </b>onCleared() 호출 과정
- Activity, Fragment는 ViewmodelStore를 implement한다.
```kotlin
getLifecycle().addObserver(new LifecycleEventObserver() {
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_DESTROY) {
            // Clear out the available context
            mContextAwareHelper.clearAvailableContext();
            // And clear the ViewModelStore
            if (!isChangingConfigurations()) {
                getViewModelStore().clear();
            }
        }
    }
});
```
- ViewmodelStoreOwner를 통해 ViewModelStore를 받아온다.
- lifecycle event가 ON_DESTROY인데 configuration change에 의한 것이 아니라면 ViewmodelStore.clear()
- ViewModel의 onCleared()가 호출되어 ViewModel이 clear된다.[↩](#r4)<br>


<b id="f5">5) </b>ViewModelStore/ViewModelStoreOwner/ViewModelProvider 구성 정리 
- Activity, Fragment는 ViewModelStoreOwner를 implement한다.
  - ViewmodelStoreOwner가 ViewmodelStore를 갖고 있는 scope
- ViewModelProvider를 통해서 Viewmodel instance를 받는다.
  - 내부적으로 mFactory, mViewModelStore를 갖고 있다.
  - mFactory는 Viewmodel 객체 생성 팩토리, mViewModelStore는 생성된 Viewmodel 객체 저장
- ViewModelStore는 Hashmap을 사용하여 Viewmodel class 이름을 key로 Viewmodel를 저장한다.
- ViewModelProvider(ViewModelOwner).get("key")
  - key에 Viewmodel class 이름을 보낸다.
  - 만약 이미 key값에 해당하는 것을 갖고 있다면 ViewModelStore에 있는 것 재사용
  - 만약 없다면 mFactory를 통해 create해서 새롭게 생성하고 ViewModelStore에 저장
  - Factory는 따로 지정하지 않으면 defaultFactory 사용[↩](#r5)<br>

<b id="f6">6) </b>activity scope란 ViewModelStoreOwner로 지정한 것을 말할까? ViewModelStoreOwner에 this를 보내는 것으로 지정 가능한 것은 Activity가 자체적으로 Lifecycle을 갖고 있기 때문인가?[↩](#r6)<br>

#### [LiveData]
<b id="f7">7) </b>LiveData observe 내부 코드 [↩](#r7)<br>
```kotlin
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```
```kotlin
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;

    ...

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }
           
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
        if (currentState == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        Lifecycle.State prevState = null;
        while (prevState != currentState) {
            prevState = currentState;
            activeStateChanged(shouldBeActive());
            currentState = mOwner.getLifecycle().getCurrentState();
        }
    }
}
```

<b id="f8">8) </b>setValue(), postValue() [↩](#r8)<br>
- setValue()는 main thread에서, postValue()는 worker thread에서 사용한다.
- setValue()를 worker thread에서 사용하면 assertMainThread("setValue")오류가 발생한다.
```kotlin
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```
```kotlin
static void assertMainThread(String methodName) {
    if (!ArchTaskExecutor.getInstance().isMainThread()) {
        throw new IllegalStateException("Cannot invoke " + methodName + " on a background"
                + " thread");
    }
}
```
- postValue()는 background thread에서 동작하다가 mainthread에서 setValue()를 통해 값을 변경한다.
```kotlin
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```
```kotlin
private final Runnable mPostValueRunnable = new Runnable() {
    @SuppressWarnings("unchecked")
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        setValue((T) newValue);
    }
};
```
- postValue()는 background thread에서 동작하다가 mainthread에서 handler 통해 값을 post하기 때문에 mainthread에서 getValue()하면 값을 바로 갖고 오지 못한다.


<b id="f9">9) </b>LiveData는 항상 UI를 처리하기 위해 사용하기에 mainthread에서 값을 처리하도록 하는 것일까? 다른 용도는 없을까? [↩](#r9)<br>

<b id="f10">10) </b> transformations LiveData 부분 [↩](#r10)<br>
- You can use transformation methods to carry information across the observer's lifecycle. The transformations aren't calculated unless an observer is watching the returned LiveData object. Because the transformations are calculated lazily, lifecycle-related behavior is implicitly passed down without requiring additional explicit calls or dependencies.

<b id="f11">11) </b>getPostalCode(address).observe() 이런 형식을 사용해서 호출될 때마다 이전 것을 unregister하고 새로운 것을 register 한다는 뜻일까? [↩](#r11)<br>
<b id="f12">12) </b> LiveData는 active observer인 상태에서만 update하고 inactive였다가 active 상태가 되면 다시 최신 상태로 update된다고 하는데 내부적으로 어디서 최신 버전으로 update되는 기능이 동작되는 것일까?



