# Activities
## 참고자료
[Android Developers: Activities](https://developer.android.com/guide/components/activities/intro-activities)
## Introduction to activities
### [The concept of activities]
- user interaction에 따라 항상 같은 곳에서 app이 시작되는 것이 아니다.
  - 이런 paradigm을 적용하여 design된 것이 Activity class
  - one app이 다른 app을 invoke할 때 activity를 invoke해서 app을 calling한다.
  - 그렇기 때문에 activity는 app과 user간의 interaction의 entry point다.
- activity는 app이 UI를 그리는 window를 제공한다.
  - 이 window는 full screen일 수도 있고 screen보다 작을 수도 있다.
- 대부분의 app은 여러 screen으로 구성되어 있고 이것은 여러 activities로 구성되어 있다는 뜻이다.
  - 일반적으로 app에서 하나의 activity는 main activity로 지정되며 이는 user가 app을 launch할 때 처음 나타나는 screen이다.
  - 각각의 activity는 다른 action을 하기 위해 다른 activity를 시작할 수 있다.
- activities들은 일관된 user experience 형성을 위해 함께 작동하지만 각 activity는 다른 activity와 loosely bound되어 있다.
  - 다른 activities들과 최소의 종속성을 가진다.
  - 실제로 activities들은 종종 다른 app의 activities를 시작한다.
- activities를 상요하려면 manifest에 등록해야 하고 반드시 activity lifecycles를 제대로 관리해야 한다.

### [Configuring the manifest]
#### Declare activities
```Gradle
<manifest ... >
  <application ... >
      <activity android:name=".ExampleActivity" />
      ...
  </application ... >
  ...
</manifest >
```
- 필수적으로 필요한 attribute는 activity class name을 나타내는 android:name이다.

#### Declare intent filters
```Gradle
<activity android:name=".ExampleActivity" android:icon="@drawable/app_icon">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```
- Intent filters를 지정하게 되는데 explicit request뿐 아니라 implicit request로 activity를 launch할 수 있도록 한다.
  - < action >은 =














