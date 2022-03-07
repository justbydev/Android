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
  - 왜냐하면 disk에 data가 serialize되기 때문이다.
  - serialization<sup id="r3">[3)](#f3)</sup>은 objects being serialized가 복잡하면 많은 메모리를 사용한다.
  - configuration change동안 serialization이 main thread에서 발생하면 프레임 하락과 시각적인 끊김 현상이 발생할 수 있다.
- Serialization 문제가 있기에 primitive type과 simple, small objects만 저장해야 한다.
- Intent를 통해 activity를 열었다면 bundle of extras가 activity에게 전달되고 onSaveInstanceState() bundle 대신 extras bundle를 사용할 수도 있다.
- onSaveInstanceState()를 사용하거나 extras bundle를 사용하여도 ViewModel를 여전히 사용하여 configuration change에서 data가 database로부터 reload되는 것을 막을 수 있다.

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








## Q&A
#### [Save UI states]
<b id="f1">1) </b>ViewModel, Saved instance state의 data limitations은 정확히 어느 정도인가? [↩](#r1)<br>
<b id="f2">2) </b>configuration change 발생해도 기존 ViewModel이 새로운 activity instance와 연결되는 이유는? [↩](#r2)<br>
- activity event가 ON_DESTROY이지만 configuration change에 의한 것이라면 ViewModelStore가 clear되지 않는다.
- onCreate()에서 ViewModelProvider(ViewModelStoreOwner).get()에 의해 ViewModel을 갖고 온다.
- 이전에 생성됐던 ViewModel은 ViewModel class 이름이 key값이 되어 ViewModelstore에 존재
- key값에 해당하는 것이 있기 때문에 기존 것을 return
- 따라서 기존 ViewModel이 새로운 activity instance와 연결된다.

<b id="f3">3) </b>Serialization이란? [↩](#r3)<br>
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

