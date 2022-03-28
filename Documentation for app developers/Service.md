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
  - blocking operation을 run하려면 ANR 방지를 위해 service 내에서 seperate thread를 만들고 그 seperate thread에서 blocking operation을 수행해야 한다.
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
  - 만약 API level 26이상을 target한다면 app이 foreground에 있지 않다면 system은 background service 실행에 restrictions을 가한다.
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
- service를 사용한다면 기본적으로 main thread에서 실행하는 것이므로 intensive or blocking operation을 수행해야 한다면 service내에서 new thread를 생성해야 한다.

### [The basics]
- service를 create하기 위해 Service subclass를 생성하거나 기존 subclass를 사용해야 한다.
- 다음은 반드시 override해야 하는 callback method다.
- onStartCommand()
  - 다른 component가 service를 시작하기를 요청할 때 startService()를 호출함으로써 system이 invoke하는 method
  - 이 method를 실행하면 service는 시작되고 background에서 실행할 수 있다.
    - 이렇게 실행하게 되면 stopSelf() or stopService()를 통해 service를 stop해야 한다.
  - 만약 binding만 원한다면 이 method를 실행할 필요 없다.
- onBind()
  - 다른 component가 service와 bind되고 싶을 때 bindService()를 호출함으로써 system이 invoke하는 method
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
- 만약 bindService()를 하고 onStartCommand()가 호출되지 않았다면 component가 bound되어 있는 동안에만 service는 실행된다.
  - service가 모든 client로부터 unbound되면 system은 service를 destroy시킨다.
- Android system이 service를 강제 중단하는 것은 memory가 부족하여 user focus를 가진 activity를 위해 system resource를 회복해야 하는 경우로만 국한된다.
  - service가 user focus를 가진 activity에 binding된 경우 종료될 가능성이 적고 foreground에서 실행되는 경우 종료될 가능성이 희박하다.
  - service가 시작되어 장시간 실행 중이라면 시간이 지나면서 background task list에서 그 위치를 낮추고 종료될 가능성이 높아진다.
    - 그렇기에 service가 시작되었다면 system에 의한 restart를 정상적으로 처리하도록 design되어야 한다.
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
  - application publish 후 이 name을 그대로 두어야 한다.
    - service를 start or bind하기 위해 explicit intent를 사용할 때 이 name에 의존하기에 name을 바꾸면 이 의존성 때문에 code가 깨질 수 있기 때문이다.
- app의 안전성을 위해 항상 explicit intent를 통해 Service를 시작하고 intent filter를 선언하지 않아야 한다.
  - implicit intent를 사용하면 어떤 service가 그 intent에 반응할 지 모르고 user는 어떤 service가 시작하는지 알 수 없기 때문에 위험하다.
  - API level 21부터 implicit intent로 bindService()를 호출하면 exception이 발생한다.
- service가 오직 같은 app 내에서만 가능하도록 하려면 android:exported attribute를 false로 설정한다.
- 사용자는 device에서 어떤 service가 실행되고 있는지 알 수 있고 그들이 인지하지 못하거나 믿지 못하는 service인 경우 service를 stop할 수 있다.
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
    - onStartCommand() return 후에 system이 service를 kill했다면 deliver할 pending intent가 있지 않다면 service를 recreate하지 않는다.
    - 이 방법은 service가 필요하지 않거나 application이 완료되지 않은 작업을 단순히 재시작할 때 service를 실행하지 않도록 하는 가장 안전한 option이다.
  - START_STICKY
    - onStartCommand() return 후에 system이 service를 kill했다면 service를 recreate하고 onStartCommand()를 호출하지만 last intent를 전달하지 않는다.
    - service를 시작하는 pending intent가 없다면 null intent와 함께 onStartCommand()를 호출한다.
      - pending intent가 있으면 그 intent를 deliver한다.
    - 이는 명령을 실행하지는 않지만 무한히 실행 중이며 작업을 기다리고 있는 media player와 같은 작업에 적합하다.
  - START_REDELIVER_INTENT
    - onStartCommand() return 후에 system이 service를 kill했다면 service를 recreate하고 service에 전달됐던 last intent와 함께 onStartCommand()를 호출한다.
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
  - onStartCommand() return 후에 system이 system memory를 복구한 후에 service를 계속 실행해야 하는 경우를 제외하고는 system은 service를 stop하거나 destroy하지 않는다. 
  - service는 반드시 stopSelf()를 통해 스스로 멈추거나 다른 component가 stopService()를 호출하여 멈춰야 한다.
- 만약 onStartCommand()를 통해 동시에 여러 request를 처리해야 하는 경우 start request 처리를 끝낸 뒤에도 service를 중단하면 안된다.
  - service를 중단하게 되면 첫번째 request가 stop하면 두번째 request를 종료시킬 수 있다.
  - 이런 문제를 해결하기 위해 stopSelf(int)를 사용하여 service를 stop하는 request가 항상 가장 최근 start request를 기준으로 하도록 해야 한다.
  - 이는 stopSelf(int)를 호출하면 stop request에 대응하는 start request ID(onStartCommand()에 전달된 startId)가 전달된다.
  - 그러면 만약 stopSelf(int)를 호출할 수 있게 되기 전에 service가 new start request를 받게 되면 ID가 일치하지 않으므로 service는 중단되지 않는다. 

