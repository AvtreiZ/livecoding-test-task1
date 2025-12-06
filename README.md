# Легенда:
Джуниор написал код и уехал в отпуск. Его код пришел на ревью, нужно сделать его чище и исправить явные баги, если таковые есть. Исправления в коде и улучшения нужно аргументировать, чтобы Джуниор чему-то научился.

# Задача:
Требуется прочитать код, понять какую логику он выполняет и что задумывал разработчик, отрефакторить и сделать чище код. Если есть явные баги - исправить. Если есть идеи по тому как сделать код лучше - реализовать их.

```Kotlin
class MainActivity: FragmentActivity() {  
  
    @Inject  
    public lateinit var getContent: GetContentRepository  
    public lateinit var service: RequestService  
    public lateinit var black: CheckBlacklistService  
    public lateinit var analytics: AnalyticsService  
  
    private lateinit var binding: LayoutMainBinding  
  
    private val vm by lazy {  
        MainViewModel(intent.getStringExtra("userId"), this, getContent, service, analytics, black, lifecycleScope)  
    }  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        MainActivityInjector.inject()  
        binding = LayoutMainBinding.inflate(layoutInflater)  
        setContentView(binding.root)  
  
        supportFragmentManager.beginTransaction()  
            .add(R.id.fragmentContainer, AdsFragment())  
            .commit()  
  
        lifecycleScope.launch {  
            vm.sharedFlow.collect {  
                if (it == null) {  
                    Toast.makeText(applicationContext, "Что-то пошло не так", Toast.LENGHT_LONG).show()  
                } else {  
                    binding.name.text = it.name!!  
                    binding.lastName.animate()  
                    binding.postDelayed({  
                        binding.lastName.text = it.lastName!!  
                    }, 2_000L)  
                }  
            }  
        }  
        binding.btn.setOnClickListener {  
            if(vm.age < 18) {  
                Toast.makeText(applicationContext, "Вам должно быть 18 лет", Toast.LENGHT_LONG).show()  
                return@setOnClickListener  
            }  
  
            lifecycleScope.launch {  
                vm.request().collect {  
                    if(it == null) {  
                        Toast.makeText(applicationContext, "Что-то пошло не так", Toast.LENGHT_LONG).show()  
                    } else {  
                        supportFragmentManager  
                            .beginTransaction()  
                            .add(R.id.fragmentContainer, ScooterFragment().apply {userId = vm.userId})  
                            .commit()  
                    }  
                }  
            }        
        }    
    }  
}  
  
class MainViewModel(  
    val userId: String?,  
    val context: Context,  
    val getContentProvider: GetContentRepository,  
    val service: RequestService,  
    val analytics: AnalyticsService,  
    val black: CheckBlacklistService,  
    val coroutineScope: CoroutineScope,  
) {  
    var age = 0  
  
    val sharedFlow = MutableSharedFlow<ContentModel?>(extraBufferCapacity = 1)  
  
    init {  
        coroutineScope.launch {  
            sharedFlow.emit(  
                try {  
                    val c = getContentProvider.getContent(userId ?: error("userId must set"))  
                    age = c.age ?: 0  
                    c.copy(  
                        c.name ?: context.getString(R.string.name),  
                        c.lastName ?: context.getString(R.string.last_name)  
                    )  
                } catch (e: Exception) {  
                    null  
                }  
            )  
        }  
    }  
  
    @Synchronized  
    fun request(): Flow<Unit>? {  
        return flow {  
            try {  
                analytics.requestClicked()  
                emit(Unit)  
            } catch (e: Exception) {  
                emit(null)  
            }  
        }.flowOn(Dispatchers.IO).flatMapConcat {  
            flow {  
                if (it != null) {  
                    try {  
                        service.requestScooter()  
                    } catch (e: Exception) {  
                        emit(null)  
                    }  
                }  
                emit(it)  
            }  
        }.flowOn(Dispatchers.IO).flatMapConcat {  
            flow {  
                if (it != null) {  
                    try {  
                        black.check(userId!!)  
                    } catch (e: Exception) {  
                        emit(null)  
                    }  
                }  
                emit(it)  
            }  
        }.flowOn(Dispatchers.IO)  
    }  
}  
  
data class ContentModel(  
    var name: String?,  
    var lastName: String?,  
    var age: Int?  
)  
  
interface GetContentRepository {  
    suspend fun getContent(id: String): ContentModel  
}  
  
interface RequestService {  
    @POST("scooter/request")  
    suspend fun requestScooter(): ScooterResponse  
}  
  
/**  
 * Methods can throw an Exception 
*/
interface CheckBlackListService {  
    @POST("users/checkBlackList")  
    suspend fun check(userId: String)  
}  
  
interface AnalyticsService {  
    @GET("users/analytics/requestClicked")  
    suspend fun requestClicked()  
}  
  
class ScooterResponse {  
    val resultCode: Int  
}
```

