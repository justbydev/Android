# Fragment1(androidx.fragment:fragment-ktx version)
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







