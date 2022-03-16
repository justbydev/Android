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