### [Creating a bound service]
- bound service는 long-standing connection을 위해 bindService()를 호출하여 application components가 service에 bind되도록 한다.
  - 일반적으로 startService()를 통해 bound service를 start하는 것을 허용하지 않는다.

- 다른 component와 service가 application 내에서 interact하거나 interprocess communication을 통해 다른 application에게 현재 application의 기능을 드러낼 떄 bound service를 생성한다.
- bound service를 생성하기 위해서 service와의 소통을 위한 interface를 정의하는 IBinder를 return하는 onBind() callback method를 implement 해야 한다.
  - 그러면 다른 application components가 bindService()를 호출하여 해당 interface를 받고 service에 있는 method를 호출하기 시작할 수 있다.
- 이런 service는 오직 그 service에 bound된 application component를 위해서만 live하기 때문에 bound된 component가 없다면 system은 service를 destroy한다.
  - onStartCommand()를 통해 start한 service처럼 직접 bound service를 stop할 필요 없다.

- bound service를 시작하려면 client가 service와 어떻게 소통할지 명시하는 interface를 정의해야 한다.
  - 이 interface는 IBinder implementation이여야 하며 onBind() callback method에서 return해야 한다.
  - client가 IBinder를 받으면 이 interface를 통해 service와 interact할 수 있다.

- 여러 client가 동시에 service에 bind될 수 있다.
  - client는 service와의 interact가 끝나면 unbind를 위해 unbindService()를 호출한다.
  - 더 이상 service에 bound된 client가 없으면 system은 service를 destroy한다.
  
### [Sending notifications to the user]
- service가 실행되고 있을 때 Toast Notifications or Status Bar Notifications를 사용하여 user에게 event를 알릴 수 있다.
  - toast notification은 현재 window surface에 잠깐 나타났다가 사라지는 message
  - status bar notification은 message와 함께 status bar에 user가 action(activity start와 같은)을 하도록 선택할 수 있는 icon으로 제공

- 보통 background로 file download와 같은 작업을 마치고 user가 그에 act를 할 수 있도록 하는 데에 status bar notification이 best technique다.
  - user가 expanded view에서 notification을 선택하면 notification은 activity를 시작할 수 있다.

### [Managing the lifecycle of a service]
- service lifecycle이 activity보다는 simple하지만 user가 인식하지 못하는 background에서 작업이 될 수 있기 때문에 service가 어떻게 create되고 destroy되는지 주의를 기울여야 한다.
- service lifecycle은 다음과 같이 2가지다.
  - started service
    - 다른 component가 startService()를 통해 create된다.
    - service가 무기한 run 되고 stopSelf()를 통해 스스로 stop해야 한다.
    - 다른 component가 stopService()를 통해서 stop할 수도 있다.
    - service가 stop되면 system은 service를 destroy한다.
  - bound service
    - 다른 component(a client)가 bindService()를 통해 create된다.
    - client는 IBinder interface를 통해 service와 소통한다.
    - client는 unbindService()를 통해 연결을 닫을 수 있다.
    - 여러 client가 같은 service에 bind될 수 있고 모든 client가 unbind되면 system은 service를 destroy한다.
      - service가 스스로 stop하지 않아도 된다.
- 이 두가지 path는 완전히 분리되지 않았다.
  - 이미 startService()를 통해 시작된 service에 bind할 수도 있다.
    - 예를 들어 startService()를 통해 background music service를 Intent를 통해 시작했을 때 이후 user가 player를 조작하거나 현재 음악 정보를 얻고 싶을 때 activity가 bindService()를 통해 background music service에 bind될 수 있다.
    - 이런 경우 모든 client가 unbind되기 전까지는 stopService()나 stopSelf()가 service를 stop하지 않는다.

#### Implementing the lifecycle callbacks
```kotlin
class ExampleService : Service() {
    private var startMode: Int = 0             // indicates how to behave if the service is killed
    private var binder: IBinder? = null        // interface for clients that bind
    private var allowRebind: Boolean = false   // indicates whether onRebind should be used

    override fun onCreate() {
        // The service is being created
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // The service is starting, due to a call to startService()
        return startMode
    }

    override fun onBind(intent: Intent): IBinder? {
        // A client is binding to the service with bindService()
        return binder
    }

    override fun onUnbind(intent: Intent): Boolean {
        // All clients have unbound with unbindService()
        return allowRebind
    }

    override fun onRebind(intent: Intent) {
        // A client is binding to the service with bindService(),
        // after onUnbind() has already been called
    }

    override fun onDestroy() {
        // The service is no longer used and is being destroyed
    }
}
```
- service는 lifecycle callback method를 제공한다.
<img width="534" alt="스크린샷 2022-03-24 오후 1 41 40" src="https://user-images.githubusercontent.com/17876424/159843759-8ad96e6f-dd1d-4aee-a14c-386153347244.png">

- startService()와 bindService()에 의한 service lifeycycle을 분리했지만 어떤 방식으로 service가 시작됐든 client는 그 service에 bind될 수 있다.
  - onStartCommand()로 처음 시작된 service도 onBind()를 호출받을 수 있다.