# Мысли

* Присутствуют утечки памяти. Передача `Context`, `Activity` и `lifecycleScope` во `MainViewModel` может быть опасно, за счет того, что мы можем потенциально обратиться к ActivityContext в время уничтожения Activity (смена конфигурации). Ко всему прочему передавать весь Context со всем функционалом может быть излишне:
  1. Получение системных сервисов по типу `VibrationManager`, `LocationManager`, `ConnectivityManager` и т.д.
  2. Доступ ко всем ресурсам (`Strings`, `Values`, `Drawables`).
  3. Доступ к файловой системе.
  4. Доступ к текущей среде (`Window`)
  
  Вьюмодели в данном случае нужен только доступ к стрингам. И такой функционал можно передать используя `Interface Segregation`. Создать отдельный интерфейс `ResourcesManager` c методом `getString(@StringRes id: Int): String` и тогда вьюмодель будет знать только о том функционале, который нужен ему в данный момент времени. Ко всему прочему реализацию всегда можно будет подменить.

* Все переменные я бы называл более понятным образом. `vm` -> `viewModel`, `black` -> `blackListService` и т.д.

* Вместо использования `FragmentActivity`, я бы лучше использовал `AppCompatActivity` для поддержки обратной совместимости со старыми версиями Android + `AppCompatActivity` наследуется от `FragmentActivity`, поэтому мы ничего не теряем.

* Инъекция бизнес логики прямо в Activity:  

```Kotlin
@Inject  
public lateinit var getContent: GetContentRepository  
public lateinit var service: RequestService  
public lateinit var black: CheckBlacklistService  
public lateinit var analytics: AnalyticsService
```

  Так как здесь присутствует явная попытка в паттерн MVVM, то следует соблюдать все его гайдлайны, а именно: `View` ничего не должна знать о `Model` (бизнес логика) и о `ViewModel`. Максимум что она может, это подписаться на изменения данных, предоставляемых `ViewModel`.
  С бизнес логикой напрямую может взаимодействовать только `ViewModel`, поэтому лучше все эти зависимости перенести во `ViewModel`. Ко всему прочему можно будет убрать `MainActivityInjector.inject()`, т.к. ничего инжектиться в активити больше не будет. Также эти поля публичные и изменяемые, что противоречит принципу **инкапсуляции**.

* `binding` лучше разделить на мутабельный и на иммутабельный, чтобы избежать случайного стирания ссылки на `binding`.

* `ViewModel` в` MainActivity` инициализируется через `lazy`, что не очень хорошо. Несколько причин: 
  1. когда активити уничтожиться (например смена конфигурации), объект `ViewModel` останется жить некоторое время в памяти и продолжит работу, до следующего *garbage collection*, что является утечкой памяти + он принимает в конструктор `ActivityContext`, что усиливает опасность, так как потенциально `ViewModel` может обратиться к уничтоженной `Activity`
  2. `ViewModel` у нас все же должна переживать такие моменты как смена конфигурации, потому что это сулит потерей предыдущих полученный данных, следовательно придется лишний раз идти за ними.
  Поэтому лучше иcпользовать либо `ViewModelProvider`, либо инжектить.
  
