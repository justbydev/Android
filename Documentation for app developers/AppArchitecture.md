# Guide to App Architecture
## 참고자료
[Android Developers: Guide to app architecture](https://developer.android.com/jetpack/guide)

## OverView
### [Mobile app user experiences]
- 일반적인 안드로이드 앱은 여러 app component(activity, fragment, service 등 app manifest에 등록)로 구성되어 있다.
- resource가 제한적이라 언제든지 OS가 app process를 종료시킬 수 있다.
  - OS나 user는 언제든지 app component를 destroy시킬 수 있다.
  - 이런 이벤트는 직접 제어할 수 없기에 app data, state를 app component에 저장해서는 안된다.
### [Common architectural principles]
#### Seperation of concerns
- Activity or Fragment에 모든 코드를 작성해서는 안된다.
  - Android OS와 app 사이를 이어주는 glue 역할일 뿐이다.
  - user interactions나 메모리 부족과 같은 시스템 조건으로 언제든지 제거될 수 있기 때문에 의존성을 높여서는 안된다.
#### Drive UI from data models
- Data model은 UI를 도출해야 한다.
- app data를 가지는 data model은 View와 같은 UI elements, app component와 독립적이어야 한다.
  - app component lifecycle에 영향을 안받을 수 있다.
  - 하지만 OS가 app process를 memory로부터 제거하면 data model도 destroy될 수 있다.
- 그래서 persistent model이 이상적이다.
  - app process가 제거되어도 data가 유지될 수 있다.
  - network 환경이 좋지 않아도 app을 사용할 수 있다.
### [Recommended app architecture]
- UI layer + (Domain layer) + Data layer
#### UI layer
- app data를 screen에 보여주는 역할
- 항상 최신의 상태를 반영해서 screen에 보여주어야 한다.
- UI elements + State holders
  - UI elements : Data를 screen에 rendering하는 역할로 View or Jetpack Compose functions를 사용
  - State holders : UI state를 갖고 그와 관련된 logic을 다루는 역할로 ViewModel class를 사용
#### Data layer
- app이 data를 어떻게 생성, 저장, 변화시키는지에 대한 business logic을 구성하는 역할
  - app data CRUD
- Repositories + Data Sources
#### Domain layer
- complex business logic이나 여러 ViewModels에서 reuse할 수 있는 business logic을 구성하는 역할
- 필수적인 layer는 아니고 필요에 의해 UI layer와 Data layer 사이에 사용되는 optional layer
### [General best practices]
- app components에 app data를 저장하지 말자.
  - app components는 생명주기가 짧기 때문에 sources of data로 지정하면 안된다.
  - user interaction이나 system 상황에 따라 짧게 종료될 수 있다.
- Android class에 대한 의존성을 줄이자.
  - Context, Toast와 같은 Android framework SDK APIs들을 사용하는 것은 오직 app component만이여야 한다.
  - 테스트 하기 용이해지며 coupling을 줄일 수 있다.
- 다양한 app module간의 책임을 명확히 구분하자.
  - 관련된 역할을 하는 것은 분산시키지 말고 관련없는 역할을 하는 코드는 같은 곳에 정의하지 말아야 한다.
- 각 모듈에서 가능한 작게 노출하자.
- 다른 앱과 차별되는 app의 unique core에 집중하자.
  - 같은 보일러플레이트 코드를 반복 작성하면 안된다.
  - 반복되는 보일러플레이트 코드는 여러 libraries에게 맡기자.
- 독립적으로 테스트할 수 있도록 하자.
  - 같은 역할을 하는 logic을 여러 곳에 분산시키면 테스트 하기 어렵다.
  - 다른 모듈에서 오는 logic을 한 곳에 mix하는 거 역시 테스트 하기 어렵게 만든다.
- 가능한 한 관련성이 높은 최신 데이터를 보존하자.
  - 이렇게 하면 오프라인 모드에서도 앱의 기능을 이용할 수 있다.

## UI layer
- UI는 data layer로부터 받은 app state의 시각적 표현이다.
- UI layer는 UI가 화면에 display할 수 있도록 app data를 변환시켜주는 pipeline이다.
  - data layer로부터 받은 data는 display를 위한 format과 다를 수 있다.
  - 2개 이상의 data sources로부터의 data를 merge해야 하는 경우도 있다.
- UI elements + State holders
### [UI layer architecture]
- UI는 결국 activity, fragment와 같은 UI elements를 가리킨다.
- UI layer의 기본적인 역할은 다음과 같다.
  - Data layer로부터 받은 app data를 UI가 쉽게 rendering할 수 있도록 변환한다.
  - user가 볼 수 있는 UI elements(View or jetpack compose)에 담을 수 있도록 변환한다.
  - UI elements는 user input events를 받고 그 영향으로 바뀌는 UI data를 반영한다.
### [Define UI state]
- UI가 user가 보는 시각적 표현이라면 UI state는 app이 현재 어떻게 보여야 하는지를 나타내는 것
  - UI state의 변화는 바로 UI에 적용되어야 한다.
- UI elements(View or jetpack compose) + UI State = UI
```kotlin
data class NewsUiState(
    val isSignedIn: Boolean = false,
    val isPremium: Boolean = false,
    val newsItems: List<NewsItemUiState> = listOf(),
    val userMessages: List<Message> = listOf()
)

data class NewsItemUiState(
    val title: String,
    val body: String,
    val bookmarked: Boolean = false,
    ...
)
```
#### Immutability
- 변하지 않는 어느 한 순간의 상태를 보장한다.
  - 변하지 않는 어느 한 순간의 스냅샷
  - 이런 불변성을 통해 UI는 state을 읽어 UI elements를 update한다.
  - UI를 통해서 UI state를 직접 modify할 수 없다.
    - UI가 직접 data를 바꾸게 되면 여러 요소들간의 data inconsistency가 발생하고 bug가 발생할 수 있다.
#### Naming conventions
- 기능 + UiState
### [Manage state with Unidirectional Data Flow]
- 앞서 UI State는 불변성을 가진다 했지만 시간이 지남에 따라 user interaction이나 다른 events에 의해 modify될 수 있다.
- UI가 UI state 변경에 대한 로직을 담당해서는 안된다.
  - UI가 이런 logic을 담당하게 되면 점점 복잡해져 logic간 뚜렷한 경계 없이 coupling되어 testability에 영향을 끼치게 된다.
  - 오직 UI state를 받고 display하는 역할만 해야 한다.
  - UI state에 대한 로직은 state holders가 갖고 있어야 하고 이렇게 UI와 state holder의 역할을 확실히 분리해주는 역할을 하는 것이  Unidirectional Data Flow(UDF) pattern
#### State holders
- UI state를 생산하고 이 작업을 위한 necessary logic을 담고 있는 것으로 ViewModel이 담당
- UI와 State holders(ViewModel class)간의 interation을 정의하는 것이 UDF
  - event input과 그와 관련된 state output 간의 관계
  - state은 ViewModel에서 UI elements로 flows down
  - event는 UI elements에서 ViewModel로 flows up
#### Unidirectional Data Flow(UDF)
- ViewModel은 Data layer로부터 app data를 받는다.
- 받은 app data를 ViewModel은 현재 UI 상황의 필요성에 맞게 변환시키고 그 결과가 UI State
- UI는 UI State를 받아 UI를 구성
- UI는 user events를 ViewModel에게 알린다.
- ViewModel은 user events를 파악해 UI state를 update
- updated UI State는 다시 UI에게 전달되어 rendering된다.
  - 이 과정을 반복하고 UI State는 ViewModel에서 UI로만, UI events는 UI에서 ViewModel로만 전달된다.
<img width="537" alt="스크린샷 2022-02-16 오후 6 31 30" src="https://user-images.githubusercontent.com/17876424/154235850-5382b9d0-cc60-4f36-90ae-114ca8438d5f.png"><br>
#### Types of logic
- Business logic
  - what to do with state changes
  - Domain or Data layers에만 존재해야 한다.
- UI behavior logic(UI logic)
  - how to display state changes on the screen
  - 현재 화면에 UI state가 어떻게 보여야 하는지에 대한 logic
  - Context와 관련된 것이나 Toast, Snackbar 등과 같은 경우는 UI에 직접 구성
  - 조금 더 복잡한 UI State에 대한 것이라면 ViewModel에 구성
#### Why use UDF?
- 데이터 일관성이 유지된다.
  - single source of truth for the UI
- UI state를 UI와 분리시켜 별개로 테스트 가능
- user event와 data source의 영향으로 인한 UI state의 변환이 well-defined pattern을 따르게 한다.

### [Expose UI state]
- 생성된 UI State를 어떻게 UI에 드러낼지에 대한 것(방법에 관한 것)
- UI state 변환에 대해 observe하여 UI에 바로 반영되어야 한다.
  - observable data holder를 사용(LiveData or StateFlow)
  - 이를 사용하면 UI가 ViewModel에서 UI state를 가져와 처리할 필요 없이 UI state가 변화하자 마자 바로 반응하여 UI를 update할 수 있다.
  - 항상 최신의 상태를 유지한다.
  - 또한 immutable stream으로 mutable stream을 드러내서 ViewModel에서만 UI state를 변경할 수 있도록 하고 드러내느 요소로는 변경할 수 없도록 한다.
```kotlin
class NewsViewModel(...) : ViewModel() {

    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    ...

}
```
- UI state는 서로 연관된 것, 결합이 필요하 것들끼리 하나의 stream으로써 handle 하자.
  - 서로 연관없는 UI State는 독립적으로 구성해야 한다.
```kotlin
data class NewsUiState(
    val isSignedIn: Boolean = false,
    val isPremium: Boolean = false,
    val newsItems: List<NewsItemUiState> = listOf()
)

val NewsUiState.canBookmarkNews: Boolean get() = isSignedIn && isPremium
```
### [Consume UI State]
- UI State를 실제 UI에 반영하는 것
- LiveData의 observe(), Kotlin flows의 collect() 사용
- UI State를 반영할 때 주의해야 할 것은 UI의 lifecycle
  - LiveData는 LifecycleOwner를 사용하여 View가 활성화되었을 때만 observe
  - Kotlin flows는 알맞은 coroutine scope를 사용
  - 결국 최소 STARTED 상태일 때 UI State를 관찰하고 UI에 반영해야 한다.
```kotlin
class NewsActivity : AppCompatActivity() {

    private val viewModel: NewsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        ...

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect {
                    // Update UI elements
                }
            }
        }
    }
}
```
### [Threading and concurrency]
- ViewModel에서의 work는 main-safe해야 한다.
- 만약 ViewModel이 long-running operations을 한다면 background thread로 옮겨서 실행해야 한다.
  - Kotlin coroutines를 사용하는 것이 great way
### [Navigation]
- Navigation 변경은 event에 의해 이루어진다.
### [Paging]
- Paging Data는 계속 변하는 데이터이기 때문에 immutable UI state로 나타내면 안된다.
### [Animations]
- screen이 data를 모두 load한 후 animation이 시작되어야 한다.
- postponeEnterTransition()으로 연기하고 모두 load되면 startPostponedEnterTransition()으로 animation을 시작한다.

## Data layer
- 앞서 UI layer가 UI state와 UI logic으로 구성되었다면 app data와 business logic은 Data layer가 담고 있다.
### [Data layer architecture]
- Repositories + Data Sources
- Repository는 다음과 같은 역할을 한다.
  - Rest of app에 App data를 노출
  - data 변화를 한곳에 집중
  - 여러 data source간의 충돌을 해결
  - Rest of app에서 data source를 추상화한다.
  - business logic을 포함한다.
- Data source는 하나의 source of data만을 사용한다.
  - file, network source, local db 등 하나의 source of data만을 나타낸다.
  - 오직 repository만이 접근할 수 있다.
- Data layer에서 노출시킨 data는 다른 class에서 변경할 수 없도록 immutable해야 한다.
```kotlin
class ExampleRepository(
    private val exampleRemoteDataSource: ExampleRemoteDataSource, // network
    private val exampleLocalDataSource: ExampleLocalDataSource // database
) { /* ... */ }
```
### [Expose APIs]
- Data layer의 class는 CRUD나 app data 변화에 대한 functions를 가진다.
- 위와 같은 functions가 가진 operation에 대한 결과나 data chaneging을 notify하기 위해서는 Kotlin에서는 suspend function, flows, Java에서는 RxJava나 callback을 사용한다.
### [Naming conventions]
- repository : type of data + Repository(예시) NewsRepository)
- data source : type of data + type of source + DataSource(예시) NewsRemoteDataSource, NewsNetworkDataSource)
### [Multiple levels of repositories]
- 한 repository는 다른 repositories에 의존성을 가질 수 있다.
  - 예를 들어 authentication data에 관련된 UserRepository는 LoginRepository, RegistrationRepository에 의존성을 가진다.
  - 이렇게 다른 repository에 의존성을 가질 때 repository 대신 UserManager와 같이 Manager로 명명하기도 한다.