- 이러한 method를 implement하려면 다음의 두 가지 service's lifecycle의 nested loop을 monitor해야 한다.
  - entire lifetime of a service
    - onCreate() 호출부터 onDestroy()가 return할 때까지의 시간
    - onCreate()에서 initial setup을 하고 onDestroy()에서 남은 resource를 해제한다.
    - onCreate(), onDestroy()는 startService(), bindService() 상관없이 모든 service에서 호출된다.
  - active lifetime of a service
    - startService()를 통한 onStartCommand()와 bindService()를 통한 onBind()가 호출되면서 시작한다.
    - 만약 service가 started service라면 active lifetime은 entire lifetime이 끝날 때와 같은 시간에 끝난다.
      - onStartCommand() return 후에도 service는 active하다.
      - service가 bound되어 있지 않다면 started service가 stopSelf() or stopService()를 통해 stop되면 system은 service를 destroy하고 onDestroy() callback만이 receive한다.
    - 만약 service가 bound되었다면 onUnbind()가 return되면 active lifetime이 끝난다.

## Foreground services
- foreground service는 user가 인지할 수 있는 operation을 수행한다.
  - status bar notification을 보여줘서 app이 foreground에서 어떤 task를 수행하고 있고 system resource를 소모하고 있는지 안다.
  - notification은 service가 stop되거나 foreground에서 제거되지 않으면 dismiss될 수 없다.

- Android 12(API level 31) 이상의 device는 short-running foreground service를 효율적으로 이용할 수 있도록 한다.(provide a streamlined experience)
  - 이런 device에서는 system이 foreground service와 관련된 notification을 보여주기 전에 10초를 기다린다.
  - 이에 대한 few exceptions은 'services that show a notification immediately'에 설명
- 이런 foreground service는 다음과 같은 app에서 사용한다.
  - music player app으로 notification을 통해 현재 재생되고 있는 음악을 보여준다.
  - fitness app으로 user permission을 받은 후 달리기 기록을 저장하고 notification을 통해 user가 움직인 거리를 보여준다.

- user가 app과 직접적으로 interact하지 않아도 user가 인식해야 하는 task를 수행하는 경우에는 foreground service만을 사용해야 한다.
  - 만약 minimum-priority notification을 사용하는, 충분히 낮은 중요도를 가진 action이라면 foreground service 대신 background task를 create해라.

### [Services that show a notification immediately]
- 만약 foreground service가 다음의 특징 중 최소 1개를 가진다면 Android 12 이상의 device여도 system이 service 시작 후 notification을 바로 시작한다.
  - action button을 포함한 notification과 관련된 service
  - mediaPlayback, mediaProjection, phoneCall foregroundServiceType을 가진 service
  - notification의 category attribute가 phone call, navigation, media playback과 관련된 use case를 제공하는 service
  - notification 설정 시 setForegroundServiceBehavior()에 FOREGROUND_SERVICE_IMMEDIATE을 전달하여 behavior change을 회피(display of the notification will be immediate even if the default behavior would be to defer visibility for a short time, Android 12 이상에서는 notification 보여주기 전에 10초 기다리는 behavior를 회피하도로 설정)한 service

### [Request the foreground service permission]
```Gradle
<manifest xmlns:android="http://schemas.android.com/apk/res/android" ...>

    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>

    <application ...>
        ...
    </application>
</manifest>
```
- Android 9(API level 28) 이상의 device에서는 foreground service 사용을 위해 반드시 FOREGROUND_SERVICE permission이 필요하다.
  - 이는 app이 요구하면 system이 자동으로 부여하는 normal permission이다.
### [Start a foreground service]
```kotlin
val intent = Intent(...) // Build the intent for the service
applicationContext.startForegroundService(intent)
```
- 먼저 foreground service로 service를 실행하도록 system에게 요청하기 전에 service 자체를 시작한다.
```kotlin
val pendingIntent: PendingIntent =
        Intent(this, ExampleActivity::class.java).let { notificationIntent ->
            PendingIntent.getActivity(this, 0, notificationIntent, 0)
        }

val notification: Notification = Notification.Builder(this, CHANNEL_DEFAULT_IMPORTANCE)
        .setContentTitle(getText(R.string.notification_title))
        .setContentText(getText(R.string.notification_message))
        .setSmallIcon(R.drawable.icon)
        .setContentIntent(pendingIntent)
        .setTicker(getText(R.string.ticker_text))
        .build()

// Notification ID cannot be 0.
startForeground(ONGOING_NOTIFICATION_ID, notification)
```
- service 안에서는 일반적으로 onStartCommand() 안에서 service를 foreground에서 실행하도록 요청할 수 있다.
  - startForeground()를 호출한다.
  - status bar에 나타날 notification을 구분하는 positivie integer, Notification object을 parameter로 받는다.
- status bar notification에서는 PRIORITY_LOW 이상의 priority를 사용해야 한다.
  - 만약 app이 priority가 낮은 notification을 사용한다면 system은 notification drawer에 message를 추가하여 app의 foreground service 사용에 대해 사용자에게 alert한다.

#### Restrictions on background starts
- Android 12(API level 31) 이상의 device에서는 background에서 foreground service를 시작할 수 없다.(몇 가지 예외 사항 제외)
  - 만약 몇 가지의 예외 사항을 만족하지 못한 상태에서 app이 background에서 작동할 때 foreground service를 시작하려고 한다면 ForegroundServiceStartNotAllowedException exception이 발생한다.