* Собирать Flow в активити следует с помощью `repeatOnLifecycle(Lifecycle.State.STARTED)`.
  Причина: ненужная работа - без этого метода значения будут собираться вплоть до `onDestroy`, потенциально может случиться такая ситуация, что при эмите следующего значения, коллектор будет пытаться обновить вью которой нет или не видна. Это пустая трата ресурсов.
  
* Проверка на возраст в `Activity`. Это больше похоже на бизнес логику, ее лучше поместить во `Model`, потому что не `Activity` должна принимать такие решения.
  
* Проверка на `null` в активити, и в зависимости от этого показывается тост. Не активити должна принимать такое решение, а `ViewModel`. По MVVM вью должна только подписываться на изменение данные вью. Проще говоря, она не должна ничего делать кроме того, чтобы показывать то, что ей говорят и отправлять инпуты юзера во вьюмодель на обработку. Под инветы по типу "показать тост" или навигацию можно завести отдельные `SharedFlow` во `ViewModel`, тем самым `ViewModel` будет принимать решение что показывать, а активити это молча принимает и показывает.

* `animate` + `postDelayed`. Я так понимаю что тут происходит анимация, в течение 2 секунд она должна закончится и должно поставиться новое название. Вроде как `animate()` внутренняя доп функция у класса, расширяющего `TextView`. В таком случае, бы сделал внутри реализации animate слушателя, который по окончанию анимации сам сетит новый текст по окончанию анимации. Текст можно в принципе передать во входной параметр `animate()`. Если это так, то ее бы по-хорошему и переименовать, аля `setTextWithAnimation(text: String)`

* Получение аргумента из `intent` можно перенести во `ViewModel`, с помощью механизма `SavedStateHandle`. + В зависимости от требований интервьюера, можно сделать, чтобы приложение не падало, если `userId = null`, например можно сделать навигацию назад в случае если был переда некорректный айди.

* Так как мы запускаем метод `request()` через корутину, аннотация `@Synchronized` не будет работать правильно, т.к. она ипользуется для синхронизации потоков, а не корутин. (Если требуется синхронизация корутин, нужно исрользовать `Mutex`)

* Для всяческих запросов лучше использовать `viewModelScope` и в принципе запускать такие корутины во `ViewModel`. Потому что `lifecycleScope` живет столько, сколько живет `Activity` и при смене конфигурации, все корутины будут принудительно отменены. `ViewModel` же вместе с `viewModelScope` переживает такие ситуации.

* переменная `sharedFlow` во `ViewModel`. В ней несколько проблем:
  1. она публичная и мутабельная, любой кто имеет объект этой вьюмодели, в любой момент сможет что-нибудь заэмитить, если захочет. ViewModel не должна предоставлять такой возможности (инкапсуляция). Следует разделить на приватный мутабельный флоу и публичный немутабельный.
  2. Это `SharedFlow`. Активити, которая пересоздаться, потеряет предыдущее значение и переподписке (в случае смены конфигурации). `extraBufferCapacity` не спасает ситуацию, т.к. она буферизуется значения именно для текущих подписчиков, а не для новый. Для новый подписчиков сохраненные значение хранит именно `replayCache`. Можно сделать либо `replay = 1`, либо заменить `SharedFlow` на `StateFlow`, у которого можно удобно получить последнее сохраненное значение. Соответственно не понадобятся всякие доп переменные как `age`.

* Пока будут идти всяческие запросы и инициализация данных, по-хорошему юзеру показать какой-то лоадер и ограничить нажатие на кнопку, чтобы нельзя было послать 100 запросов разом и что-нибудь нечайно задудосить. У интервьюера можно спросить, что показывать (или не показывать) во время работы в фоне. Можно использовать `UiState`, с разделением на `Loading`, `Success`, `Error`,  а можно завести под это дело отдельный flow `isLoading` (выбрал этот вариант, просто потому что, он быстрее)

