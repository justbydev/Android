# Fragment2(androidx.fragment:fragment-ktx version)
## 1. Animate transitions between Fragments
#### 1) Animation
- res/anim resource file 사용
- anim은 view에 대한 animation
```javascript
supportFragmentManager.commit {
  setCustomAnimations( 
    enter = R.anim.~
    exit = R.anim.~
    popEnter = R.anim.~
    popExit = R.anim.~
  )
}
```
- setCustomAnimations() 사용(Fragment A -> Fragment B라 한다면)
  - enter : B 들어올 때의 animation
  - exit : A 빠져나갈 때의 animation
  - popEnter : 빠져나간 A 다시 들어올 때의 animation
  - popExit : 들어왔던 B가 종료되어 빠져나갈 때의 animation
#### 2) Transition
- res/transition resource file 사용
- transtion은 layout/view hierarchy의 화면 전환에 대한 animation
```javascript
class FragmentA : Fragment(R.layout.fragment_A) {
    var binding: FragmentABinding?= null
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val inflater = TransitionInflater.from(requireContext())
        exitTransition = inflater.inflateTransition(R.transition.exit_trans)
    }
}
```
```javascript
class FragmentB : Fragment(R.layout.fragment_B) {
    var binding: FragmentBBinding?= null
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val inflater = TransitionInflater.from(requireContext())
        enterTransition = inflater.inflateTransition(R.transition.enter_trans)
    }
}
```
- onCreate에서 TransitionInflater 사용
- exitTransition은 나가는 transition
- enterTransition은 들어오는 transition
#### 3) Shared element transitions
- 공유 요소 뷰에 대한 transition을 하는 것
- Fragment에서 Fragment 뿐 아니라 Activity에서 Activity도 가능
- 설정 순서(FragmentA -> FragmentB로 이하 A->B)
  1) A, B가 share하는 요소의 transitionName을 지정
     - ViewCompat.setTransitionName(View, "View name")
     - xml에서 app:TransitionName으로도 지정 가능 
  2) FragmentTransaction에 addSharedElement 추가
     - addSharedElement(sharedElement, transitionName)
     - shredElement는 B와 share중인 A의 View
     - transitionName은 share하는 B의 transitionName(첫번째 순서에서 지정한 TransitionName)
  3) transition animations 지정
     - 모든 method는 B에 지정
     - sharedElementEnterTransition() : A에서 B로 이동할 때의 transition
     - sharedElementReturnTransition() : B에서 다시 A로 이동할 때의 transition
     - sharedElementEnterTransition = TransitionInflater.from(context).inflateTransition(R.trans.~)

## 2. Lifecycle
#### 1) 정의
- Fragment는 2개의 Lifecycle을 가진다
  - Fragment instance life cycle & View lifecycle
- Fragment는 LifecycleOwner를 가진다
  - this는 Fragment, getViewLifecyclerOwner는 View
  - getViewLifecycleOwner를 권고
#### 2) Fragment Callbacks
- onResumeFragments()
  - activity에 attach된 모든 Fragment의 onResume() 호출되는 callback
  - 만약 이 method를 override한다면 반드시 super를 해야 onResume()이 호출된다
  - activity lifecycle의 onresume() 다음 호출
- onAttach()/onDetach()
  - onAttach() 되었으면 FragmentManager에 추가되고 activity에 attach
  - onDetach() 되었으면 FragmentManager에서 제거되고 activity에 detach
- onCreate()/onDestroy()
  - Fragment에 대한 lifecycle
- onCreateView()/onDestroyView()
  - View에 대한 lifecycle
- onViewCreated()
  - Fragment의 View들이 fragment에 모두 세팅된 직후에 호출
  - onCreateView()에서 반환한 View를 인자로 받음
#### 3) LifecycleOwner
- Fragment와 View의 lifecycleowner를 각각 가지고 있음
- INITIALIZED, CREATED, STARTED, RESUMED, DESTROYED가 lifecycleowner가 갖고 있는 lifecycle
  - 이런 LifecycleOwner를 직접 사용하기 보다 Fragment Callbacks를 제공하여 다룰 수 있도록 한 것
<img width="320" alt="스크린샷 2022-02-13 오전 10 53 49" src="https://user-images.githubusercontent.com/17876424/153734954-296f993d-a821-42fe-bfd9-1dbb3ebfdfb2.png">

## 3. Saving state
#### 1) 상태 유형
- Variable
  - Fragment의 local variable
  - Fragment가 파괴되면 다 초기화
- View state
  - View가 보유하 데이터
  - View는 내부적으로 onSaveInstanceState(), onRestoreInstanceState()가 구현되어 있어 자동으로 유지됨
  - 단, ID를 반드시 설정해야 함
- savedState
  - onSaveInstanceState()로 저장한 데이터
  - Bundle로 전단되는 인자를 통해서 복구