#### Exemptions from background start restrictions
- 다음과 같은 상황에서는 app이 background에서 작동할 때에도 foreground service를 시작할 수 있다.
  - activity와 같이 사용자가 볼 수 있는 상태에서 전환하는 상황
  - 기존 task의 back stack에 activity가 있는 경우를 제외하고 background에서 activity를 시작할 수 있는 상황
  - Firebase Cloud Messaging을 사용하여 high-priority message을 받는 상황
  - app과 관련된 UI element에 대해 작업을 수행하는 상황
    - 예를 들어 bubble, notification, widget, activity과 interact할 수 있다.
  - user가 요구한 action을 complete하기 위한 exact alarm을 invoke한 상황
  - app이 device의 현재 input method로 사용하는 app인 상황
  - app이 geofencing or activity recognition transition 관련 event를 받은 상황
  - device가 reboot된 후 broadcast receiver로부터 ACTION_BOOT_COMPLETED, ACTION_LOCKED_BOOT_COMPLETED, or ACTION_MY_PACKAGE_REPLACED intent action을 받은 상황
  - app이 broadcast receiver로부터  ACTION_TIMEZONE_CHANGED, ACTION_TIME_CHANGED, or ACTION_LOCALE_CHANGED intent action을 받은 상황
  - app이 BLUETOOTH_CONNECT or BLUETOOTH_SCAN permission을 필요로 하는 Bluetooth broadcast을 받은 상황
  - app이 device owner and profile owner와 같은 특정 system role or permission이 있는 경우
  - app이 Companion Device Manager를 사용하고 REQUEST_COMPANION_START_FOREGROUND_SERVICES_FROM_BACKGROUND permission or REQUEST_COMPANION_RUN_IN_BACKGROUND permission을 선언하 경우
  - user가 app의 battery optimization을 끈 경우
    - ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS intent action을 포함하는 intent를 보내서 system setting에서 App info page에서 위의 설정을 할 수 있다.

### [Remove a service from the foreground]
- foreground에서 service를 제거하려면 stopForeground()를 호출한다.
  - 이 method는 status bar notification을 제거할지의 여부를 가리키는 boolean을 받는다.
  - service는 계속 실행된다.
- foreground에서 실행할 때 service를 멈추면 notification은 제거된다.

### [Declare foreground service types]
- 만약 app이 Android 10(API level 29) 이상을 targetting하고 foreground service에서 location information에 접근하고자 한다면 < service > component의 attribute로 foreground service type을 location으로 지정해야 한다.
- 만약 app이 Android 11(API level 30) 이상을 targetting하고 foreground service에서 camera or microphone에 접근하고자 한다면 < service > component의 attribute로 foreground service type을 camera or microphone으로 지정해야 한다.
  - 이렇게 지정해도 Android 11에서 나왔던 access restriction은 그대로 적용된다.
- 기본적으로 runtime 때 startForeground()를 호출하면 system은 app manifest에서 정의한 각각의 service type에 대한 access를 허용한다.
  - 다음에 나올 예시들처럼 선언된 service type의 subset으로 access를 제한할 수 있다.

#### Example using location and camera
```Gradle
<manifest>
    ...
    <service ... android:foregroundServiceType="location|camera" />
</manifest>
```
- 만약 foreground service가 device의 location과 camera에 접근해야 한다면 위와 같이 AndroidManifest file에 정의한다.

```kotlin
val notification: Notification = ...;
Service.startForeground(notification, FOREGROUND_SERVICE_TYPE_LOCATION)
```
- runtime 때 foreground service가 manifest에 정의했던 type 중 일부만 access가 필요하다면 위와 같이 service의 access를 제한할 수 있다.

#### Example using location, camera, and microphone
```Gradle
<manifest>
    ...
    <service ...
        android:foregroundServiceType="location|camera|microphone" />
</manifest>
```
- 만약 foreground service가 device의 location, camera, microphone에 접근해야 한다면 위와 같이 AndroidManifest file에 정의한다.
```kotlin
val notification: Notification = ...;
Service.startForeground(notification,
        FOREGROUND_SERVICE_TYPE_LOCATION or FOREGROUND_SERVICE_TYPE_CAMERA)
```
- 위 example과 마찬가지로 runtime 때 foreground service가 manifest에 정의했던 type 중 일부만 access가 필요하다면 위와 같이 service의 access를 제한할 수 있다.

#### Add foreground service types of Work Manager workers
- app이 Work Manager를 사용하고 location, camera, microphone access를 필요로 하는 long-running worker를 사용한다면 앞서 나왔던 API level 별 location, camera, microphone foreground service type 지정 과정을 따르고 이러한 foreground service type을 worker에 사용할 수 있다.
  - FOREGROUND_SERVICE_TYPE_LOCATION, FOREGROUND_SERVICE_TYPE_CAMERA, FOREGROUND_SERVICE_TYPE_MICROPHONE

### [Restricted access to location, camera, and microphone]
- user privacy를 위해 Android 11(API level 30)부터 foreground service가 device의 location, camera, microphone에 접근할 때 limitation이 생겼다.
  - app이 background에서 실행되고 있을 때 foreground service를 시작할 때 다음과 같은 limitation이 있다.
    - ACCESS_BACKGROUND_LOCATION permission을 받았어도 foreground service는 location에 접근할 수 없다.
    - foreground service는 microphone, camera에 접근할 수 없다.