* Непонятый порядок запросов в методе `request()`. Тут очень много нюансов:
  1. `flowOn(Dispatchers.IO)` после каждого запроса. Толку от этого ноль, одного метода `flowOn(Dispatchers.IO)` в самом низу цепочки уже достаточно. Все методы, которые выше него, будут выполнять на IO.
  2. `flatMapConcat` которые делают запросы и ничего не делают с результатом `service.requestScooter()` какой бы код не вернулся, ошибочный или нет, на UI это отобразиться как успешный результат.
  3. Сначала посылаем аналитику, затем запрашиваем скутер, а затем проверяем, нет ли юзера в черном списке. Логичнее будет наверное сначала проверить не в черном списке ли юзер и исходя из результата принять решение, заказывать ли скутер или нет. 
  4. Непонятно почему это сделано в виде цепочки flow, так еще и код написан так, что при нажатии на кнопку каждый раз запускается новый флоу, не отменяя предыдущий, что потенциально приводит к не консистентным данным. Это все дело можно организовать и без flow, используя простые `suspend` функции.
  5. Вообще 3 этих запрос представляют из себя цельную бизнес логику. Их всех можно совместить в один `UseCase`. Ко всему прочему ему можно передать `age` как параметр и теперь бизнес логика принимает решение на основе возраста.

* В `getContentRepository` мы получаем наш `ContentModel` c `nullable` мутабельными полями. Во-первых я бы из сделал иммутабельными (инкапсуляция). Во-вторых оперировать `nullable` полями не очень удобно + ко всему, в init у вьюмодели ставятся дефолтные значения если они `null`. В таком случае можно завести отдельную UI модель `ContentModelUI`. Затем смапить `ContentModel` в `ContentModelUI`. Presentation слой не должен оперировать моделями из бизнес логики. Если бизнес логика поменяется - придется полностью переписывать Presentation.

* Передавать аргументы во фрагмент лучше через `arguments` с передачей `Bundle`.

# Результат

