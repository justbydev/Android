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
- started service는 다른 component가 startService()를 호출하여 시작하는 service로 onStartCommand()가 호출된다.
  - Intent를 통해 특정 service를 명시하고 service가 사용할 data를 포함할 수 있다.
  - onStartCommand()가 Intent를 받는다.
- service가 시작하면 그 service를 시작한 component와 독립적인 lifecycle을 가진다.
  - component가 destroy되어도 background에서 계속 run될 수 있다.
  - 이런 경우 stopSelf()를 통해 직접 끝내거나 다른 component가 stopService()를 통해 끝내야 한다.
- Service class가 모든 service의 base class다.
  - 기본적으로 main thread를 사용하므로 activity performance가 느려지지 않도록 new thread를 create하는 것도 중요하다.
- Android framework는 worker thread를 사용하도록 IntentService를 제공한다.
  - 이는 한번에 하나씩만 처리한다.
  - Background execution limits 때문에 Android 8 Oreo 부터는 제대로 동작하지 않을 수 있어서 추천하지는 않는다.
  - 또한 Android 11부터는 depracted되었다.
  - IntentService 대신 JobIntentService를 사용할 수 있다.

#### Extending the Service class
```kotlin
class HelloService : Service() {

    private var serviceLooper: Looper? = null
    private var serviceHandler: ServiceHandler? = null

    // Handler that receives messages from the thread
    private inner class ServiceHandler(looper: Looper) : Handler(looper) {

        override fun handleMessage(msg: Message) {
            // Normally we would do some work here, like download a file.
            // For our sample, we just sleep for 5 seconds.
            try {
                Thread.sleep(5000)
            } catch (e: InterruptedException) {
                // Restore interrupt status.
                Thread.currentThread().interrupt()
            }

            // Stop the service using the startId, so that we don't stop
            // the service in the middle of handling another job
            stopSelf(msg.arg1)
        }
    }

    override fun onCreate() {
        // Start up the thread running the service.  Note that we create a
        // separate thread because the service normally runs in the process's
        // main thread, which we don't want to block.  We also make it
        // background priority so CPU-intensive work will not disrupt our UI.
        HandlerThread("ServiceStartArguments", Process.THREAD_PRIORITY_BACKGROUND).apply {
            start()

            // Get the HandlerThread's Looper and use it for our Handler
            serviceLooper = looper
            serviceHandler = ServiceHandler(looper)
        }
    }

    override fun onStartCommand(intent: Intent, flags: Int, startId: Int): Int {
        Toast.makeText(this, "service starting", Toast.LENGTH_SHORT).show()

        // For each start request, send a message to start a job and deliver the
        // start ID so we know which request we're stopping when we finish the job
        serviceHandler?.obtainMessage()?.also { msg ->
            msg.arg1 = startId
            serviceHandler?.sendMessage(msg)
        }

        // If we get killed, after returning from here, restart
        return START_STICKY
    }

    override fun onBind(intent: Intent): IBinder? {
        // We don't provide binding, so return null
        return null
    }

    override fun onDestroy() {
        Toast.makeText(this, "service done", Toast.LENGTH_SHORT).show()
    }
}
```
- 위 예시에서 모든 call은 onStartCommand()가 관리하고 background thread의 Handler에서 작업을 게시한다.
  - IntentService처럼 이루어지며 모든 request가 순서대로 진행된다.
  - 여러 request가 동시에 이루어지길 원하면 thread pool을 사용할 수 있다.
- onStartCommand()의 경우 반드시 integer를 return한다.
  - 이 값은 system이 service를 kill한 경우 system이 어떻게 service를 continue할지를 설명하는 값이다.
  - START_NOT_STICKY
    - system이 service를 kill한 후 deliver할 pending intent가 있지 않다면 recreate하지 않는다.
    - 이 방법은 필요하지 않거나 application이 unfinished job을 단순하게 restart할 때 안전한 option이다.
  - START_STICKY
    - system이 service를 kill한 후 service를 recreate하고 onStartCommand()를 호출하지만 last intent를 전달하지 않는다.
    - service를 시작하는 pending intent가 없다면 null intent와 함께 onStartCommand()를 호출한다.
      - pending intent가 있으면 그 intent를 deliver한다.
    - 이는 명령으 실행하지는 않지만 무한히 실행 중이며 작업을 기다리고 있는 media player에 적합하다.
  - START_REDELIVER_INTENT
    - system이 service를 kill한 후 service를 recreate하고 service에 전달됐던 last intent와 함께 onStartCommand()를 호출한다.
    - 모든 pending intent가 차례로 전달된다.
    - file downloading과 같이 바로 다시 시작해야 하는 job에 적합하다.

#### Starting a service
```kotlin
Intent(this, HelloService::class.java).also { intent ->
    startService(intent)
}
```
- startService()나 startForegroundService()에 Intent를 전달하여 다른 component에서 service를 시작할 수 있다.
- API level 26 이상부터 app이 foreground에 없으면 background service를 사용하거나 생성하는데 제한이 있다.
  - 만약 foreground service 생성이 필요하면 startForegroundService()를 사용한다.
    - 이는 background service를 생성하지만 system에게 service는 foreground에서 진행될 것이라는 것을 알린다.
    - 생성된 후 5초 이내에 service는 startForeground()를 호출해야 한다.
- startService()는 바로 return되고 Android system은 service의 onStartCommand()를 호출한다.
  - 만약 실행중이던 service가 없다면 onCreate()를 먼저 호출한 후 onStartCommand()를 호출한다.
- 만약 binding을 제공하지 않으면 startService() 함께 전달된 intent가 application component와 service의 유일한 통신 수단이다.
  - 하지만 만약 service가 result를 돌려보내기를 원한다면 service를 시작하는 client가 getBroadcast()를 통해 broadcast를 위한 PendingIntent를 만들고 이렇게 만든 PendingIntent를 service를 시작하는 Intent를 통해 service에게 전달한다.
    - 그러면 service는 이 broadcast를 사용하여 result를 전달한다.

#### Stopping a service
- started service는 반드시 own lifecycle을 관리해야 한다.
  - system memory를 회복해야 하고 onStartCommand() return 후에도 계속 실행되는 경우를 제외하고는 system은 servic를 stop하거나 destroy하지 않는다.
  - service는 반드시 stopSelf()를 통해 스스로 멈추거나 다른 component가 stopService()를 호출하여 멈춰야 한다.
- 만약 onStartCommand()를 통해 동시에 여러 request를 처리해야 하는 경우 start request 처리를 끝낸 뒤에도 service를 중단하면 안된다.


  
  
## Q&A
<b id="f1">1) </b> background service restrictions의 예외 사항이 있을까? [↩](#r1)<br>

<b id="f2">2) </b> worker thread를 만들었는데 그 속에서 servicd를 수행할 필요가 있을까? [↩](#r2)<br>