<img width="500" alt="스크린샷 2022-02-16 오후 8 12 02" src="https://user-images.githubusercontent.com/17876424/154253151-21af354f-cf72-461e-987d-4bab7fbae9b9.png"><br>
### [Source of truth]
- 각 repository는 single source of truth를 정의해야 한다.
  - source of truth는 data source가 될 수 있다.
  - database, cache in memory등이 source of truth일 수 있다.
  - 예를 들어 LoginRepository는 cache를, PaymentsRepository는 network data source를 source of truth로 사용
  - 오프라인 우선 지원을 제공하려면 local data source를 source of truth로 사용하는 것이 좋다.
### [Threading]
- UI layer와 마찬가지로 main-safe해야 한다.
  - long-running blocking operations라면 로직에 알맞은 thread에서 이루어져야 한다.
  - Room or Retrofit 같은 API가 main-safe API를 제공한다.
### [Lifecycle]
- 필요한 scope에 맞게 lifecycle을 고려해야 한다.
  - 어떤 class가 in-memory data(cache)를 갖고 있고 일정 시간동안 재사용한다면 그 class instance에 맞는 lifecycle로
  - whole application에서 필요로 한다면 Application class의 lifecycle로
  - 어느 일정 flow에서만 필요로 한다면 그 flow가 가진 lifecycle로