#### Exemptions from the restrictions
- limitation이 있다고 했지만 몇몇 상황에서는 background에서 foreground service를 시작했어도 app이 foreground에서 실행중일 때 location, camera, microphone information에 접근할 수 있다.
  - 같은 상황에서 foreground service type을 location으로 하고 ACCESS_BACKGROUND_LOCATION permission을 가졌다면 service는 app이 background에서 실행할 때도 location information을 항상 접근할 수 있다.

- 다음이 위에서 허용된 상황이다.
  - system component에 의해 시작된 service
  - app widget과 interact하여 시작된 service
  - notification과 interact하여 시작된 service
  - 다른, visible app에서 보낸 PendingIntent에 의해 시작된 service
  - device owner mode에서 실행중인 device policy controller를 실행하는 app에서 시작된 service
  - VoiceInteractionService를 제공하는 app에서 시작된 service
  - START_ACTIVITIES_FROM_BACKGROUND privileged permission를 가진 app에서 시작된 service

#### Determine which services are affected in your app
- app을 test할 때 foreground service를 시작해라.
  - 만약 started service가 location, microphone, camera 접근에 대한 제한이 있다면 다음의 message가 Logcat에 나타난다.
```
Foreground service started from background can not have \
location/camera/microphone access: service SERVICE_NAME
```


## Bound services overview
- bound service는 client-server interface에서 server다.
- component가 service에 bind되도록 하며 request를 전송하고 response를 받고 interprocess communication(IPC)를 수행한다.
- 일반적으로 bound service는 another application component를 제공하는 동안에만 유지되며 background에서 무한히 실행되지 않는다.

### [The basics]
- bound service는 다른 application이 bind되어 service와 interact할 수 있도록 한다.
- onBind() callback method를 반드시 implement해야 하며 이는 client가 service와 interact할 수 있게 하는 IBinder object를 return한다.

#### Binding to a started service
- startService()를 통해 무한히 실행되도록 service를 시작하고 bindService()를 통해 client가 그 service에 bind되도록 할 수 있다.
- 만약 service가 started되고 bound되었다면 모든 client가 unbind되어도 system이 service를 destroy하지 않는다.
  - 대신 stopSelf() or stopService()로 직접 stop해야 한다.
- 보통 onBind(), onStartCommand() 중 하나를 구현하지만 둘 다 구현해야 하는 경우도 있다.
  - 예를 들면 music player의 경우 service로 무한히 실행되도록 하고 binding을 제공하는 경우다.
  - activity가 music을 play할 service를 시작하고 application을 떠나도 music이 계속 play되도록 한다.
  - 이후 application으로 다시 돌아가면 activity가 service에 bind되어 다시 playback을 control할 수 있도록 한다.

- bindService()를 통해 client가 service에 bind된다.
  - 이때 반드시 service와의 connection을 monitor할 ServiceConnection을 제공해야 한다.
  - bindService()의 return value는 requested service가 존재하고 client가 그 service에 접근할 수 있는지의 여부를 나타낸다.
- Android system이 client와 service 사이의 connection을 create할 때 ServiceConnection의 onServiceConnected()를 호출한다.
  - onServiceConnected() method는 IBinder argument를 포함한다.

- 하나의 service에 동시에 여러 client가 conenct될 수 있다.
  - 하지만 system은 IBinder service communication channel을 cache한다.
  - 이 말은 system은 IBinder를 generate하는 onBind() method를 가장 처음 client가 bind될 때에만 호출된다는 뜻이다.
  - 추가적인 client가 bind되면 system은 onBind()를 다시 호출하지 않고 같은 IBinder를 전달한다.
- 마지막 client가 service로부터 unbind되면 startService()로 시작하지 않았다면 system은 service를 destroy한다.
- bound service 실행에서 가장 중요한 부분은 onBind() callback method가 return하는 IBinder interface이다.

### [Creating a bound service]
- service가 binding을 제공하기 위해서 client가 service와 interact할 수 있도록 하는 programming interface IBinder를 반드시 제공해야 한다.
- 다음은 interface를 제공하는 3가지 방법이다.
- Extending the Binder class
  - 만약 service가 own application에서만 사용가능하고 client와 same process에서 실행하면 Binder class를 extend한 interface를 만들고 onBind()에서 그 instance를 return해야 한다.
  - client는 Binder를 받아서 Binder implementation or the Service의 public method에 직접 접근해서 사용할 수 있다.
- Using a Messenger
  - 만약 서로 다른 process끼리 소통할 interface가 필요하다면 Messenger를 사용한 interface를 create한다.
  - service는 Message object에 응답하는 Handler를 정의한다.
  - 이 Handler는 client와 IBinder를 공유할 수 있는 Messenger의 기반으로 client가 Message object를 통해 service에게 command를 보낸다.
  - 게다가 client는 독자적인 Messenger를 정의할 수 있기 때문에 service는 message를 send back할 수 있다.
  - Messenger가 single thread에 모든 request를 queue해서 별도로 thread-safe하게 service를 만들 필요가 없기 때문에 interprocess communication(IPC)를 수행하기에 가장 간단한 방법이다.
