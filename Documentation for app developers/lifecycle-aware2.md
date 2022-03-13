# Lifecycle-aware components 2
## Save UI states
- user expectation과 system behavior간의 gap을 줄이기 위해 ViewModel, onSaveInstanceState() method, local storage를 사용하여 UI state를 persist해야 한다.
  - 어떤 것을 사용할 지는 UI data의 complexity, app use cases, 메모리 사용량 대비 검색 속도를 고려한다.
  - 어떤 방법을 사용하든 user expectation을 만족해야 하며 매끄럽고 원활한 UI를 제공해야 한다.
  - 대부분의 경우 ViewModel과 onSaveInstanceState()를 모두 사용해야 한다.

### [User expectations and system behavior]
#### User-initiated UI state dismissal
- 다음과 같은 경우 activity르 완전히 dismiss하고 user 또한 clean state와 함께 activity가 시작되길 expect한다.
  - back button pressed
  - Recents screen에서 swiping
  - activity에서 상위 항목으로 이동
  - 설정 화면에서 app 종료
  - Activity.finish()에 의해 finishing

- 다음과 같은 상황에서는 activity instance가 destroy되고 memory에서 제거되며 저장된 state들도 remove 된다.
  - 이는 user expectation과 system behavior가 match된다.

#### System-initiated UI state dismissal
- configuration change(rotation or switching into multi-window mode)와 같은 상황에서 user는 activity's UI state가 유지되길 expect하지만 system은 activity를 destroy하고 저장된 UI state도 없앤다.
- 잠깐 다른 app으로 switch했다가 돌아오는 경우에도 user는 activity's UI state가 유지되길 expect한다.
  - 하지만 이 경우에도 system이 application process를 destroy할 수 있다.
  - 이렇게 destroy하는 경우에도 UI state도 함께 destroy된다.
- user는 state가 그대로 유지되길 예상하지만 clean state가 되어 user expectation과 달라지게 된다.

### [Options for preserving UI state]<sup id="r1">[1)](#f1)</sup>
<img width="742" alt="스크린샷 2022-03-07 오후 11 32 58" src="https://user-images.githubusercontent.com/17876424/157054003-07513d4a-ab0c-4250-9465-a723605a6f56.png">

### [Use ViewModel to handle configuration changes]
- user가 actively하게 사용하는 application에 대해 UI-related data를 store, manage하기에는 ViewModel이 이상적이다.
  - UI data에 빠르게 접근할 수 있도록 한다.
  - configuration changes가 발생해도 UI data가 유지된다.
