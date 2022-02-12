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
                add<Fragment1>(R.id.container)
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
                add<Fragment1>(R.id.container, args = bundle)
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