- Using AIDL
  - Android Interface Definition Language(AIDL)은 object를 operating system이 이해할 수 있는 primitive로 분해하여 process 전반에 걸쳐 분류하여 IPC를 실행한다.
  - 앞서 나왔던 Messenger는 그 기반이 AIDL로 되어 있다.
  - Messenger의 경우 single thread에서 모든 client request를 queue하기에 service는 한번에 하나의 request를 받는다.
    - 하지만 만약 동시에 여러 request를 다루고 싶다면 AIDL를 직접 사용할 수 있다.
    - 이런 경우 service는 반드시 thread-safe하고 multi-threading이 가능해야 한다.
  - AIDL를 직접 사용하기 위해서는 .aidl file를 생성해야 한다.
    - Android SDK는 이 file을 사용하여 interface를 구현하고 IPC를 처리하는 abstract class를 생성한다.
    - 이 class는 service 내에서 확장할 수 있다.
  - 하지만 이런 방법은 multithreading 기능을 필요로 하고 더 복잡한 실행을 초래할 수 있기 때문에 대부분의 application에서 사용하면 안된다.

#### Extending the Binder class
- service가 오직 local application에서만 사용하고 다른 process와의 작업이 필요없다면 Binder class를 실행하면 된다.
- 다음과 같은 순서로 설정한다.
1. service에서 다음 작업 중 하나를 수행하는 Binder instance를 생성한다.
  - client가 호출할 수 있도록 public method를 포함
  - client가 service의 public method를 호출할 수 있는 Service instance를 return
  - client가 호출할 수 있는 public method를 사용하여 service에서 hosting하는 다른 class의 instance를 return 

2. onBind() callback method에서 Binder instance return
3. client에서 onServiceConnected() callback method로부터 Binder를 받아서 제공된 method를 사용하여 bound service를 호출

```kotlin
class LocalService : Service() {
    // Binder given to clients
    private val binder = LocalBinder()

    // Random number generator
    private val mGenerator = Random()

    /** method for clients  */
    val randomNumber: Int
        get() = mGenerator.nextInt(100)

    /**
     * Class used for the client Binder.  Because we know this service always
     * runs in the same process as its clients, we don't need to deal with IPC.
     */
    inner class LocalBinder : Binder() {
        // Return this instance of LocalService so clients can call public methods
        fun getService(): LocalService = this@LocalService
    }

    override fun onBind(intent: Intent): IBinder {
        return binder
    }
}
```
- LocalBinder가 getService() method를 제공하여 client가 현재 LocalService instance를 받는다.
  - 이는 client가 service의 public method를 호출할 수 있도록 한다.
```kotlin
class BindingActivity : Activity() {
    private lateinit var mService: LocalService
    private var mBound: Boolean = false

    /** Defines callbacks for service binding, passed to bindService()  */
    private val connection = object : ServiceConnection {

        override fun onServiceConnected(className: ComponentName, service: IBinder) {
            // We've bound to LocalService, cast the IBinder and get LocalService instance
            val binder = service as LocalService.LocalBinder
            mService = binder.getService()
            mBound = true
        }

        override fun onServiceDisconnected(arg0: ComponentName) {
            mBound = false
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.main)
    }

    override fun onStart() {
        super.onStart()
        // Bind to LocalService
        Intent(this, LocalService::class.java).also { intent ->
            bindService(intent, connection, Context.BIND_AUTO_CREATE)
        }
    }

    override fun onStop() {
        super.onStop()
        unbindService(connection)
        mBound = false
    }

    /** Called when a button is clicked (the button in the layout file attaches to
     * this method with the android:onClick attribute)  */
    fun onButtonClick(v: View) {
        if (mBound) {
            // Call a method from the LocalService.
            // However, if this call were something that might hang, then this request should
            // occur in a separate thread to avoid slowing down the activity performance.
            val num: Int = mService.randomNumber
            Toast.makeText(this, "number: $num", Toast.LENGTH_SHORT).show()
        }
    }
}
```
- client가 service에 bind되기 위해 ServiceConnection을 실행하고 onServiceConnected() callback을 통해 IBinder를 받는다.

#### Using a Messenger
- 만약 service가 remote process와 통신해야 한다면 Messenger를 사용하여 interface를 제공할 수 있다.
  - 이는 AIDL 사용 없이 interprocess communication(IPC)를 수행할 수 있게 한다.
- Messenger는 AIDL을 사용하는 것보다 간단하다.
  - Messenger는 service에 오는 모든 call을 queue한다.
  - pure AIDL은 service에게 simultaneous request를 전송하고 이를 받은 service는 multi-threading을 처리해야 한다.
- 다음은 Messenger를 사용하는 과정이다.
1. service는 client로부터의 call에 대한 callback을 받기 위해 Handler를 implment한다.
2. service는 Handler를 사용하여 Messenger object를 생성한다.
3. Messenger는 IBinder를 생성하고 이를 client가 받도록 onBinder()에서 return한다.
4. client는 IBinder를 사용하여 service의 Handler를 참조할 Messenger를 인스턴스화한다.
   - 이는 client가 service에게 Message object 보낼 때 사용한다.
5. service는 Handler의 handleMessage()를 통해 각 Message를 받는다.

