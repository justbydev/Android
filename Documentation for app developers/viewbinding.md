# View Binding
## 참고 자료
[Android Developers: View Bindig](https://developer.android.com/topic/libraries/view-binding)

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



## Q&A
<b id="f1">1) </b>Pacal case는 첫 단어를 대문자로 표기하고 붙여 쓰는 방식[↩](#r1)<br>





















 