- ViewModel은 data를 memory에 저장하기 때문에 disk나 network로부터 data를 받는 것보다 cheap하다.
- ViewModel은 activity(or some other lifecycle owner)와 연관되고 configuration change가 발생해도 memory에 유지된다.
  - system은 configuration change에 의해 새롭게 생성된 activity instance와 Viewmodel을 자동으로 연결시킨다.<sup id="r2">[2)](#f2)</sup>
- ViewModel은 사용자가 activity, fragment에서 벗어나거나 finish()하는 경우 자동으로 system이 destroy한다.
- ViewModel은 saved instance state와 다르게 system-initiated process death 동안 destroy된다.
  - 따라서 onSaveInstanceState()와 함께 사용해야 한다.

### [Use onSaveInstanceState() as backup to handle system-initiated process death]
- Saved instance state는 configuration change, process death에서도 유지되지만 storage, speed limitation이 있다.
  - 왜냐하면 onSaveInstanceState()는 disk에 data가 serialize되기 때문이다.<sup id="ra">[a)](#fa)</sup>
  - serialization<sup id="r3">[3)](#f3)</sup>은 objects being serialized가 복잡하면 많은 메모리를 사용한다.
  - configuration change동안 serialization이 main thread에서 발생하면 프레임 하락과 시각적인 끊김 현상이 발생할 수 있다.
- Serialization 문제가 있기에 primitive type과 simple, small objects만 저장해야 한다.
- Intent를 통해 activity를 열었다면 bundle of extras가 activity에게 전달되고 onSaveInstanceState() bundle 대신 extras bundle를 사용할 수도 있다.<sup id="rb">[b)](#fb)</sup>
- onSaveInstanceState()를 사용하거나 extras bundle를 사용하여도 ViewModel를 여전히 사용하여 configuration change에서 data가 database로부터 reload되는 것을 막을 수 있다.
  - 유지해야 하는 UI data가 simple, lightweight하다면 onSaveInstanceState()만 사용해도 된다.

#### Hook into saved state using SavedStateRegistry
- Fragment 1.1.0, Activity 1.0.0부터 Activity or Fragment와 같은 UI controllers에서 SavedStateRegistryOwner를 implement하고 SavedStateRegistry를 제공한다.
  - SavedStateRegistryOwner는 SavedStateRegistry를 제공
  - SavedStateRegistry가 UI controller의 saved state를 consume하거나 contribute할 수 있도록 한다.
  - SavedStateRegistry가 SavedStateHandle를 create<sup id="r4">[4)](#f4)</sup>
  - getSavedStateRegistry()를 통해 UI controller의 SavedStateRegistry를 받을 수 있다.
- saved state에 contribute하기 위해서는 반드시 SavedStateRegistry.SavedStateProvider를 implement해야 한다.
  - SavedStateProvider는 saveState() single method가 있는 interface
  - saveState()를 통해서 state가 담긴 Bundle를 return받는다.
- saveState()를 통해 saved state Bundle를 받으려면 SavedStateProvider가 필요하고 이를 사용하려면 register해야 한다.
  - SavedStateRegistry.registerSavedSateProvider()를 통해 등록한다.
  - 등록 시 키 값과 해당 Provider를 함께 넘겨준다.
  - 이렇게 등록된 키 값을 이용하여 SavedStateRegistry.consumeRestoredStateForKey()를 호출하여 이전에 저장된 data를 saved state를 통해 받아오게 된다.

- Activity or Fragment에서 onCreate()에서 SavedStateProvider를 register할 수 있다.
  - LifecycleOwner를 implement한 SavedStateRegistryOwner에 LifecycleObserver를 add해서 ON_CREATE event 발생 시 SavedStateProvider를 register할 수도 있다.
```kotlin
class SearchManager(registryOwner: SavedStateRegistryOwner) : SavedStateRegistry.SavedStateProvider {
    companion object {
        private const val PROVIDER = "search_manager"
        private const val QUERY = "query"
    }

    private val query: String? = null

    init {
        // Register a LifecycleObserver for when the Lifecycle hits ON_CREATE
        registryOwner.lifecycle.addObserver(LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_CREATE) {
                val registry = registryOwner.savedStateRegistry

                // Register this object for future calls to saveState()
                registry.registerSavedStateProvider(PROVIDER, this)

                // Get the previously saved state and restore it
                val state = registry.consumeRestoredStateForKey(PROVIDER)

                // Apply the previously saved state
                query = state?.getString(QUERY)
            }
        }
    }

    override fun saveState(): Bundle {
        return bundleOf(QUERY to query)
    }

    ...
}

class SearchFragment : Fragment() {
    private var searchManager = SearchManager(this)
    ...
}
```
- SavedStateRegistry는 onSaveInstanceState()와 같은 Bundle에 데이터를 저장한다.


### [Use local persistence to handle process death for complex or large data]
- database, shared prefereces와 같은 persisten local storage는 application이 user's device에 있는 한 계속 남아있다.
- application process death되어도 데이터는 남아있지만 retrieve하기에 expensive하다.
  - memory의 local storage로부터 읽어와야 하기 때문이다.
- transient UI state의 경우 ViewModel, saved instance state를 사용하고 그 이외의 app data는 persistent storage를 사용해서 저장해야 한다.

### [Managing UI state: divide and conquer]
#### Local persistence
- activity가 finish되어도 잃지 않아야 하는 데이터를 저장한다.
#### ViewModel
- UI Controller에 display해야 하는 모든 데이터를 ViewModel를 통해서 memory에 저장한다.
#### onSaveInstanceState()
- system이 멈추고 다시 생성하는 UI Controller에 대해 reload해야 하는 activity state를 저장한다.
- activity가 background로 간 경우에도 사용
  - small amount of data인 경우


## Saved State module for ViewModel
- configuration change에 대해서는 ViewModel를 사용하면 되지만 system-initiated process death에 대해서도 handle하려면 onSaveInstanceState()를 함께 사용해야 한다.
- UI state는 보통 activities가 아닌 ViewModel에 저장하는데 onSaveInstanceState()를 사용하여 boilerplate를 require하게 된다.
  - 그래서 사용하는 것이 SavedStateHandle
- ViewModel object는 constructor를 통해 SavedStateHandle object를 받는다.
  - 이 object는 key-value map으로 되어 있다.

### [Setup]
```kotlin
class SavedStateViewModel(private val state: SavedStateHandle) : ViewModel() { ... }
```
```kotlin
class MainFragment : Fragment() {
    val vm: SavedStateViewModel by viewModels()

    ...
}
```
- Fragment 1.2.0 혹은 Activity 1.1.0부터 위와 같이 사용한다.
- default ViewModel factory가 ViewModel에게 appropriate한 SavedStateHandle를 제공한다.
- custom ViewModelProvider.Factory instance를 사용하면 AbstractSavedStateViewModelFactory를 extend 해서 SavedStateHandle를 사용할 수 있다.
- 만약 earlier version을 사용한다면 lifecycle-viewmodel-savedstate를 dependency에 추가하고 SavedStateViewModelFactory를 factory로 사용한다.
### [Working with SavedStateHandle]
- SavedStateHandle은 key-value map으로 set(), get()를 통해 사용한다.
- getLiveData()를 통해 MutableLiveData도 return 받는다.
- key 값이 update되면 LiveData는 새로운 값을 받는다.
  - updated value는 transfrom LiveData로 사용할 수 있다.
```kotlin
class SavedStateViewModel(private val savedStateHandle: SavedStateHandle) : ViewModel() {
    val filteredData: LiveData<List<String>> =
        savedStateHandle.getLiveData<String>("query").switchMap { query ->
        repository.getFilteredData(query)
    }

    fun setQuery(query: String) {
        savedStateHandle["query"] = query
    }
}
```
- SavedStateHandle를 사용하면 process death되어도 따로 저장할 필요 없이 데이터가 유지된다.
- 다음과 같은 method도 존재한다.
  - contains(String key) : 주어진 key에 대한 value가 있는지 check
  - remove(String key) : 주어진 key에 대한 value를 remove
  - keys() : SavedStateHandle에 있는 모든 key를 return

### [Supported types]
- SavedStateHandle에 저장된 data는 activity, fragment의 다른 savedInstanceState와 함께 Bundle로 저장된다.
#### Directly supported types
<img width="694" alt="스크린샷 2022-03-08 오전 1 37 55" src="https://user-images.githubusercontent.com/17876424/157077471-6a564b65-5d57-425b-be99-e684b1320f87.png">

- 만약 class가 위의 list를 extend하지 않는다면 parcelable하게 만들어야 한다.

#### Saving non-parcelable classes
- 만약 class가 Parcelable or Serializable<sup id="r5">[5)](#f5)</sup>를 implement하지 않고 modify할 수 없다면 SavedStateHandle를 통해 직접 저장할 수 없다.
- Lifecycle 2.3.0-alpha03부터 SavedStateHandle은 setSavedStateProvider()를 사용하여 Bundle에 object를 저장할 수 있도록 한다.
- saveState()를 통해서 SavedStateProvider로부터 Bundle을 가져오고 저장한다.
- SavedStateHandle.get(key)르 통해서 Bundle를 가져오게 된다.
  - 이때의 key 값은 setSavedStateProvider에서 저장한 Provider key값이다.
```kotlin
private fun File.saveTempFile() = bundleOf("path", absolutePath)

private fun Bundle.restoreTempFile() = if (containsKey("path")) {
    File(getString("path"))
} else {
    null
}

class TempFileViewModel(savedStateHandle: SavedStateHandle) : ViewModel() {
    private var tempFile: File? = null
    init {
        val tempFileBundle = savedStateHandle.get<Bundle>("temp_file")
        if (tempFileBundle != null) {
            tempFile = tempFileBundle.restoreTempFile()
        }
        savedStateHandle.setSavedStateProvider("temp_file") { // saveState()
            if (tempFile != null) {
                tempFile.saveTempFile()
            } else {
                Bundle()
            }
        }
    }

    fun createOrGetTempFile(): File {
      return tempFile ?: File.createTempFile("temp", null).also {
          tempFile = it
      }
    }
}
```
```java
class TempFileViewModel extends ViewModel {
    private File tempFile = null;

    public TempFileViewModel(SavedStateHandle savedStateHandle) {
        Bundle tempFileBundle = savedStateHandle.get("temp_file");
        if (tempFileBundle != null) {
            tempFile = TempFileSavedStateProvider.restoreTempFile(tempFileBundle);
        }
        savedStateHandle.setSavedStateProvider("temp_file", new TempFileSavedStateProvider());
    }

    @NonNull
    public File createOrGetTempFile() {
        if (tempFile == null) {
            tempFile = File.createTempFile("temp", null);
        }
        return tempFile;
    }

    private class TempFileSavedStateProvider implements SavedStateRegistry.SavedStateProvider {
        @NonNull
        @Override
        public Bundle saveState() {
            Bundle bundle = new Bundle();
            if (tempFile != null) {
                bundle.putString("path", tempFile.getAbsolutePath());
            }
            return bundle;
        }

        @Nullable
        private static File restoreTempFile(Bundle bundle) {
            if (bundle.containsKey("path") {
                return File(bundle.getString("path"));
            }
            return null;
        }
    }
}
```

## Use Kotlin coroutines with lifecycle-aware components
### [Add KTX dependencies]
- ViewModelScope 사용 위해서 androidx.lifecycle:lifecycle-viewmodel-ktx:2.4.0 혹은 그 이상 버전 사용
- LifecycleScope 사용 위해서  androidx.lifecycle:lifecycle-runtime-ktx:2.4.0 혹은 그 이상 버전 사용
- liveData 사용 위해서 androidx.lifecycle:lifecycle-livedata-ktx:2.4.0 혹은 그 이상 버전 사용

### [Lifecycle-aware coroutine scopes]
#### ViewModelScope
```kotlin
class MyViewModel: ViewModel() {
    init {
        viewModelScope.launch {
            // Coroutine that will be canceled when the ViewModel is cleared.
        }
    }
}
```
- ViewModel안에 정의하며 ViewModel이 clear되면 자동으로 cancel된다.
- ViewModel이 active할 때만 동작하는 Coroutines을 사용할 때 유용하다.

#### LifecycleScope
```kotlin
class MyFragment: Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewLifecycleOwner.lifecycleScope.launch {
            val params = TextViewCompat.getTextMetricsParams(textView)
            val precomputedText = withContext(Dispatchers.Default) {
                PrecomputedTextCompat.create(longTextContent, params)
            }
            TextViewCompat.setPrecomputedText(textView, precomputedText)
        }
    }
}
```
- Lifecycle object 안에서 정의하며 Lifecycle이 destroy되면 자동을 cancel된다.
- Lifecycle이 CoroutineScope는 lifecycle.coroutineScope or lifecycleOwner.lifecycleScope를 통해 접근할 수 있다.
### [Restarable Lifecycle-aware coroutines]
```kotlin
class MyFragment : Fragment() {

    val viewModel: MyViewModel by viewModel()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Create a new coroutine in the lifecycleScope
        viewLifecycleOwner.lifecycleScope.launch {
            // repeatOnLifecycle launches the block in a new coroutine every time the
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Trigger the flow and start listening for values.
                // This happens when lifecycle is STARTED and stops
                // collecting when the lifecycle is STOPPED
                viewModel.someDataFlow.collect {
                    // Process item
                }
            }
        }
    }
}
```
- lifecycleScope가 Lifecycle이 DESTROYED되면 자동으로 끝나지만 특정 state에서 시작되고 다른 state가 되면 끝나는 작업을 원할 수도 있다.
- Lifecycle과 LifecycleOwner가 repeatOnLifecycle API를 제공한다.
- 특정 state가 되면 항상 run 되고 벗어나게 되면 cancel된다.
### [Suspend Lifecycle-aware coroutines]
```kotlin
class MyFragment: Fragment {
    init { // Notice that we can safely launch in the constructor of the Fragment.
        lifecycleScope.launch {
            whenStarted {
                // The block inside will run only when Lifecycle is at least STARTED.
                // It will start executing when fragment is started and
                // can call other suspend methods.
                loadingView.visibility = View.VISIBLE
                val canAccess = withContext(Dispatchers.IO) {
                    checkUserAccess()
                }

                // When checkUserAccess returns, the next line is automatically
                // suspended if the Lifecycle is not *at least* STARTED.
                // We could safely run fragment transactions because we know the
                // code won't run unless the lifecycle is at least STARTED.
                loadingView.visibility = View.GONE
                if (canAccess == false) {
                    findNavController().popBackStack()
                } else {
                    showContent()
                }
            }

            // This line runs only after the whenStarted block above has completed.

        }
    }
}
```
- Lifecycle이 특정 state에 있지 않다면 code block 실행을 suspend하고 싶을 때 lifecycle.whenCreated, lifecycle.whenStarted, lifecycle.whenResumed method를 사용할 수 있다.
```kotlin
class MyFragment: Fragment {
    init {
        lifecycleScope.launchWhenStarted {
            try {
                // Call some suspend functions.
            } finally {
                // This line might execute after Lifecycle is DESTROYED.
                if (lifecycle.state >= STARTED) {
                    // Here, since we've checked, it is safe to run any
                    // Fragment transactions.
                }
            }
        }
    }
}
```
- Even though these methods provide convenience when working with Lifecycle, you should use them only when the information is valid within the scope of the Lifecycle (precomputed text, for example). Keep in mind that if the activity restarts, the coroutine is not restarted.
- Warning: Prefer collecting flows using the repeatOnLifecycle API instead of collecting inside the launchWhenX APIs. As the latter APIs suspend the coroutine instead of cancelling it when the Lifecycle is STOPPED, upstream flows are kept active in the background, potentially emitting new items and wasting resources.
### [Use coroutines with LiveData]
```kotlin
val user: LiveData<User> = liveData {
    val data = database.loadUser() // loadUser is a suspend function.
    emit(data)
}
```
- liveData building block 내에 suspend function을 사용하면 LiveData가 active하면 시작되고 LiveData가 inactive하면 자동으로 cancel된다.
- 만약 작업이 다 끝나지 않았는데 cancel되었다면 LiveData가 다시 active하게 되면 재시작하게 된다.
  - 이후 성공적으로 마무리하면 재시작하지 않는다.
  - 하지만 자동으로 cancel되었을 때만 재시작하며 CancellationException과 같이 다른 이유로 cancel되었다면 재시작하지 않는다.

## Q&A
#### [Save UI states]
<b id="f1">1) </b>ViewModel, Saved instance state의 data limitations은 정확히 어느 정도인가? [↩](#r1)<br>
- Saved instance state의 경우 parcelable한 Bundle를 사용하는데 Binder transaction buffer를 사용하고 1MB의 size limit를 가진다.
  - 이는 saved instance state, startactivity, system interaction까지 포함되는 것이기에 50K까지만 권장한다.
- ViewModel은?


<b id="f2">2) </b>configuration change 발생해도 기존 ViewModel이 새로운 activity instance와 연결되는 이유는? [↩](#r2)<br>
- activity event가 ON_DESTROY이지만 configuration change에 의한 것이라면 ViewModelStore가 clear되지 않는다.
- onCreate()에서 ViewModelProvider(ViewModelStoreOwner).get()에 의해 ViewModel을 갖고 온다.
- 이전에 생성됐던 ViewModel은 ViewModel class 이름이 key값이 되어 ViewModelstore에 존재
- key값에 해당하는 것이 있기 때문에 기존 것을 return
- 따라서 기존 ViewModel이 새로운 activity instance와 연결된다.

<b id="f3">3) </b>Serialization이란? [↩](#r3)<br>
- 자바 내부 시스템에서 사용하는 객체를 바이트 형태 데이터로 변환해서 자바 외부 시스템에서 사용할 수 있도록 하는 것


<b id="f4">4) </b> SavedStateProvider를 등록하고 SavedStateRegistry를 통해 saved state를 consume했기에 onSavedInstanceState, onRestoreInstanceState도 작동하는 것인가?[↩](#r4)<br>
```kotlin
getSavedStateRegistry().registerSavedStateProvider(ACTIVITY_RESULT_TAG,
        new SavedStateRegistry.SavedStateProvider() {
            @SuppressLint("SyntheticAccessor")
            @NonNull
            @Override
            public Bundle saveState() {
                Bundle outState = new Bundle();
                mActivityResultRegistry.onSaveInstanceState(outState);
                return outState;
            }
        });
addOnContextAvailableListener(new OnContextAvailableListener() {
    @SuppressLint("SyntheticAccessor")
    @Override
    public void onContextAvailable(@NonNull Context context) {
        Bundle savedInstanceState = getSavedStateRegistry()
                .consumeRestoredStateForKey(
                        ACTIVITY_RESULT_TAG);
        if (savedInstanceState != null) {
            mActivityResultRegistry.onRestoreInstanceState(savedInstanceState);
        }
    }
});
```
<b id="f5">5) </b>Parceable과 Serializable의 차이[↩](#r5)<br>

## 추가 Q&A
<b id="fa">a) </b>onSaveInstanceState에 의해 저장된 Saved State는 해당 activity가 destroy되면 disk에서 제거되는가?
- 화면 전환 시 저장됐던 Saved State가 finish()에 의해 destroy되면 제거된 것을 확인할 수 있었다.[↩](#ra)<br>

<b id="fb">b) </b>intent를 통해 activity를 열었다면 bundle of extras가 configuration change, system destroying으로부터 UI state를 저장할 수 있다는 말[↩](#rb)<br>
- Intent를 통해 startActivity로 activity를 시작할 때 bundle of extras를 담아 시작한다.
- 새로운 activity에서 getIntent()를 통해 Intent를 받고 bundle of extras를 받는다.
- onSaveInstanceState의 Bundle이 아닌 Intent에 담겼던 bundle of extras에 데이터를 저장한다.
- 이 데이터가 configuration change, system destroying으로부터 UI state를 저장해서 복구할 수 있다.  