```kotlin
/** Command to the service to display a message  */
private const val MSG_SAY_HELLO = 1

class MessengerService : Service() {

    /**
     * Target we publish for clients to send messages to IncomingHandler.
     */
    private lateinit var mMessenger: Messenger

    /**
     * Handler of incoming messages from clients.
     */
    internal class IncomingHandler(
            context: Context,
            private val applicationContext: Context = context.applicationContext
    ) : Handler() {
        override fun handleMessage(msg: Message) {
            when (msg.what) {
                MSG_SAY_HELLO ->
                    Toast.makeText(applicationContext, "hello!", Toast.LENGTH_SHORT).show()
                else -> super.handleMessage(msg)
            }
        }
    }

    /**
     * When binding to the service, we return an interface to our messenger
     * for sending messages to the service.
     */
    override fun onBind(intent: Intent): IBinder? {
        Toast.makeText(applicationContext, "binding", Toast.LENGTH_SHORT).show()
        mMessenger = Messenger(IncomingHandler(this))
        return mMessenger.binder
    }
}
```
- Handler의 handleMessage()를 통해 service로 들어오는 Message를 받고 what member로 무엇을 할지 결정한다.

```kotlin
class ActivityMessenger : Activity() {
    /** Messenger for communicating with the service.  */
    private var mService: Messenger? = null

    /** Flag indicating whether we have called bind on the service.  */
    private var bound: Boolean = false

    /**
     * Class for interacting with the main interface of the service.
     */
    private val mConnection = object : ServiceConnection {

        override fun onServiceConnected(className: ComponentName, service: IBinder) {
            // This is called when the connection with the service has been
            // established, giving us the object we can use to
            // interact with the service.  We are communicating with the
            // service using a Messenger, so here we get a client-side
            // representation of that from the raw IBinder object.
            mService = Messenger(service)
            bound = true
        }

        override fun onServiceDisconnected(className: ComponentName) {
            // This is called when the connection with the service has been
            // unexpectedly disconnected -- that is, its process crashed.
            mService = null
            bound = false
        }
    }

    fun sayHello(v: View) {
        if (!bound) return
        // Create and send a message to the service, using a supported 'what' value
        val msg: Message = Message.obtain(null, MSG_SAY_HELLO, 0, 0)
        try {
            mService?.send(msg)
        } catch (e: RemoteException) {
            e.printStackTrace()
        }

    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.main)
    }

    override fun onStart() {
        super.onStart()
        // Bind to the service
        Intent(this, MessengerService::class.java).also { intent ->
            bindService(intent, mConnection, Context.BIND_AUTO_CREATE)
        }
    }

    override fun onStop() {
        super.onStop()
        // Unbind from the service
        if (bound) {
            unbindService(mConnection)
            bound = false
        }
    }
}
```
- client에서는 service로부터 받은 IBinder를 기반으로 Messenger를 생성하고 send()를 사용하여 message를 보낸다.
- 만약 service가 client가 보낸 message에 대해 응답하는 것을 원한다면 client에서도 Messenger를 생성한다.
  - client가 onServiceConnected() callback을 받을 때 send() method를 사용하여 replyTo parameter가 있는 Messenger를 포함하는 Message를 service에게 보낸다.

### [Binding to a service]
- client 역할을 하는 application component는 bindService()를 통해 service에 bind된다.
  - Android system은 service의 onBind() method를 호출하고 service와의 interact를 위해 IBinder를 return한다.
- binding은 비동기이며 bindService()는 client에게 IBinder를 return 없이 바로 return된다.
- IBinder를 받기 위해 client는 반드시 ServiceConnection instance를 만들고 bindService()에 전달해야 한다.
  - ServiceConnection은 IBinder를 전달하는 callback method를 포함한다.
- client로부터 service를 bind하기 위해 다음의 과정을 거친다.
1. implement ServiceConnection 
  - onServiceConnected(), onServiceDisconnected() 두 개의 callback method를 override해야 한다.
  - onServiceConnected()
    - system은 onBind() method에서 return되는 IBinder를 전달하기 위해 이 method를 호출한다.
  - onServiceDisconnected()
    - service가 crash되거나 kill되는 것과 같이 예상치 못하게 service과의 connection을 잃어버린 경우 Android system은 이 method를 호출한다.
    - 이 method는 client가 unbind될 때 호출되지 않는다.
2. bindService()를 호출하고 ServiceConnection implementation을 전달한다.
  - 만약 bindService()가 false를 return 하면 client는 service와의 valid connection를 갖고 있지 않다는 것이다.
  - 하지만 이런 상황에서도 client는 반드시 unbindService()를 호출해야 한다.
    - 그렇지 않으면 client가 idle 상태인 service를 종료하지 못하게 한다.
3. system이 onServiceConnected()를 호출할 때 client가 service를 호출할 수 있고 interface에 정의된 method를 사용할 수 있다.
4. service와 disconnect하려면 unbindService()를 호출한다.
  - 만약 app이 client를 destroy시킬 때 service에게 아직 bound되어 있다면 이 destruction에 인해 client가 unbind된다.
  - service와의 interact가 끝나자마자 client를 unbind시키는 것이 좋다.
    - 이렇게 하면 idle service를 끝낼 수 있다.

