# Getting a result from an activity
## 참고자료
https://developer.android.com/training/basics/intents/result#kotlin

- startActivityForResult()와 onActivityResult() API는 모든 API level에서 Activity class에서 사용할 수 있지만 AndroidX Activity and Fragment에서 소개하는 Activity Result APIs를 사용할 것을 강력하게 추천한다.

### [Registering a callback for an Activity Result]
- result를 받기 위한 activity를 start할 때 process와 activity는 low memory 때문에 destory될 수 있다.
- 이런 이유로 Activity Result APIs는 result callback을 다른 activity를 launch하는 코드와 분리한다.
- result callback은 process와 activity가 recreate할 떄 사용할 수 있어야 하므로 다른 activity를 launch하는 logic이 user event or business logic을 기반으로 발생하여도 activity가 생성될 때마다 조건없이 register되어야 한다.
- ComponentActivity or Fragment를 사용할 때 Activity Result APIs는 result callback을 register하기 위해 registerForActivityResult() API를 제공한다.
  - registerForActivityResult()는 ActivityResultContract, ActivityResultCallback을 take하고 다른 activity를 launch할 때 ActivityResultLauncher를 return한다.
  - ActivityResultContract은 result를 produce할 때 필요한 input type과 result의 output type을 정의한다.
    - 이는 default contracts를 갖고 있고 custom contracts를 만들 수도 있다.
  - ActivityResultCallback은 onActivityReseult() method를 가진 single method interface이다.
    - onActivityResult() method는 ActivityResultContract에서 정의한 output type에 해당하는 object를 받는다.
```kotlin
// GetContent creates an ActivityResultLauncher<String> to allow you to pass
// in the mime type you'd like to allow the user to select
val getContent = registerForActivityResult(GetContent()) { uri: Uri? ->
    // onActivityResult
    // Handle the returned Uri
}
```
- 만약 다른 contract을 사용하거나 seperate callback을 사용한 multiple activity result calls를 가진다면 multiple ActivityResultLauncher instance를 register하기 위해 여러 번 resgisterForActivityResult()를 호출할 수 있다.
  - 진행 중인 결과가 올바른 callback에게 전달되도록 fragment or activity를 만들 때마다 registerForActivityResult()를 항상 동일하 순서로 호출해야 한다.
- registerForActivityResult()는 fragment or activity를 만들기 전에 호출하는 것이 안전하므로 return되는 ActivityResultLauncher instance의 멤버 변수를 선언할 때 직접 사용할 수 있다.
  - 하지만 fragment or activity의 Lifecycle이 CREATED에 도달할 때까지 ActivityResultLauncher를 실행할 수 없다.

### [Launching an activity for result]
```kotlin
val getContent = registerForActivityResult(GetContent()) { uri: Uri? ->
    // Handle the returned Uri
}

override fun onCreate(savedInstanceState: Bundle?) {
    // ...

    val selectButton = findViewById<Button>(R.id.select_button)

    selectButton.setOnClickListener {
        // Pass in the mime type you'd like to allow the user to select
        // as the input
        getContent.launch("image/*")
    }
}
```
- registerForActivityResult()가 callback을 register할 때 다른 activity를 launch하거나 result를 요구하지 않는다.
  - 대신 이는 ActivityResultLauncher instance의 책임이 된다.
- input이 존재한다면 ActivityResultContract에 정의한 input type에 맞게 input을 launcher에게 넘긴다.
- launch()를 통해 result를 produce하는 프로세스를 시작한다.
- activity가 끝나고 return되면 ActivityResultCallback이 실행되면서 result를 받는다.
  - launch()가 call되고 onActivityReesult()가 result를 받는 사이에 process와 activity가 destroy될 수 있기 때문에 result를 다루는 추가적인 state는 이 APIs와 별도로 save되고 restore되어야 한다.

### [Receiving an activity result in a seperate class]
```kotlin
class MyLifecycleObserver(private val registry : ActivityResultRegistry)
        : DefaultLifecycleObserver {
    lateinit var getContent : ActivityResultLauncher<String>

    override fun onCreate(owner: LifecycleOwner) {
        getContent = registry.register("key", owner, GetContent()) { uri ->
            // Handle the returned Uri
        }
    }

    fun selectImage() {
        getContent.launch("image/*")
    }
}

class MyFragment : Fragment() {
    lateinit var observer : MyLifecycleObserver

    override fun onCreate(savedInstanceState: Bundle?) {
        // ...

        observer = MyLifecycleObserver(requireActivity().activityResultRegistry)
        lifecycle.addObserver(observer)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val selectButton = view.findViewById<Button>(R.id.select_button)

        selectButton.setOnClickListener {
            // Open the activity to select an image
            observer.selectImage()
        }
    }
}
```
- ComponentActivity와 Fragment class가 ActivityResultCaller interface를 implement하여 registerActivityResult() APIs를 사용할 수 있는데 ActivityResultRegistry를 직접 사용해서 ActivityResultCaller를 implement하지 않고 seperate class에서 activity result를 받을 수 있다.
- ActivityResultRegistry APIs를 사용할 때 LifecycleOwner를 사용할 것을 강력히 권장한다.
  - LifecycleOwner가 Lifecycle이 destroyed일 때 등록된 launcher를 자동으로 remove해주기 때문이다.
  - 만약 LifecycleOwner를 사용하지 못한다면 각각의 ActivityResultLauncher class를 unregister()를 통해 직접 remove할 수도 있다.

### [Creating a custom contract]
```kotlin
class PickRingtone : ActivityResultContract<Int, Uri?>() {
    override fun createIntent(context: Context, ringtoneType: Int) =
        Intent(RingtoneManager.ACTION_RINGTONE_PICKER).apply {
            putExtra(RingtoneManager.EXTRA_RINGTONE_TYPE, ringtoneType)
        }

    override fun parseResult(resultCode: Int, result: Intent?) : Uri? {
        if (resultCode != Activity.RESULT_OK) {
            return null
        }
        return result?.getParcelableExtra(RingtoneManager.EXTRA_RINGTONE_PICKED_URI)
    }
}
```
- ActivityResultContracts는 prebuilt ActivityResultContract class를 제공하지만 custom contract를 create할 수도 있다.
- 각각의 ActivityResultContract은 input, output type을 define해야 한다.
- 각 contract은 반드시 createIntent() method를 실행해야 한다.
  - Context와 input을 받고 startActivityForResult()에서 사용할 Intent를 contruct한다.
- 각 contract은 parseResult() method도 실행해야 한다.
  - resultCode와 Intent를 받고 이로부터 output을 생산한다.
- createIntent()를 호출하여 다른 activity를 시작하고 parseResult()를 사용하여 result를 build할 필요 없이 주어진 입력의 결과를 확인할 수 있으면 getSynchronousResult()를 구현할 수도 있다.

```kotlin
val startForResult = registerForActivityResult(StartActivityForResult()) { result: ActivityResult ->
    if (result.resultCode == Activity.RESULT_OK) {
        val intent = result.data
        // Handle the Intent
    }
}

override fun onCreate(savedInstanceState: Bundle) {
    // ...

    val startButton = findViewById(R.id.start_button)

    startButton.setOnClickListener {
        // Use the Kotlin extension in activity-ktx
        // passing it the Intent you want to start
        startForResult.launch(Intent(this, ResultProducingActivity::class.java))
    }
}
```
- custom contract이 필요하지 않다면 StartActivityForResult() contract을 사용하면 된다.
  - generic contract으로 Intent를 input으로 받고 ActivityResult를 return한다.
  - ActivityResult로부터 resultCode, Intent를 받을 수 있다.