- NonConfig
  - configuration과 무관한 사용자 생성 데이터(서버 결과값, DB값 등)
<img width="670" alt="스크린샷 2022-02-13 오전 11 32 57" src="https://user-images.githubusercontent.com/17876424/153735785-81dbf7a2-0ae1-4ca6-8957-b4cdb35d6767.png">

## 4. Communicate with fragment
#### 1) 기본
- 기본적으로 Fragment의 통신은 Activity의 FragmentManager를 통해 이루어짐
- 하지만 Activity를 통해 통신을 하게 되면 의존성이 높아져 문제가 생길 수 있음
  - Viewmodel을 사용하도록 반강제성을 부여
#### 2) Viewmodel
- configuration 변경과 상관없이 데이터 유지
- viewmodelStoreOwner가 Viewmodel을 저장하는데 owner가 같으면 같은 viewmodel을 돌려준다
  - 이런 이유로 data 공유 가능
#### 3) communicate between Activity and Fragment using Viewmodel
- Activity와 Fragment간의 통신
  - Activity에서 Viewmodel을 생성
  - Fragment에서 동일 Viewmodel을 공유
```javascript
private val viewModel : ShareViewModel by viewModels()
```
```javascript
private val viewModel : ShareViewModel by activityViewModels()
```
#### 4) communicate between Fragments using Viewmodel
- Fragment끼리의 통신
  - 역시 Activity에서 생성한 Viewmodel을 공유하여 통신
  - activityViewModels()을 통신할 Fragment에서 작성하여 통신
#### 5) communicate between parent and child fragments using Viewmodel
- requireParentFragment() 사용
```javascript
class ParentFragment: Fragment() {
  private val viewModel : ShareViewModel by viewModels()
}
```
```javscript
class ChildFragment: Fragment() {
  private val viewModel : ShareViewModel by viewModels({requireParentFragment()})
}
```
#### 6) communicate between Fragments using Result API
- FragmentManager를 통해서 통신하는 방법
- 데이터를 받는 쪽에서는 setFragmentResultListener() 설정(onCreate()에서 설정)
- 데이터 보내는 쪽에서는 setFragmentResult() 작성
- requiestkey와 데이터 담을 bundle 작성
```javascript
class Dataget_Fragment : Fragment(R.layout.fragment_Data_get) {
    var binding: Fragment1Binding?= null
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val inflater = TransitionInflater.from(requireContext())
        setFragmentResultListener("requestkey") { requestKey, bundle ->  

        }
    }
}
```
```javascript
class Datasend_Fragment : Fragment(R.layout.fragment_Data_send) {
  ...
  setFragmentResult("requestkey", bundleOf(...))
}
```
## 5. DialogFragment
#### 1) 정의
- Dialog를 create & host 하기 위해 설계된 Fragment의 subclass
- Fragment 안에 따로 dialog를 만들 필요 없이 FragmentManager가 관리
#### 2) onCreateDialog()
```javascript
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
```javascript
val dialog = PurchaseConfirmationDialogFragment()
dialog.show(childFragmentManager, PurchaseConfirmationDialog.TAG)
dialog.dismiss()
dialog.hide()
```
- dialog start할 때는 dialog.show()
- dialog stop할 때는 dialog.hide()
- dialog detroy할 때는 dialog.dismiss()
- show, hide, dismiss 등 내부적으로 FragmentTransaction()이 구현되어 있음

#### 3) using custom views
- onCreateDailog() 방법은 기존 AlertDialog() 사용하는 방식대로 하는 것
- 원하는 View를 inflate하여 생성 가능
- 일반 Fragment와 마찬가지로 onCreateView(), onViewCreated()를 사용
  - onCreateDialog() 사용안해도 된다
```javascript
class PurchaseConfirmationDialogFragment : DialogFragment(R.layout.purchase_confirm_dialog) {
    var binding: PurchaseConfirmDialogBinding?= null
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding = PurchaseConfirmDialogBinding.bind(view)
    }
    companion object {
        const val TAG = "PurchaseConfirmationDialog"
    }
}
```
## 6. 참고 자료
[Fragment Animation](https://developer.android.com/guide/fragments/animate)<br>
[Fragment Lifecycle](https://developer.android.com/guide/fragments/lifecycle)<br>
[Saving state](https://developer.android.com/guide/fragments/saving-state)<br>
[Communicate with fragment](https://developer.android.com/guide/fragments/communicate)<br>
[DialogFragment](https://developer.android.com/guide/fragments/dialogs)<br>
[Shared element transitions](https://medium.com/@bherbst/fragment-transitions-with-shared-elements-7c7d71d31cbb)

## 7. Additional Study Needed
- Fragment Animation
- Shared element transitions
- Viewmodel