```kotlin
var mService: LocalService

val mConnection = object : ServiceConnection {
    // Called when the connection with the service is established
    override fun onServiceConnected(className: ComponentName, service: IBinder) {
        // Because we have bound to an explicit
        // service that is running in our own process, we can
        // cast its IBinder to a concrete class and directly access it.
        val binder = service as LocalService.LocalBinder
        mService = binder.getService()
        mBound = true
    }

    // Called when the connection with the service disconnects unexpectedly
    override fun onServiceDisconnected(className: ComponentName) {
        Log.e(TAG, "onServiceDisconnected")
        mBound = false
    }
}
```
```kotlin
Intent(this, LocalService::class.java).also { intent ->
    bindService(intent, connection, Context.BIND_AUTO_CREATE)
}
```
- bindService()의 첫번째 parameter는 bind할 service를 명시적으로 지정한 intent
- 두번째 parameter는 ServiceConnection object
- 세번째 parameter는 binding을 위한 flag
  - service가 아직 활성화되지 않은 경우 service를 create하기 위해 보통 BIND_AUTO_CREATE를 사용해야 한다.
  - BIND_AUTO_CREATE는 binding이 존재하는 한 service를 자동으로 생성한다.
  - BIND_DEBUG_UNBIND는 unbind할 때 mismatched call에 대해 debugging help를 포함시킨다.
  - BIND_NOT_FOREGROUND는 binding할 때 target service의 process를 foreground scheduling prioirty로 올리지 않는다.
  - 0 for none

#### Additional notes
- connection이 끊어지면 발생하는 DeadObjectException 예외를 trap해야 한다.
  - 이 exception은 remote method에 의해 유일하게 발생하는 exception이다.
- objects는 여러 process에 걸친 reference counted다.
- client의 lifecycle에 맞게 binding과 unbinding을 matching시킨다.
  - 만약 activity가 visible할 때만 service와 interact해야 한다면 onStart()에서 bind하고 onStop()에서 unbind한다.
  - 만약 activity가 background에서 stop되었어도 필요하다면 onCreate()에서 bind하고 onDestroy()에서 unbind한다.
    - 이는 entire time 동안 activity가 service를 사용해야 한다는 것을 뜻한다.
    - 따라서 만약 servicd가 다른 process에 있다면 process가 무거워지며 system이 service를 중단시킬 가능성이 높아진다.
  - onResume(), onPause()는 모든 lifecycle transition에서 발생하기 때문에 사용하지 않는다.
  - 만약 여러 activities가 같은 service에 bind되어 있을 때 두 activity간의 전환이 일어난다면 다음 activity가 resume될 때 bind되기 전에 현재 activity가 pause 될 때 unbind되면서 service가 destroy되고 recreate될 것이다.

### [Managing the lifecycle of a bound service]
<img width="553" alt="스크린샷 2022-03-26 오후 6 04 32" src="https://user-images.githubusercontent.com/17876424/160232616-c2ac734a-f503-469e-860b-adfa360894c7.png">

- 모든 client로부터 unbound되면 startService()로 service를 시작하지 않았다면 Android system은 그 service를 destroy한다.
  - 따라서 purely a bound service라면 client에 binding되었는지 여부에 따라 Android system이 대신 관리해 주기 때문에 service의 lifecycle을 관리하지 않아도 된다.
- 하지만 onStartCommand() callback method를 implement한다면 이 service는 started service로 고려하기 때문에 반드시 명시적으로 service를 stop해야 한다.
  - 이런 경우 다른 client가 bound되어 있는지의 여부와 상관없이 스스로 stopSelf()하거나 다른 component가 stopService()를 호출하여 service를 stop할 때까지 run한다.
- 또한 service가 started인 상태에서 binding을 받는다면 system이 onUnbind() method를 호출할 때 선택적으로 true를 return할 수 있다.<sup id="r1">[1)](#f1)</sup>
  - 다음에 client가 service에 bind뙬 때 onRebind()가 호출되기를 원한다면 onUnbind()에서 true를 return하도록 한다.
  - onRebind()는 void를 return하지만 client는 여전히 onServiceConnected() callback에서 IBinder를 받는다.










## Q&A
<b id="f1">1) </b> service bind, unbind, rebind [↩](#r1)<br>
- 여러 client가 동시에 하나의 service에 bind될 수 있다.
  - 이렇게 동시에 bind되면 첫번째 client에 대해서만 onBind()가 호출되고 이후에는 첫번째 client에 대해 return된 IBinder를 재활용한다.
  - onUnbind() 역시 첫번째 client에 대해서만 호출된다.
  - 만약 bind되었던 client가 전부 unbind되고 바로 다시 다른 client가 bind된 경우에는 onBind(), onUnbind()가 호출된다.
    - 이는 모든 client가 unbind될 때 system이 service를 종료시키기 때문에 이후 새로운 client를 bind하는 것은 새로운 service를 시작하는 것이기 때문이다.
- bindService()에 전달하는 flag로 AUTO_CREATE를 사용해야 service를 자동으로 create해준다.
  - 0을 전달하면 service를 자동으로 create하지 않는다.
  - 물론, startService()로 started된 service에 대해서는 상관없다.
- 만약 여러 client가 동시에 하나의 service에 bind 될 때 onUnbind()를 다시 호출하고 싶다면 onUnbind()에 true를 return 해야 한다.
  - true를 return하면 onRebind()가 호출되고 onUnbind()가 호출된다.