### [Represent business models]
- data model은 app data을 담은 model data class
- Data source가 받는 app data가 app에서 전부 필요로 하지 않는 경우가 있다.
  - 필요한 data만으로 구성된 model class를 따로 분리해서 생성
  - 필요한만큼만 보관하여 app memory save 가능
  - 외부 데이터 유형을 앱에서 사용하는 데이터 유형에 맞게 조정
  - 필요한 요소들로만 구성된 model class가 미리 구성되어 있으면 역할을 쉽게 나눌 수 있다.

### [Types of data operations]
#### UI-oriented operations
- 오직 user가 보고 있는 specific screen과 관련된 operations
- UI layer에 의해 triggered되고 예를 들면 ViewModel의 lifecycle을 따른다.
- 예를 들면 user가 screen을 켜면 latest news를 볼 수 있도록 network를 통해 fetch하는 것
  - data source와 repository를 create한다.
```kotlin
class NewsRemoteDataSource(
  private val newsApi: NewsApi,
  private val ioDispatcher: CoroutineDispatcher
) {
    /**
     * Fetches the latest news from the network and returns the result.
     * This executes on an IO-optimized thread pool, the function is main-safe.
     */
    suspend fun fetchLatestNews(): List<ArticleHeadline> =
        // Move the execution to an IO-optimized thread since the ApiService
        // doesn't support coroutines and makes synchronous requests.
        withContext(ioDispatcher) {
            newsApi.fetchLatestNews()
        }
    }
}

// Makes news-related network synchronous requests.
interface NewsApi {
    fun fetchLatestNews(): List<ArticleHeadline>
}
```
```javascript
// NewsRepository is consumed from other layers of the hierarchy.
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource
) {
    suspend fun fetchLatestNews(): List<ArticleHeadline> =
        newsRemoteDataSource.fetchLatestNews()
}
```
#### App-oriented operations
- app이 시작되는 순간 동작하는 operation
- app이 closed되거나 process kill되면 종료
- Application class나 data layer lifecycle을 따른다.
- 예를 들면 app을 시작하는 순간 이전에 받아와 cached된 news를 보여주거나 바로 network를 통해서 latest news를 fetch하는 것
  - data를 받아올 때 다른 thread에서의 read/write를 막기 위해 mutex 사용
