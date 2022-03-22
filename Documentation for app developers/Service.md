# Service
## 참고 자료
[Android Developers Core topics: Services](https://developer.android.com/guide/components/services)

## Services overview
- Service는 background에서 long-running operations를 수행하는 application component다.
  - user interface를 제공하지 않는다.
- 한번 시작하면 사용자가 다른 app으로 switch해도 service는 계속 running한다.
- 또한, service와 interact할 component는 service에 bind할 수 있으며 interprocess communication(IPC)를 수행할 수도 있다.
  - network transactions, play munic, perform file I/O, interact with a content provider
    - 이 모든 것을 background에서 수행할 수 있다.
- service는 직접 지정하지 않는 한 own thread나 seperate process가 아니라 hosting process의 main thread에서 run한다.
  - blocking operation을 run하려면 ANR 방지를 위해 seperate thread에서 service를 run해야 한다.
### [Types of Services]
- Foreground
  - foreground service는 user에게 noticable하다.
    - play an audio track
  - 사용자가 app과 interact하지 않아도 계속 running한다.
  - 반드시 Notification을 나타내야 한다.
    - foreground service는 notification을 통해서 현재 service가 running 중임을 사용자에게 알려야 한다.
    - service가 foreground로부터 stop되거나 remove되지 않으면 notification은 사라질 수 없다.
- Background
  - background service는 user가 직접적으로 알지 못하는 operation을 수행한다.
    - compact app's storage
  - 만약 API level 26이상을 target한다면 app이 foreground에 있지 않다면 system은 background service 실행에 restrictions을 가한다.<sup id="r1">[1)](#f1)</sup>
- Bound
  - bindService()를 통해 application component에 bind되는 service
  - bound service는 client-server interface를 제공하여 components가 service와 interact하고 request 전송하고 result를 받을 수 있도록 하며 interprocess communication(IPC)가 가능하도록 한다.
  - 다른 application component가 bound되어 있는 한 bound service는 run한다.
  - 여러 component가 하나의 service에 bind될 수 있고 모든 component가 unbind되면 service는 destroy된다.

- 위처럼 started, bound service를 분류해서 정리되어 있지만 하나의 service가 start되고 binding도 될 수 있다.
  - 이는 단지 어떤 callback method를 실행할 것인지의 차이다.
    - onStartCommand()는 component가 service를 시작하도록 한다.
    - onBind()는 component가 service에 bind되도록 한다.
- service가 start되든 bound되든 혹은 둘다 되었든 any application component에서 Intent를 사용해서 activity를 사용할 수 있듯이 service도 마찬가지다.
  - 어떤 application component에서든지 Intent를 통해 service를 사용할 수 있지만 manifest file에서 service를 private으로 선언해서 다른 application의 접근을 막을 수 있다.

### [Choosing between a service and a thread]
- service는 user가 application과 interact하지 않아도 background에서 실행 가능한 component이기 때문에 필요할 때에만 service를 create해야 한다.
- user가 application과 interact하는 동안 main thread 밖에서 작업을 수행해야 하는 경우 new thread를 create해야 한다.
  - 예를 들어 activity가 running하는 동안에만 음악 재생을 원한다면 onCreate()에서 thread를 생성해서 onStart()에서 thread를 running하고 onStop()에서 stop해야 한다.
    - 전통적인 Thread 대신 thread pool이나 java.util.concurrent package의 executors를 사용하거나 Kotlin coroutines을 사용할 수도 있다.
- service를 사용한다면 기본적으로 main thread에서 실행하는 것이므로 intensive or blocking operation을 수행해야 한다면 new thread를 생성해야 한다.<sup id="r2">[2)](#f2)</sup>

### [The basics]
- service를 create하기 위해 Service subclass를 생성하거나 기존 subclass를 사용해야 한다.
- 다음은 반드시 override해야 하는 callback method다.
- onStartCommand()
  - 다른 component가 service를 시작하기를 요청할 때 startService()를 호출함으로써 system이 invoke하는 method
  - 이 method를 실행하면 service는 시작되고 background에서 실행할 수 있다.
    - 이렇게 실행하게 되면 stopSelf() or stopService()를 통해 service를 stop해야 한다.
  - 만약 binding만 원한다면 이 method를 실행할 필요 없다.
- onBind()
  - 다른 component가 service와 bind되고 싶을 때 bindSercie()를 호출함으로써 system이 invoke하는 method
  - IBinder를 return해서 service와 client가 소통할 수 있는 interface를 제공해야 한다.
  - 만약 binding를 허용하고 싶지 않다면 null을 return해야 한다.
- onCreate()
  - onStartCommand()나 onBind()를 통해 service가 처음 시작할 때 one-time setup 과정을 수행할 때 system이 invoke하는 method
  - 만약 service가 이미 실행중이면 이 method는 호출되지 않는다.
- onDestroy()
  - service가 더 이상 사용되지 않고 destroy되었을 때 system이 invoke하는 method
  - thread, registered listeners, receivers 등 any resource를 clean up할 때 사용해야 한다.
  - 이는 service가 수신하는 마지막 call이다.
- 만약 component가 startService()를 하여 service를 시작했다면 스스로 stopSelf()를 하거나 다른 component가 stopService()를 할 때까지 service는 지속된다.
- 만약 bindService()를 하고 onStartCommand()가 호출되지 않았다면 component가 bound되어 있는 한 service는 실행된다.
  - service에 bound된 모든 clients가 unbound되면 system은 service를 destroy시킨다.
- Android system이 service를 강제 중단하는 것은 memory가 부족하여 user focus를 가진 activity를 위해 system resource를 회복해야 하는 경우로만 국한된다.
  - service가 user focus를 가진 activity에 binding된 경우 종료된 가능성이 적고 foreground에서 실행되는 경우 종료될 가능성이 희박하다.
  - service가 시작되어 장시간 실행 중이라면 시간이 지나면서 background task list에서 그 위치를 낮추고 종료될 가능성이 높아진다.
    - service가 시작되었다면 system에 의한 restart를 정상적으로 처리하도록 design되어야 한다.
  - system이 service를 중단하는 경우 resource를 다시 사용할 수 있게 되면 가능한 한 빨리 service가 다시 시작된다.
    - 다만 이는 onStartCommand()에서 return되는 값에 따라 다르다.

#### Declaring a service in the manifest
```Gradle
<manifest ... >
  ...
  <application ... >
      <service android:name=".ExampleService" />
      ...
  </application>
</manifest>
```
- 모든 service는 manifest file에 선언해야 한다.
- < service > element에는 service를 시작할 때 필요한 permission과 어떤 process에서 실행해야 하는지를 정의하는 attribute를 포함할 수 있다.
  - android:name attribute만이 required attribute다.
  - application publish 후 이 name을 그대로 두어야 service가 start or binding할 explicit intent에 대한 종속성 때문에 코드가 깨질 위험을 막을 수 있다.
- app의 안전성을 위해 항상 explicit intent를 통해 Service를 시작하고 intent filter를 선언하지 않아야 한다.
  - implicit intent를 사용하면 어떤 service가 그 intent에 반응할 지 모르고 user는 어떤 service가 시작하는지 알 수 없기 때문에 위험하다.
  - API level 21부터 implicit intent로 bindService()를 호출하면 exception이 발생한다.
- service가 오직 같은 app 내에서만 가능하도록 하려면 android:exported attribute를 false로 설정한다.
- 만약 사용자가 실수로 service를 stop하는 것을 방지하려면 < service > element에 android:description을 추가하여 이 service가 무엇이며 어떤 benefits을 제공하는지에 대한 short sentence를 제공한다.

### [Creating a started service]


  
  
## Q&A
<b id="f1">1) </b> background service restrictions의 예외 사항이 있을까? [↩](#r1)<br>

<b id="f2">2) </b> worker thread를 만들었는데 그 속에서 servicd를 수행할 필요가 있을까? [↩](#r2)<br>









