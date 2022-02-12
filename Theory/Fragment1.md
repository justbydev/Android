# Fragment1(androidx.fragment:fragment-ktx version)
## 1. Fragment란
#### 1) Fragment 정의
- Micro Activity
- Activity가 할 수 있는 일은 Fragment가 다 할 수 있다.
#### 2) Create Fragment
add dependency
```javascript
dependencies {
    val fragment_version = "1.4.0"
    // Kotlin
    implementation("androidx.fragment:fragment-ktx:$fragment_version")
}
```
Past Version
- inflater를 이용해서 layout을 container에 inflate
```javascript
class Fragment1 : Fragment() {

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.fragment_1, container, false)
    }

}
```

2022 기준 Latest Version
- Fragment() 내부에 layout을 써서 onCreateView에서 했던 inflate가 됨
- onViewCreated의 view는 inflate 하여 return 되는 view가 담김
- Binding.bind(View) 사용하여 view binding
```javascript
class Fragment1 : Fragment(R.layout.fragment_1) {
    var binding: Fragment1Binding?= null
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding = Fragment1Binding.bind(view)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        binding = null
    }
}
```
#### 3) Attach Fragment to Activity
View
- FragmentContainerView 사용
- Activity Attach
Past Version
- FragmentManager가 관리
- Transaction 얻음(backstack에 transaction 단위로 쌓임)
- commit() 실행
- savedInstanceSate == null의 경우 초기 한번만 Fragment를 attach 하기 위한 것
```javscript
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        if (savedInstanceState == null){
            val fragmentManager = supportFragmentManager
            val transaction = fragmentManager.beginTransaction()
            transaction.setReorderingAllowed(true)
            transaction.add(R.id.container, Fragment1())
            transaction.commit()
        }
    }
}
```
Latest Version
- ktx에서 사용 가능(androidx.fragment:fragment-ktx)
- fragmentmanger에서 바로 commit{}
- lambda 형식으로 this:FragmentTransaction
```javascript
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        if (savedInstanceState == null){
            supportFragmentManager.commit {
                setReorderingAllowed(true)
                add<Fragment1>(R.id.container, tag = "Fragment1")
            }
        }
    }
}
```

#### 4) Pass Data
- Bundle을 사용하여 arguments로 전달 가능
- onViewCreated에서 arguments 받음
- 두가지 방법 존재
  - null인지 아닌지 모를 경우 arguments?로 받음
  - null이 확실히 아니 경우 requireArguments()로 받음
```javascript
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        if (savedInstanceState == null){
            supportFragmentManager.commit {
                val bundle = bundleOf("Intvalue" to 0, "Stringvalue" to "test")
                setReorderingAllowed(true)
                add<Fragment1>(R.id.container, args = bundle, tag = "Fragment1")
            }
        }
    }
}
```
```javascript
class Fragment1 : Fragment(R.layout.fragment_1) {
    var binding: Fragment1Binding?= null
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val value = arguments?.getInt("Intvalue")
        val notnullvalue = requireArguments().getString("Stringvalue")
        binding = Fragment1Binding.bind(view)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        binding = null
    }
}
```

## 2. FragmentManager
#### 1) 정의
- Fragment와 관련된 작업이나 instance를 관리하는 클래스
#### 2) 접근 방법
- getSupportFragmentManager() 사용
- getChildFragmentManager(), getParentFragmentManager()를 통해 서로 다른 Fragment의 Manager도 얻을 수 있음
- getFragmentById(), getFragmentByTag()를 통해 Fragment를 얻을 수 있음

## 3. FragmentTransaction
#### 1) 정의
- Fragment에 대한 여러 가지 제어를 Transaction 개념으로 추상화 한 것
- beginTransaction을 하면 반드시 commit()을 해야 한다
- 위 예시에서는 fragment-ktx를 사용하여 Transaction이 생략됨
#### 2) Fragment visibility operation
- Fragment는 Fragment instance(FragmentManager가 관리)와 뷰 계층이 별도로 동작할 수 있음
- add/replace/attach
  - add : Fragment instance 생성, 뷰 계층 추가, 처음 생성 시 사용
    - overdraw 발생
  - replace : 이미 존재하는 Fragment instance가 제거되고 새로운 Fragment instance 생성(뷰 계층에서도 제거 후 추가)
    - remove() 후 add()
    - overdraw 발생하지 않음
  - attach : detach 되었던 Fragment를 뷰 계층에 다시 추가
- remove/detach
  - remove : Fragment instance와 뷰 계층에서 둘다 제거
  - detach : 뷰 계층에서만 제거
- show/hide
  - Fragment 상태와 상관 없이 visibility를 관리
#### 3) commit()
- Transaction을 실행하면 반드시 commit()을 해야 함
  - fragment-ktx에서도 transaction을 생략했지만 suppotFragmentManager.commit{} 으로 코드 작성
- 비동기 작업임
  - 비동기이기 때문에 commit() 작업이 바로 되지 않아 이후 원하는 결과를 얻지 못할수도 있음
  - commit()은 결국 inflate 작업이 필요한데 무게가 꽤 있어 동기로 하면 ANR 발생 가능성이 있어 비동기로 실행
  - commitNow()를 하면 동기 작업 보장
- onSaveInstanceState() 호출 후 commit() 호출 시 Runtime Exception
  - onSaveInstanceState()는 onStop() 후 호출되는 것이기 때문에 현재 Background
  - 화면 보이지 않는데 UI rendering 하는 것이 의미 없다 생각하여 막음
  - Runtime Exception 막기 위해 보통 commit(allowingStateLoss = true) 함
#### 4) Back stack
- Back stack에는 Transaction 단위로 들어가는 것
- addToBackStack("name") : back stack에 Fragment transaction push
- popBackStack() : back stack에서 Fragment transaction pop
- popBackStack("name", 0) : "name" 제외하고 "name" transaction 나오기 전까지 pop
- popBackStack("name", POP_BACK_STACK_INCLUSIVE) : "name" 포함 "name" transaction 나올때까지 pop
- androidx 에서는 back key 처리가 onBackPressedDispatcher로 위임
- pop을 하면 push transaction 반대로
  - add <-> remove
  - attach <-> detach
  - replace라면 add한 것을 remove 후 remove한 것을 add
#### 5) Multiple back stack
- saveBackStack("transaction name")
  - back stack 상 name부터 그 위의 transaction의 상태를 따로 save
- restoreBackStack("name")
  - save한 transaction을 다시 back stack에 push


## 4. 참고 자료
[Fragment Overview](https://developer.android.com/guide/fragments)<br>
[Create Fragment](https://developer.android.com/guide/fragments/create)<br>
[Fragment Manager](https://developer.android.com/guide/fragments/fragmentmanager)<br>
[Fragment Transaction](https://developer.android.com/guide/fragments/transactions)<br>
[Fragment back stack & multiple back stack](https://medium.com/androiddevelopers/multiple-back-stacks-b714d974f134)<br>

## 5. Additional Study Needed
- Back stack & Multiple Back stack