```kotlin
class NewsRepository(
  private val newsRemoteDataSource: NewsRemoteDataSource
) {
    // Mutex to make writes to cached values thread-safe.
    private val latestNewsMutex = Mutex()

    // Cache of the latest news got from the network.
    private var latestNews: List<ArticleHeadline> = emptyList()

    suspend fun getLatestNews(refresh: Boolean = false): List<ArticleHeadline> {
        if (refresh || latestNews.isEmpty()) {
            val networkResult = newsRemoteDataSource.fetchLatestNews()
            // Thread-safe write to latestNews
            latestNewsMutex.withLock {
                this.latestNews = networkResult
            }
        }

        return latestNewsMutex.withLock { this.latestNews }
    }
}
```
  - 만약 데이터를 받아오는 도중 user가 screen을 벗어나게 되면 operation이 중단될 수 있으니 externalScope을 사용
```kotlin
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource,
    private val externalScope: CoroutineScope
) {
    /* ... */

    suspend fun getLatestNews(refresh: Boolean = false): List<ArticleHeadline> {
        return if (refresh) {
            externalScope.async {
                newsRemoteDataSource.fetchLatestNews().also { networkResult ->
                    // Thread-safe write to latestNews.
                    latestNewsMutex.withLock {
                        latestNews = networkResult
                    }
                }
            }.await()
        } else {
            return latestNewsMutex.withLock { this.latestNews }
        } 
    }
}
```
#### Business-oriented operations
- cancel될 수 없다.
  - process death가 되어도 살아있다.