```Kotlin
class MainActivity : AppCompatActivity() {  
  
    private var _binding: LayoutMainBinding? = null  
    private val binding get() = _binding!!  
  
    private val viewModel: MainViewModel by viewModels<MainViewModel>()  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        _binding = LayoutMainBinding.inflate(layoutInflater)  
        setContentView(binding.root)  
  
        supportFragmentManager.beginTransaction()  
            .add(R.id.fragmentContainer, AdsFragment())  
            .commit()  
  
        collectWithLifecycle(viewModel.contentModelFlow) {  
            binding.name.text = it.name  
            binding.lastName.setTextWithAnimation(it.lastName)  
        }  
  
        collectWithLifecycle(viewModel.isLoading) { isLoading ->  
            binding.btn.isVisible = !isLoading  
            bindgin.loader.isVisible = isLoading  
        }  
  
        collectWithLifecycle(viewModel.events) { event ->  
            when (event) {  
                is MainViewModel.Event.ErrorToast ->  
                    Toast.makeText(applicationContext, event.message, Toast.LENGHT_LONG).show()  
  
                is MainViewModel.Event.NavigateToScooterFragment ->  
                    supportFragmentManager  
                        .beginTransaction()  
                        .add(  
                            R.id.fragmentContainer,  
                            ScooterFragment().apply {  
                                arguments = bundleOf(ScooterFragment.USER_ID_KEY to event.userId)  
                            }  
                        )  
                        .commit()  
            }  
        }  
    }  
}  
  
fun <T> AppCompatActivity.collectWithLifecycle(flow: Flow<T>, block: (T) -> Unit) {  
    lifecycleScope.launch {  
        repeatOnLifecycle(Lifecycle.State.STARTED) {  
            flow.collect(block)  
        }  
    }}  
  
class MainViewModel(  
    private val getContentRepository: GetContentRepository,  
    private val bookScooterUseCase: BookScooterUseCase,  
    private val resourcesManager: ResourcesManager,  
    private val savedStateHandle: SavedStateHandle,  
) {  
  
    sealed class Event {  
        data class ErrorToast(val message: String) : Event()  
        data class NavigateToScooterFragment(val userId: String) : Event()  
    }  
  
    private val userId: String = savedStateHandle["userId"] ?: error("userId must be set")  
  
    private val _events = MutableSharedFlow<Event>()  
    val events = _events.asSharedFlow()  
  
    private val _contentModelFlow = MutableStateFlow<ContentModelUI>(ContentModelUI("", "", -1))  
    val contentModelFlow = _contentModelFlow.asStateFlow()  
  
    private val _isLoading = MutableStateFlow(true)  
    val isLoading = _isLoading.asStateFlow()  
  
    init {  
        viewModelScope.launch(Dispatchers.IO) {  
            runCatching { getContentRepository.getContent(userId) }  
                .onSuccess { content ->  
                    _contentModelFlow.value = ContentModelUI(  
                        name = content.name ?: resourcesManager.getString(R.string.name),  
                        lastName = content.lastName ?: resourcesManager.getString(R.string.last_name),  
                        age = content.age ?: 0  
                    )  
                }  
                .onFailure {  
                    _events.emit(Event.ErrorToast(resourcesManager.getString(R.string.something_went_wrong)))  
                }  
            _isLoading.update { false }  
        }    
    }  
  
    fun request() {  
        viewModelScope.launch(Dispatchers.IO) {  
            _isLoading.update { true }
			val age = _contentModelFlow.value.age
			bookScooterUseCase(userId, age)  
				.onSuccess { _events.emit(Event.NavigateToScooterFragment) }  
				.onFailure {  
					val errorMessage = when (it) {  
						LowAgeException() -> resourcesManager.getString(R.string.low_age_error_text)  
						else -> resourcesManager.getString(R.string.something_went_wrong)  
					}  
					_events.emit(Event.ErrorToast(errorMessage))  
				}   
            _isLoading.update { false }  
        }
    }  
}  
  
interface ResourcesManager {  
    fun getString(@StringRes id: Int): String  
  
    class Base(  
        private val context: Context  
    ) : ResourcesManager {  
  
        override fun getString(id: Int): String =  
            context.getString(id)  
    }  
}  
  
data class ContentModel(  
    val name: String?,  
    val lastName: String?,  
    val age: Int?  
)  
  
data class ContentModelUI(  
    val name: String,  
    val lastName: String,  
    val age: Int,  
)  
  
class LowAgeException() : IllegalArgumentException()  
  
class BookScooterUseCase @Inject constructor(  
    private val requestService: RequestService,  
    private val checkBlackListService: CheckBlackListService,  
    private val analyticsService: AnalyticsService,  
) {  
    suspend operator fun invoke(userId: String, age: Int): Result<Unit> = 
	    runCatching {  
	        if (age < MIN_AGE) throw LowAgeException()  
	        analyticsService.requestClicked()  
	        checkBlackListService.check(userId)  
	        if (requestService.requestScooter().resultCode != SUCCESS_CODE)  
	            throw IllegalStateException()  
	    }  
  
    companion object {  
        private const val SUCCESS_CODE = 200  
        private const val MIN_AGE = 18  
    }  
}  
  
interface GetContentRepository {  
    suspend fun getContent(id: String): ContentModel  
}  
  
interface RequestService {  
    @POST("scooter/request")  
    suspend fun requestScooter(): ScooterResponse  
}  
  
/**  
 * Methods can throw an Exception 
*/
interface CheckBlackListService {  
    @POST("users/checkBlackList")  
    suspend fun check(userId: String)  
}  
  
interface AnalyticsService {  
    @GET("users/analytics/requestClicked")  
    suspend fun requestClicked()  
}  
  
class ScooterResponse(  
    val resultCode: Int  
)
```
