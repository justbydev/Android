# Fragments
## 참고자료
[Android Developers: Fragments](https://developer.android.com/guide/fragments)

## Overview
- Fragment는 app UI의 재사용 가능한 부분을 나타낸다.
- Fragment는 자체 layout을 정의 및 관리하고 자체 lifecycle을 갖고 자체 input events를 처리한다.
- Fragment는 독립적으로 존재할 수 없고 activity나. 다른 Fragment에 hosting되어야 한다.
  - Fragment의 view hierarchy는 host view hierarchy의 일부가 되거나 attach된다.

### [Modularity]
- Fragment는 activity의 UI를 discrete chunk로 나누면서 modularity, reusability를 제공한다.
- activity는 navigation drawer, bottom navigation bar와 같은 global elements를, fragments는 UI의 single screen or portion of a screen을 정의하기에 적합하다.
<img width="767" alt="스크린샷 2022-03-15 오전 10 00 36" src="https://user-images.githubusercontent.com/17876424/158285450-600e406e-2719-47ea-9c18-b587a221536c.png">

- UI를 fragments로 나누면 runtime 때 activity의 appearance를 더 쉽게 수정할 수 있다.
  - activity lifecycle state이 STARTED 또는 그 이상의 상태일 때 fragments를 add, replace, remove할 수 있다.
  - 이런 변경사항은 activity에서 관리하는 back stack에 보관할 수 있어 취소도 가능하다.
- 같은 activity, 여러 activities 혹은 다른 fragment의 child로 같은 fragment class의 여러 instance를 사용할 수 있다.
  - 그렇기 때문에 자체 UI를 관리하기 위한 logic만 fragment에게 제공해야 한다.
  - 한 fragment를 다른 fragment에 의존하거나 조작하지 않아야 한다.

## Create a fragment
### [Setup your environment]
- Fragments는 AndroidX Fragment library dependency를 필요로 한다.
  - 이를 위해 project의 settings.gradle file에 Google Maven repository를 추가해야 한다.
```kotlin
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        ...
    }
}
```
- AndroidX Fragment library를 project에 include하기 위해 app's build.gradle file에 다음의 종속성을 추가한다.
```kotlin
dependencies {
    val fragment_version = "1.4.1"

    // Java language implementation
    implementation("androidx.fragment:fragment:$fragment_version")
    // Kotlin
    implementation("androidx.fragment:fragment-ktx:$fragment_version")
}
```

### [Create a fragment class]
- Fragment를 create하기 위해 AndroidX Fragment class를 extend하고 app logic을 삽입하기 위해 extend한 class의 method를 override한다.
- 자체적인 layout을 갖는 최소한의 fragment를 create하려면 fragment의 layout resource를 base constructor에 제공한다.
```kotlin
class ExampleFragment : Fragment(R.layout.example_fragment)
```
- Fragment library는 다른 specialized fragment base class도 제공합니다.
- DialogFragment<sup id="r1">[1)](#f1)</sup>
  - floating dialog를 display
  - fragments가 Dialog의 creation, cleanup을 자동으로 관리해 주기 때문에 Activity class에서 사용하는 dialog helper method를 대체하기 좋다.
- PreferenceFragmentCompat
  - Preference objects hierarchy를 list로 display
  - settings screen을 create할 때 사용할 수 있다.

### [Add a fragment to an activity]
- 일반적으로 fragment가 activity layout UI의 일부가 되려면 AndroidX FragmentActivity 내에 포함되어야 한다.
  - AppcompatActivity는 FragmentActivity를 상속하고 있고 하위 호환성을 위해 이미 AppcompatActivity를 상속하고 있다면 activity base class를 수정할 필요 없다.
- activity view hierarchy에 fragment를 추가하는 방법은 두 가지가 있다.
  - activity layout file에 fragment를 정의하는 방법
  - activity layout file에 fragment container를 정의하고 프로그래밍적으로 activity에 fragment를 추가하는 방법
- 위 두 방법 모두 activity view hierarchy 내에 fragment가 위치해야 하는 장소를 정의하고 있는 FragmentContainerView<sup id="r2">[2)](#f2)</sup>를 추가해야 한다.
  - FragmentContainerView는 FrameLayout과 같은 다른 view group이 제공하지 않는 fragment 관련 수정 사항이 포함되어 있어 이를 사용할 것을 강력히 권장한다. 

#### Add a fragment via XML
```Gradle
<!-- res/layout/example_activity.xml -->
<androidx.fragment.app.FragmentContainerView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragment_container_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:name="com.example.ExampleFragment" />
```
- activity layout xml에 fragment 추가를 위해 FragmentContainerView를 사용한다.
- android:name에 인스턴스화될 Fragment class name을 적는다.
- activity layout이 inflate되면 FragmentContainerView가 inflate되면서 지정한 fragment가 인스턴스화 되며 새롭게 인스턴스화된 fragment에서 onInflate()가 호출된다.
  - 그리고 FragmentManager에 fragment를 add하기 위해 FragmentTransaction이 create된다.

#### Add a fragment programmatically
```kotlin
<!-- res/layout/example_activity.xml -->
<androidx.fragment.app.FragmentContainerView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragment_container_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```
- android:name attribute를 사용하지 않고 FragmentTransaction을 사용하여 fragment를 인스턴스화하고 activity layout에 add한다.
```kotlin
class ExampleActivity : AppCompatActivity(R.layout.example_activity) {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        if (savedInstanceState == null) {
            supportFragmentManager.commit {
                setReorderingAllowed(true)
                add<ExampleFragment>(R.id.fragment_container_view)
            }
        }
    }
}
```
- activity가 running하는 동안 adding, removing, replacing과 같은 fragment transaction을 만들 수 있다.
- FragmentActivity에서 FragmentManager instance를 얻을 수 있고 이를 이용하여 FragmentTransaction을 create한다.
  - activity onCreate()에서 fragment container 역할을 하는 ViewGroup ID와 fragment class를 FragmentTransaction.add()에 pass하여 fragment를 인스턴스화 한다.
  - 이후 transaction을 commit한다.<sup id="r3">[3)](#f3)</sup>
```kotlin
class ExampleActivity : AppCompatActivity(R.layout.example_activity) {
      override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        if (savedInstanceState == null) {
            val bundle = bundleOf("some_int" to 0)
            supportFragmentManager.commit {
                setReorderingAllowed(true)
                add<ExampleFragment>(R.id.fragment_container_view, args = bundle)
            }
        }
    }
}
```
```kotlin
class ExampleFragment : Fragment(R.layout.example_fragment) {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val someInt = requireArguments().getInt("some_int")
        ...
    }
}
```
- fragment transaction은 savedInstanceState이 null일 때만 create 되도록 한다.
  - 이는 activity가 처음 create 될 때 fragment가 한번만 add되도록 하는 것을 보장하기 위해서다.
  - 예를 들어 configuration change가 발생하여 activity가 recreate될 떄는 savedInstanceState가 더 이상 null이 아니며 savedInstanceState가 fragment가 자동으로 복원하기 때문에 두번 add할 필요가 없다.<sup id="r4">[4)](#f4)</sup>
- 만약 fragment가 initial data를 필요로 한다면 Bundle을 통해 fragment에게 arguments로 전달할 수 있다.
  - 이 Bundle은 requireArguments()를 통해 retrieve하고 알맞은 Bundle getter method를 사용하여 각각의 argument를 retrieve한다.

## Fragment manager
- FragmentManager는 app의 fragment를 add, remove, replace하거나 back stack에 fragment를 추가하는 작업을 수행하는 class다.
- Jetpack Navigation library를 사용하면 FragmentManager를 직접 다루어야 할 일이 없어진다.
### [Access the FragmentManager]
#### Accessing in an activity
- 모든 FragmentActivity 및 그 subclass(AppcompatActivity)는 getSupportFragmentManager()를 통해 FragmentManager에 access할 수 있다.
#### Accessing in a Fragment
- Fragments도 하나 이상의 child fragments를 hosting할 수 있다.
  - fragment의 child fragments를 관리하는 FragmentManager는 getChildFragmentManager()를 통해 접근할 수 있다.
  - host FragmentManager에 access하려면 getParentFragmentManager()를 통해 가능하다.
<img width="728" alt="스크린샷 2022-03-15 오후 1 36 52" src="https://user-images.githubusercontent.com/17876424/158307407-7b098102-80e1-4a78-914b-267f631410bf.png">

#### Child fragments
- 일반적으로 앱은 application project에서 하나 또는 적은 수의 activity로 구성되어야 하며 각 activity는 관련 screen group을 나타낸다
- 이 activity는 top-level navigation point를 제공하고 ViewModel 및 fragment 간 다른 view-state 범위를 제공한다.
  - 각 activity의 individual destination은 fragments로 표시되어야 한다.
- 만약 여러 fragmentsf를 한번에 표시하려면 child fragments와 child fragment manager를 사용해야 한다.

### [Using the FragmentManager]
- FragmentManager는 fragment back stack을 관리한다.
- runtime 때, user interaction에 반응하여 fragment를 adding or removing 하는 등의 back stack operations를 수행한다.
  - 각각의 change set은 FragmentTransaction이 불리는 single unit에 의해 함께 commit된다.
- user가 Back button을 누르거나 FragmentManager.popBackStack()을 호출하면 가장 상위 fragment transaction이 stack으로부터 pop off된다.
  - 즉, stack으로부터 pop off되면 transaction이 reverse된다.
  - 만약 stack에 fragment transaction이 없고 child fragments를 사용하지 않는다면 back event는 activity로 넘어간다.
- transaction에서 addToBackStack()을 호출하면 많은 작업이 transaction에 포함될 수 있다.
  - back stack이 pop되면 이런 operation들이 single atomic action으로써 reverse된다.
- 만약 popBackStack() 하기 전에 추가적인 transaction을 addToBackStack() 없이 commit 했다면 이 operations들은 reverse되지 않는다.
  - 그러므로 single FragmentTransaction에서 back stack에 영향을 주는 것과 그렇지 않은 transaction을 섞어서 쓰지 않아야 한다.<sup id="r5">[5)](#f5)</sup>

### [Perform a transaction]
- fragment를 layout container에 display하기 위해서 FragmentManager를 통해 FragmentTransaction을 create해야 한다.
  - 그 후 transaction을 통해 container에 add() or replace()를 한다.<sup id="r6">[6)](#f6)</sup>

```kotlin
supportFragmentManager.commit {
   replace<ExampleFragment>(R.id.fragment_container)
   setReorderingAllowed(true)
   addToBackStack("name") // name can be null
}
```
- replace()를 통해 R.id.fragment_container라는 ID를 가진 layout container에 현재 있는 fragment를 ExampleFragment가 replace한다.
- setReorderingAllowed(true)는 animation과 transition이 올바르게 작동하도록 transaction과 관련된 상태 변화를 최적화한다.
- addToBackStack()은 transaction을 back stack으로 commit한다.
  - Back button을 통해 transaction을 reverse하여 이전 fragment를 다시 가져온다.
  - single transaction에서 여러 fragment가 add되고 remove되었다면 모든 operation이 undone된다.
  - "name"처럼 이름을 지정할 수 있는데 이를 통해 popBackStack()에서 특정 transaction을 pop back할 수 있다.
- 만약 remove a fragment without addToBackStack()이라면
  - transaction이 commit되면 removed fragment는 destroy되고 그 fragment로 다시 돌아갈 수 없다.
- 만약 remove a fragment with addToBackStack()이라면
  - 그 fragment는 STOPPED만 되고 이후에 RESUMED되어 다시 돌아갈 수 있다.
  - view는 destroy되었다.
#### Finding an existing fragment
- findFragmentById()를 통해 현재 layout container에 있는 fragment에 대한 참조를 얻을 수 있다.
- XML로부터 inflate된 ID나 FragmentTransaction에 add할 때 사용하는 container ID를 사용할 수 있다.
```kotlin
supportFragmentManager.commit {
   replace<ExampleFragment>(R.id.fragment_container)
   setReorderingAllowed(true)
   addToBackStack(null)
}

...

val fragment: ExampleFragment =
        supportFragmentManager.findFragmentById(R.id.fragment_container) as ExampleFragment
```
- findFragmentById() 대체로 fragment에 unique tag를 부여하고 findFragmentByTag()를 통해서도 가능하다.
  - layout에 정의된 fragments에 XML attribute android:tag를 사용하거나 FragmentTransaction에서 add() or replace()  할 때 지정할 수 있다.
```kotlin
supportFragmentManager.commit {
   replace<ExampleFragment>(R.id.fragment_container, "tag")
   setReorderingAllowed(true)
   addToBackStack(null)
}

...

val fragment: ExampleFragment =
        supportFragmentManager.findFragmentByTag("tag") as ExampleFragment
```
#### Special considerations for child and sibling fragments<sup id="r7">[7)](#f7)</sup>
- 특정 시점에서 오직 한 개의 FragmenManager만 fragment back stack을 조절할 수 있다.
  - 만약 한 시점에 여러 sibling fragment를 보여주거나 child fragment를 사용한다면 하나의 FragmentManager만 지정하여 app primary navigation을 처리해야 한다.
- fragment transaction에서 primary navigation fragment를 정의하기 위해서 transaction에서 setPrimaryNavigationFragment() method를 호출하여 childFragmentManager가 primary control이 되어야 할 fragment instance를 전달한다.
- navigation structure를 layers라고 생각하고 activity를 가장 바깥쪽 layer로 간주하고 child fragment의 각 layer를 아래에  wrapping한다.
  - 각 layer에는 primary navigation fragment가 하나씩 있어야 한다.
  - Back event가 발생하면 가장 안쪽 layer가 navigation behavior를 제어하고 가장 안쪽 layer가 다시 표시할 fragment transaction이 더 이상 없으면 control은 다음 바깥 layer로 돌아가며 이는 activity에 도달할 때까지 반복한다.
- 동시에 두 개 이상의 fragment가 표시되면 그 중 하나만 primary navigation fragment가 될 수 있다.
  - fragment를 primary navigation fragment로 설정하면 이전 fragment 지정은 삭제된다.

### [Support multiple back stacks]
- bottom navigation bar를 사용하는 것과 같은 상황에서 multiple back stack를 필요로 할 수 있다.
- FragmentManager는 saveBackStack(), restoreBackStack() method를 통해 multiple back stacks를 지원한다.
- saveBackStack()은 addToBackStack()에서 지정했던 name parameter를 사용하여 지정된 특정 transaction과 stack 상에서 그 이후의 모든 transaction을 pop한다.
  - 이렇게 pop된 transactions의 모든 fragments의 state는 save된다.
  - savestate()를 통해 fragment의 state가 save된다.
```kotlin
supportFragmentManager.commit {
  replace<ExampleFragment>(R.id.fragment_container)
  setReorderingAllowed(true)
  addToBackStack("replacement")
}
```
```kotlin
supportFragmentManager.saveBackStack("replacement")
```
- 이후 같은 name parameter를 restoreBackStack()에 사용하여 pop된 모든 transaction과 saved된 모든 fragment state를 restore 한다.
```kotlin
supportFragmentManager.restoreBackStack("replacement")
```
- transaction이 single, atomic operation으로 restore되기 위해 setReorderingAllowed(true)<sup id="r8">[8)](#f8)</sup>인 transaction에 대해서만 saveBackStack()이 가능하다.
- addToBackStack()에서 name을 지정하지 않으면 saveBackStack(), restoreBackStack()을 사용할 수 없다.<sup id="r9">[9)](#f9)</sup>

### [Provide dependencies to your fragments]
```kotlin
fragmentManager.commit {
    // Instantiate a new instance before adding
    val myFragment = ExampleFragment()
    add(R.id.fragment_view_container, myFragment)
    setReorderingAllowed(true)
}
```
- fragment를 add할 때 fragment를 인스턴스화 하고 FragmentTransaction에 add 할 수 있다.
- fragment transaction을 commit하면 생성했던 fragment instance를 사용한다.
- 하지만 configuration change에는 actgivity와 모든 fragment가 destroy되고 recreate되는데 FragmentManager가 fragment instance를 recreate하고 host에 attach하고 back stack state를 recreate한다.
  
  
## Q&A
<b id="f1">1) </b>DialogFragment를 dialog helper method 대신 사용할 때의 장점은 무엇일까? [↩](#r1)<br>
- fragment이기 때문에 fragment의 생명주기를 활용할 수 있다.
- dialog helper method는 Activity가 destroy되어도 dialog가 존재해서 leaked window 문제가 발생할 수 있다.

<b id="f2">2) </b>FragmentContainerView란 [↩](#r2)<br>
- https://developer.android.com/reference/androidx/fragment/app/FragmentContainerView
- FragmentContainerView 등장 이전에는 Activity에 Fragment를 hosting할 때 FrameLayout을 많이 사용했고 FragmentContainerView 는 FrameLayout class를 상속하는 클래스다

<b id="f3">3) </b>programmatically add 한번 더 정리 [↩](#r3)<br>
- 우선 fragment를 담을 container view 역할을 FragmentContainerView가 한다.
- fragment는 FragmentManager가 관리하기 때문에 FragmentManager instance를 얻어야 한다.
  - FragmentManager는 abstract class이고 FragmentActivity class에 구현되어 있다.
  - getSupportFragmentManager() method를 통해 FragmentManager instance를 얻는다.
- FragmentManager를 인스턴스(클래스를 바탕으로 구현된 실체, 메모리에 할당)화 한후 beginTransaction()을 통해 FragmentTransaction을 create한다.
- FragmentTransaction 클래스가 제공하는 add() method를 통해 fragment가 추가될 container view ID와 추가할 fragment class name을 추가하여 add한다.
- 마지막으로 해당 transaction(여기서는 add)를 commit하여 마무리한다.
  
<b id="f4">4) </b>SavedInstanceState를 통해 fragment가 복원될 때 activity view hierarchy와 마찬가지로 fragment view hierarchy도 전부 복원될까?[↩](#r4)<br>

<b id="f5">5) </b> 정말 이 같은 상황이라면 어떤 현상이 일어날까?[↩](#r5)<br>
  
<b id="f6">6) </b> add와 replace의 차이는? add 하지 않고 replace를 먼저 하면 어떻게 될까?[↩](#r6)<br>
  
<b id="f7">7) </b> Special considerations for child and sibling fragments[↩](#r7)<br>
- https://developer.android.com/guide/fragments/fragmentmanager#considerations

<b id="f8">8) </b>setReorderingAllowed(true)란? [↩](#r8)<br>

<b id="f9">9) </b>saveBackStack()과 restoreBackStack() 예시 [↩](#r9)<br>
- 현재 back stack에 A, B 순서대로 add 되어 있다.
- 그 위에 C, D를 다시 back stack에 add하고 C add시 "C" name 지정
- saveBackStack("C") 하면 C, D transaction이 다른 back stack으로 move
- 다시 E를 back stack에 add하고 "E" name 지정하면 A, B, E 순서가 됨
  - 여기까지 (A, B, E), (C, D) 이렇게 2개의 back stack
- 이 상황에서 saveBackStack("E") 후 restoreBackStack("C")를 하면
  - E만 따로 또 다른 back stack으로 move되고 A, B, E 모두 기존  back stack으로 restore
  - 결론적으로 (A, B, C, D), (E)가 된다.
- 즉, saveBackStack("name")을 지정하면 name부터 그 위로의 transaction이 하나의 back stack을 구성하고 다른 이름으로 saveBackStack("name2")로 하면 name2부터 그 위로의 transaction이 또 다른 back stack을 구성
- restorebackStack("name") 하면 save되었던 모든 transaction과 fragment saved sate가 restore

  

