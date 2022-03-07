# Lifecycle-aware components
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
- DefaultLifecycleObserver를 implement한 component와 LifecycleOwner를 implement한 component는 함께 원활하게 작동한다.
  - LifecycleOwner가 Lifecycle를 제공하고 이 Lifecycle에 Observer가 등록되어 monitor하게 된다.
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
- Support Library 26.1.0 이후부터는 Fragments, Activities는 이미 LifecycleOwner를 implement한다.
  - Activities는 ComponentActivity가, Fragments는 Fragment가 LifecycleOwner를 implement
  - getLifecycle()에 대한 실제 구현도 되어 있다.
  - LifecycleRegistry<sup id="r1">[1)](#f1)</sup>를 return
- LifecycleOwner를 직접 구현한 custom class를 갖고 있다면 LifecycleRegistry class를 사용할 수 있다.
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



## Q&A
<b id="f1">1) </b>Lifecycle class에 대한 구현체로 여러 observer를 handle할 수 있다. [↩](#r1)<br>