- 예를 들면 device charging 상태이거나 unmetered network인 경우 news app은 정기적, 자동으로 latest news를 가져오는 경우다.
  - WorkManager를 사용하며 Worker class를 만든다.
```kotlin
class RefreshLatestNewsWorker(
    private val newsRepository: NewsRepository,
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result = try {
        newsRepository.refreshLatestNews()
        Result.success()
    } catch (error: Throwable) {
        Result.failure()
    }
}
```
```kotlin
private const val REFRESH_RATE_HOURS = 4L
private const val FETCH_LATEST_NEWS_TASK = "FetchLatestNewsTask"
private const val TAG_FETCH_LATEST_NEWS = "FetchLatestNewsTaskTag"

class NewsTasksDataSource(
    private val workManager: WorkManager
) {
    fun fetchNewsPeriodically() {
        val fetchNewsRequest = PeriodicWorkRequestBuilder<RefreshLatestNewsWorker>(
            REFRESH_RATE_HOURS, TimeUnit.HOURS
        ).setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.TEMPORARILY_UNMETERED)
                .setRequiresCharging(true)
                .build()
        )
            .addTag(TAG_FETCH_LATEST_NEWS)

        workManager.enqueueUniquePeriodicWork(
            FETCH_LATEST_NEWS_TASK,
            ExistingPeriodicWorkPolicy.KEEP,
            fetchNewsRequest.build()
        )
    }

    fun cancelFetchingNewsPeriodically() {
        workManager.cancelAllWorkByTag(TAG_FETCH_LATEST_NEWS)
    }
}
```
### [Save and retrieve data from disk]
- data 중 user가 bookmark한 news나 user preferences와 같은 data는 process death되어도 유지되어야 한다.(network 없어도)
- largest dataset : Room DB + Data Access Object(DAO, DB CRUD object)
- small dataset : DataStore(key-value pairs
- Chunks of data like a JSON : File
  - 각각의 class들은 only one source for a specific type of data에 대해서만 책임을 갖도록 구성되어야 한다.

### [Expose errors]
- repository와 data source 간의 interaction에서 succeed하거나 failure할 수 있다.
  - try/catch block을 사용하거나 flows의 경우 catch operator를 사용한다.



## Domain layer
- UI layer와 Data layer 사이에 있는 optional layer
- complex business logic이나 여러 ViewModels에서 reuse할 수 있는 business logic을 구성하는 역할
  - code duplication을 방지
  - readability 증가
  - app testability 개선
  - 역할을 나누어 class가 커지는 것을 방지
### [Naming conventions]
- verb in present tense(single action) + noun/what(optional) + UseCase
  - 예를 들면 FormatDateUseCase, LogOutUserUseCase
### [Dependencies]
- UI layer의 ViewModels와 Data layer의 repositories 사이에 위치
  - 여러 repository classes에 의존
  - Domain layer의 Use case는 reusable logic이기 때문에 다른 Use case를 사용할 수도 있다.
```kotlin
class GetLatestNewsWithAuthorsUseCase(
  private val newsRepository: NewsRepository,
  private val authorsRepository: AuthorsRepository,
  private val formatDateUseCase: FormatDateUseCase
) { /* ... */ }
```
### [Call use cases in Kotlin]
- operator invoke를 사용하여 function과 같이 call하도록 작성
```kotlin
class FormatDateUseCase(userRepository: UserRepository) {

    private val formatter = SimpleDateFormat(
        userRepository.getPreferredDateFormat(),
        userRepository.getPreferredLocale()
    )

    operator fun invoke(date: Date): String {
        return formatter.format(date)
    }
}
```
```kotlin
class MyViewModel(formatDateUseCase: FormatDateUseCase) : ViewModel() {
    init {
        val today = Calendar.getInstance()
        val todaysDate = formatDateUseCase(today)
        /* ... */
    }
}
```
### [Lifecycle]
- own lifecycle을 갖지 않고 Use case class를 사용하는 class lifecycle에 scope됨
- mutable data를 가지면 안되기 때문에 필요할 때마다 new instance를 생성한다.
### [Threading]
- 역시 main-safe해야 한다
- long-running blocking operations라면 알맞은 thread를 사용해야 한다.
  - 하지만 그 전에 정말 domain layer에서 해야 하는지, 다른 layer에서 실행하는 것이 더 나은지 check한다.
### [Common tasks]
#### Reusable simple business logic
- date format을 바꾸는 FormatDateUseCase와 같이 여러 곳에서 재사용될 business logic을 포함
#### Combine repositories
- news와 author 데이터를 같이 operate하기 위해 2개 이상의 repository를 의존하듯 여러 개의 repository를 의존해서 결과를 도출한다.
```kotlin
/**
 * This use case fetches the latest news and the associated author.
 */
class GetLatestNewsWithAuthorsUseCase(
  private val newsRepository: NewsRepository,
  private val authorsRepository: AuthorsRepository,
  private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    suspend operator fun invoke(): List<ArticleWithAuthor> =
        withContext(defaultDispatcher) {
            val news = newsRepository.fetchLatestNews()
            val result: MutableList<ArticleWithAuthor> = mutableListOf()
            // This is not parallelized, the use case is linearly slow.
            for (article in news) {
                // The repository exposes suspend functions
                val author = authorsRepository.getAuthor(article.authorId)
                result.add(ArticleWithAuthor(article, author))
            }
            result
        }
}
```
- 만약 database가 source of truth라면 여러 entity의 관계를 받는 query를 통해 결과를 얻을 수 있다면 Use case보다는 NewsWithAuthorRepository를 만드는 것이 더 낫다.
### [Other consumers]
- domain layer class는 services, Application class와 같은 other classes나 TV, Wearable과 같은 other platforms에서도 reuse할 수 있다.

## UI events
- UI events : UI layer(UI or by the ViewModel)에 의해 handle되어야 하는 actions
- UI : Views or Jetpack compose(UI elements)
- User events : app과 interaction하는 user에 의해 발생하는 events
### [UI event decision tree]
<img width="614" alt="스크린샷 2022-02-16 오후 3 44 00" src="https://user-images.githubusercontent.com/17876424/154273031-bc0cf516-4572-46ed-8bc0-4694cc0408c7.png"><br>
### [Handle user events]
- 예를 들면 item을 펼쳐서 보여주는 것과 같이 UI element의 상태를 변화시키는 것이라면 직접 UI를 통해서 변경
- refresh data와 같이 UI state가 변경되어 app data가 변하는 등의 business logic이라면 ViewModel에게 위임
```kotlin
class LatestNewsActivity : AppCompatActivity() {

    private lateinit var binding: ActivityLatestNewsBinding
    private val viewModel: LatestNewsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        /* ... */

        // The expand section event is processed by the UI that
        // modifies a View's internal state.
        binding.expandButton.setOnClickListener {
            binding.expandedSection.visibility = View.VISIBLE
        }

        // The refresh event is processed by the ViewModel that is in charge
        // of the business logic.
        binding.refreshButton.setOnClickListener {
            viewModel.refreshNews()
        }
    }
}
```
#### User events in RecyclerViews
- RecyclerView item이나 Custom view와 관련된 events 같은 경우도 ViewModel이 담당
- 이때, RecyclerView adapter가 ViewModel에 대한 의존성을 직접 갖는 것이 아니라 state object를 통해서 이루어지도록 해야 한다.
  - RecyclerView adapter가 ViewModel class와 tightly coupling되도록 하는 것이 옳지 않다.
```kotlin
data class NewsItemUiState(
    val title: String,
    val body: String,
    val bookmarked: Boolean = false,
    val publicationDate: String,
    val onBookmark: () -> Unit
)

class LatestNewsViewModel(
    private val formatDateUseCase: FormatDateUseCase,
    private val repository: NewsRepository
)
    val newsListUiItems = repository.latestNews.map { news ->
        NewsItemUiState(
            title = news.title,
            body = news.body,
            bookmarked = news.bookmarked,
            publicationDate = formatDateUseCase(news.publicationDate),
            // Business logic is passed as a lambda function that the
            // UI calls on click events.
            onBookmark = {
                repository.addBookmark(news.id)
            }
        )
    }
}
```
### [Handle ViewModel events]
- ViewModel events라는 것은 예를 들면 ViewModel operation 중 refresh data가 있을 때 internetconnection이 좋지 않아 message를 보내야 하는 상황
- ViewModel events는 반드시 UI state update로 마무리해야 한다.
  - 즉, UI action이 UI state과 mapping되도록 하는 것이 좋다.
  - 단순히 events의 결과로 화면만 전환된다고 하자
  - 중간에 configuration이 변하거나 process가 죽으면 상태가 유지되지 않는다.
  - UI state update를 통해 상태를 저장해놓아야 한다.
    - UI state 이외에 ViewModel이라면 SavedStateHandle을 통해 상태를 저장할 수도 있다.
- 예를 들면 로그인 화면에서 사용자가 로그인을 한 후 화면이 전환될 때 화면 전환만 이루어지는 것이 아니라 isUserLoggedIn이라는 UI State 변수를 만들어 상태를 저장한다.
#### Consuming events can trigger state updates
- 어떤 Viewmodel events는 다른 UI state를 update시킬 수도 있다.
- ViewModel은 UI에 어떻게 나타나는지 알 필요는 없다.
  - UI가 UI state가 update되도록 ViewModel에게 알린다.
  - 다음 예시는 transient message가 나타난 이후 UI가 ViewModel에게 알려 UI state가 update되는 예시다.
```kotlin
// Models the message to show on the screen.
data class UserMessage(val id: Long, val message: String)

// Models the UI state for the Latest news screen.
data class LatestNewsUiState(
    val news: List<News> = emptyList(),
    val isLoading: Boolean = false,
    val userMessages: List<UserMessage> = emptyList()
)
```
```kotlin
class LatestNewsViewModel(/* ... */) : ViewModel() {

    private val _uiState = MutableStateFlow(LatestNewsUiState(isLoading = true))
    val uiState: StateFlow<LatestNewsUiState> = _uiState

    fun refreshNews() {
        viewModelScope.launch {
            // If there isn't internet connection, show a new message on the screen.
            if (!internetConnection()) {
                _uiState.update { currentUiState ->
                    val messages = currentUiState.userMessages + UserMessage(
                        id = UUID.randomUUID().mostSignificantBits,
                        message = "No Internet connection"
                    )
                    currentUiState.copy(userMessages = messages)
                }
                return@launch
            }

            // Do something else.
        }
    }

    fun userMessageShown(messageId: Long) {
        _uiState.update { currentUiState ->
            val messages = currentUiState.userMessages.filterNot { it.id == messageId }
            currentUiState.copy(userMessages = messages)
        }
    }
}
```
```kotlin
class LatestNewsActivity : AppCompatActivity() {
    private val viewModel: LatestNewsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        /* ... */

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { uiState ->
                    uiState.userMessages.firstOrNull()?.let { userMessage ->
                        // TODO: Show Snackbar with userMessage.
                        // Once the message is displayed and
                        // dismissed, notify the ViewModel.
                        viewModel.userMessageShown(userMessage.id)
                    }
                    ...
                }
            }
        }
    }
}
```
### [Other use cases]
- 하나의 class는 하나의 책임, 역할만 해야 한다.
  - UI는 UI logic, ViewModel은 business logic + update UI State
- Decision tree를 통해서 event가 어디서 발생했고 어디서 logic이 이루어져야 하는지 따져본다.
- 만약 여러 명의 consumers가 있고 event가 여러 번 발생하는 것이 문제가 된다면 app architecture를 재고려해봐야 한다.
- 언제 state가 사용되어야 하는지 생각하자
  - app이 background에 있을 때는 사용되지 않는 것이 좋을 수도 있다.
