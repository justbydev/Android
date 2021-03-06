# View Binding
## 참고 자료
[Android Developers: View Bindig](https://developer.android.com/topic/libraries/view-binding)<br>
[Blog: Use view binding to replace findViewById](https://medium.com/androiddevelopers/use-view-binding-to-replace-findviewbyid-c83942471fc)

## Overview
- view binding은 각각의 XML layout file을 위한 binding class를 생성한다.
- binding class instance는 corresponding layout의 ID가 있는 모든 view에 대한 direct references를 갖고 있다.
- view binding은 findViewById를 대체한다.

### [Setup instructions]
```kotlin
android {
    ...
    buildFeatures {
        viewBinding = true
    }
}
```
- module-level build.gradle file에 설정

```Gradle
<LinearLayout
        ...
        tools:viewBindingIgnore="true" >
    ...
</LinearLayout>
```
- 어떤 layout file에 대해 binding class가 만들어지지 않도록 하기 위해 설정

### [Usage]
```Gradle
<LinearLayout ... >
    <TextView android:id="@+id/name" />
    <ImageView android:cropToPadding="true" />
    <Button android:id="@+id/button"
        android:background="@drawable/rounded_button" />
</LinearLayout>
```
- module이 가진 각각의 XML layout file에 대해 binding class가 생성된다.
- 각 binding class에는 root view와 ID가 있는 모든 view에 대한 direct references를 가진다.
- binding class 이름은 XMl file 이름의 Pascal case<sup id="r1">[1)](#f1)</sup>와 Binding을 마지막에 합친 것으로 생성된다.
  - 위의 예시에서 result_profile.xml이기 때문에 ResultProfileBinding이 binding class 이름이 되고 ID가 없는 ImageView를 제외한 모든 view에 대한 참조가 가능하다.
  - getRoot()를 통해 root view를 참조한다.

#### Use view binding in activities
```kotlin
private lateinit var binding: ResultProfileBinding

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ResultProfileBinding.inflate(layoutInflater)
    val view = binding.root
    setContentView(view)
}
```
```kotlin
binding.name.text = viewModel.name
binding.button.setOnClickListener { viewModel.userClicked() }
```
- Activity의 경우 onCreate()에서 다음과 같은 과정으로 view binding을 사용한다.
1) binding class의 static inflate() method를 통해 binding class instance를 생성한다.
2) getRoot()를 통해 root view에 대한 reference를 받아온다.
3) setContentView()에 root view를 넘긴다.<sup id="r2">[2)](#f2)</sup>

#### Use view binding in fragments
```kotlin
private var _binding: ResultProfileBinding? = null
// This property is only valid between onCreateView and
// onDestroyView.
private val binding get() = _binding!!

override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View? {
    _binding = ResultProfileBinding.inflate(inflater, container, false)
    val view = binding.root
    return view
}

override fun onDestroyView() {
    super.onDestroyView()
    _binding = null
}
```
```kotlin
binding.name.text = viewModel.name
binding.button.setOnClickListener { viewModel.userClicked() }
```
- fragments의 경우 onCreateView()에서 다음과 같은 과정으로 view binding을 사용한다.
1) binding class의 static inflate() method를 통해 binding class instance를 생성한다.
2) getRoot()를 통해 root view에 대한 reference를 받아온다.
3) root view를 return한다.
4) 만약 이미 layout을 inflate했다면 onViewCreated()에서 static bind() method를 사용한다.
```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
     super.onViewCreated(view, savedInstanceState)
     val binding = FragmentBlankBinding.bind(view)
     fragmentBlankBinding = binding
     binding.textViewFragment.text = getString(string.hello_from_vb_bindfragment)
}
```
5) fragments와 view lifecycle이 있고 fragments는 views를 outlive하기 때문에 onDestroyView()에서 binding class instance를 clean up 한다.<sup id="r3">[3)](#f3)</sup>

#### Provide hints for different configurations
- 만약 여러 configurations에 대해 view들을 정의할 때 그에 따라 서로 다른 view를 사용할 수도 있다.(같은 ID를 가진)
```Gradle
# in res/layout/example.xml (unchanged)
<TextView android:id="@+id/user_bio" />

# in res/layout-land/example.xml
<EditText android:id="@+id/user_bio" tools:viewBindingType="TextView" />
```
- binding class에 TextView로 생성되길 예상하지만 technical limitations으로 인해 View로 생성된다.
- 그래서 viewBindingType을 통해 원하는 attribute를 지정하게 된다.
  - compiler가 이 attribute에 따라 TextView field로 생성한다.
- 생성 시 compile time, runtime error가 발생하지 않기 위해 다음의 조건을 만족해야 한다.
  - 반드시 android.view.View를 상속받는 class여야 한다.
  - viewBindingType에 지정된 type은 반드시 tag값의 superclass여야 한다.
  - 최종 type은 반드시 모든 configurations에서 resolve consistently해야 한다.

### [Differences from findViewById]
```kotlin
public final class ActivityAwesomeBinding implements ViewBinding {
  @NonNull
  private final ConstraintLayout rootView;
  @NonNull
  public final Button button;
  @NonNull
  public final TextView subtext;
  @NonNull
  public final TextView title;
```
```kotlin
private ActivityAwesomeBinding(@NonNull ConstraintLayout rootView, @NonNull Button button,
    @NonNull TextView subtext, @NonNull TextView title) { … }
  
@NonNull
public static ActivityAwesomeBinding inflate(@NonNull LayoutInflater inflater) {
  /* Edited: removed call to overload inflate(inflater, parent, attachToParent) */
  View root = inflater.inflate(R.layout.activity_awesome, null, false);
  return bind(root);
}
```
```kotlin
@NonNull
public static ActivityAwesomeBinding bind(@NonNull View rootView) {
  /* Edit: Simplified code – the real generated code is an optimized version */
  Button button = rootView.findViewById(R.id.button);
  TextView subtext = rootView.findViewById(R.id.subtext);
  TextView title = rootView.findViewById(R.id.title);
  if (button != null && subtext != null && title != null) {
    return new ActivityAwesomeBinding((ConstraintLayout) rootView, button, subtext, title);
  }
  throw new NullPointerException("Missing required view […]");
}
```
#### Null safety
- binding class에는 ID가 있는 view에 대해서 direct references를 생성하기에 invalid view ID에 대한 null pointer exception risk가 없다.
- 만약 어떤 view가 특정 configurations에 해당하는 layout에만 존재한다면 이를 detect하여 binding class에서 @Nullable로 표시된다.

#### Type safety
- 각 binding class의 field는 binding class가 reference하는 xml file의 view와 correctly하게 type matching을 한다.

### [Comparison with data binding]
#### Benefits over data binding
- Faster compilation : annotation processing를 필요로 하지 않아 compile time이 빠르다.
- Ease of use : specially-tagged XML layout files를 필요로 하지 않고 build.gradle에 view binding를 사용가능하도록 설정만 하면 모든 layout에 대해 자동으로 binding class를 생성해준다.

#### Limitations compared to data binding
- layout variables or layout expressions를 지원하지 않아 dynamic UI content를 XML layout file에 정의할 수 없다.
- two-way data binding<sup id="r4">[4)](#f4)</sup>를 지원하지 않는다.

## include, merge tag가 있을 때 View Binding 사용
### [include tag without merge tag]
- include tag(without merge tag)가 있는 경우 tag에 id를 지정해준다.
  - 이 id는 binding class 내에 include tag layout으로 지정한 것에 대한 property를 생성하는데 필요하다.
- binding class 내에 include tag layout에 대한 object가 생성된다.
```Gradle
<!-- activity_awesome.xml -->
<androidx.constraintlayout.widget.ConstraintLayout>
    <include android:id="@+id/includes" layout="@layout/included_buttons"
</androidx.constraintlayout.widget.ConstraintLayout>
  
<!-- included_buttons.xml -->
<androidx.constraintlayout.widget.ConstraintLayout>
    <Button android:id="@+id/include_me" />
</androidx.constraintlayout.widget.ConstraintLayout>
```
```kotlin
public final class ActivityAwesomeBinding implements ViewBinding {
  ...

  @NonNull
  public final IncludedButtonsBinding includes;
```
- 위와 같이 지정한 id가 이름이 된 object가 생성된다.
- 이를 통해서 ActivityAwesomeBinding.includes.includeme와 같이 접근할 수 있게 된다.

### [include tag with merge tag]
- merge tag가 있는 경우 include tag가 eliminate 된다.
- merge tag가 있을 때 include tag에 id를 지정하고 view binding을 사용하면 다음과 같은 오류가 발생한다.
  - Missing required view with ID
```kotlin
@NonNull
public static ActivityTestBinding bind(@NonNull View rootView) {
  // The body of this method is generated in a way you would not otherwise write.
  // This is done to optimize the compiled bytecode for size and performance.
  String missingId;
  missingId: {
    View viewMerged = rootView.findViewById(R.id.view_merged);
    if (viewMerged == null) {
      missingId = "viewMerged";
      break missingId;
    }
    MergeViewBinding viewMergedBinding = MergeViewBinding.bind(viewMerged);
    return new ActivityTestBinding((LinearLayout) rootView, viewMergedBinding);
  }
  throw new NullPointerException("Missing required view with ID: ".concat(missingId));
}
```
- 따라서 merge tag가 있는 경우 다음과 같은 방법으로 view binding을 사용할 수 있다.
  - include 되는 layout에 대해 생성된 binding class에 대해 static bind()를 통해 instance를 생성한다.
  - bind() method에 pass하는 root view는 include 해주는 layout root view다.
  - 이렇게 생성된 binding class instance를 통해 접근한다.

```Gradle
/* placeholder.xml */
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android">
    
    <TextView
        android:id="@+id/tvPlaceholder"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
    
</merge>
```
```Gradle
/* result_profile */
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <include layout="@layout/placeholder" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
```kotlin
private lateinit var binding: ResultProfileBinding
private lateinit var mergebinding: PlaceholderBinding
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ResultProfileBinding.inflate(layoutInflater)
    val view = binding.root
    mergebinding = PlaceholderBinding.bind(view)
    setContentView(view)
    
    mergebinding.tvPlaceholder.text = "test"
}
```

    
## Q&A
<b id="f1">1) </b>Pacal case는 첫 단어를 대문자로 표기하고 붙여 쓰는 방식 [↩](#r1)<br>
<b id="f2">2) </b> setContentView()에 layout file ID를 넘기지 않고 root view를 넘기는 이유
```kotlin
@Override
public void setContentView(View v) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    contentParent.addView(v);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}

@Override
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```
- Easy Mistake: Calling setContentView(…) with the layout resource id instead of the inflated binding object is an easy mistake to make. This causes the layout to be inflated twice and listeners to be installed on the wrong layout object. 
- setContentView()는 view hierarchy를 inflate 시키고 activity content로 setting. [↩](#r2)<br>

<b id="f3">3) </b> onDestroyView()에서 _binding = null 한 이유[↩](#r3)<br>
- Fragment view는 종료되고 Fragment는 남아있어 후에 재사용할 때 다른 view를 사용할 수 있다.
  - Fragment가 남아있고 _binding = null 하지 않으면 _binding은 모든 inflated view에 대한 접근을 갖고 있게 되어 garbage collected되는 것을 막아 memory leak이 발생할 수 있다.
- 그렇다면 왜 private var _binding, private val binding 2개의 property를 사용했을까?

<b id="f4">4) </b> data 변화가 view에 적용되는 것 뿐 아니라 user update를 listen해서 user update에 의한 view 변화가 역으로 data에 적용되는 것 [↩](#r4)<br>


## 추가 정리
[추가 Q&A](https://github.com/justbydev/Android/issues/3)
















 
