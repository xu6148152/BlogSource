---
title: Little stories about an Android application architecture
tags: Android
date: 2016-06-11 15:59:27
---

[原文](http://roroche.github.io/AndroidStarter/)

写这篇文章的目的是为了描述我建议的``Android app``架构。我一步步的通过下面的原因来选择不同的组件。

模板应用的目的很简单：它是``master/detail``结构，用来展示指定用户的``Github``仓库。虽然它简单，但它集合了一些通用的东西：

 * 使用``REST``API  
 * 本地数据存储
 * 本地数据加载
 * 架构逻辑层以及页面导航

让我们来看看这背后都有什么!
 
 
###Consuming REST API

>REST Representational State Transfer表述性状态传递

[Retrofit](http://square.github.io/retrofit/)是开发者必备的知名网络库  
我将解释为何我认为它是必备的库：  

* 类型安全
* 接口可读性好，采用注解``API``地址
* 对如何工作采用完全抽象层
* 支持多部分请求体(上传文件)
* 使用注解直接管理头部信息
* 能够使用多种序列化类型(``JSON, XML, protobuf, etc``)的转换器
* 能够添加全局的请求拦截器
* 方便的进行``MOCK``测试

使用时仅需在``build.gradle``文件中添加

```
compile 'com.squareup.retrofit:retrofit:{{last_version}}'
```

然后我能够声明``GitHubService``接口和我们需要使用这个接口的方法

```
public interface GitHubService {
    @GET("/users/{user}/repos")
    Call<List<DTORepo>> listRepos(@Path("user") final String psUser);
}
```

下一步通过``RestAdapter``去实现这个接口

```
final Retrofit loRetrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .build();

final GitHubService loService = loRetrofit.create(GitHubService.class);
return loService;
```

我使用[Merlin](https://github.com/novoda/merlin)。它能够观察网络的连接状态以及改变。它提供流畅``API``，设置简单。

```
final MerlinsBeard merlinsBeard = MerlinsBeard.from(context);

```

调用

```
merlinsBeard.isConnected()
```
来检查网络是否可用。

###解析数据

现在我们已经能够从服务器拿到数据了，我们需要将这些数据转换成``POJO``对象。通常的格式是``JSON``,我们使用[Jackson](https://github.com/FasterXML/jackson)来作为转换器。

当然，我将会说明一下为什么我选择``Jackson``。首先，我很满意它流畅的注解``API``。能够获取或存储那些没有通过``@JsonProperty``注解声明的属性

```
@JsonIgnore
    private Map<String, Object> mAdditionalProperties = new HashMap<String, Object>();

    @JsonAnyGetter
    public Map<String, Object> getAdditionalProperties() {
        return mAdditionalProperties;
    }

    @JsonAnySetter
    public void setAdditionalProperty(final String psName, final Object poValue) {
        mAdditionalProperties.put(psName, value poValue;
    }
```

与``Retrofit``组合

[https://github.com/square/retrofit/tree/master/retrofit-converters/jackson.](https://github.com/square/retrofit/tree/master/retrofit-converters/jackson.)

```
compile 'com.squareup.retrofit2:converter-jackson:{{last_version}}'
```

然后我们将其设置到我们之前的``RestAdapter``

```
final Retrofit loRetrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(JacksonConverterFactory.create()) // add the Jackson specific converter
    .build();

final GitHubService loService = loRetrofit.create(GitHubService.class);
return loService;
```

福利
我通常使用[jsonchema2pojo](http://www.jsonschema2pojo.org/)  
> Generate Plain Old Java Objects from JSON or JSON-Schema.

也可以使用[moshi](https://github.com/square/moshi)或者[LoganSquare](https://github.com/bluelinelabs/LoganSquare)  

####组件之间的通讯和数据传递
通常的做法是使用事件总线

* [Otto](http://square.github.io/otto/)  
* [EventBus](http://greenrobot.github.io/EventBus/) 
* [TinyBus](https://github.com/beworker/tinybus) 

> Now can using RxJava to replace above all  

####线程管理

> RxAndroid thread

####任务管理
多线程和并发是开发者经常要考虑的问题。我罗列出一些不能再主线程操作的任务

* 调用远程API
* 数据库CURD操作
* 读取本地文件

建议采取:

* 使用``Service``
* 设置``ServiceHelper``来进行网络请求
* 使用专门的类来处理查询操作

但需要注意的是``Service``是运行在主线程中，因此可以使用``IntentService``或者使用[Android Priority Job Queue](https://github.com/yigit/android-priority-jobqueue)或者是官方的[JobScheduler](https://developer.android.com/reference/android/app/job/JobScheduler.html)

####Job Manager(android-priority-jobqueue)
#####Job Manager Configuration

```
final Configuration loConfiguration = new Configuration.Builder(poContext)
    .minConsumerCount(1) // always keep at least one consumer alive
    .maxConsumerCount(3) // up to 3 consumers at a time
    .loadFactor(3) // 3 jobs per consumer
    .consumerKeepAlive(120) // wait 2 minute
    .build();

final JobManager loJobManager = new JobManager(poContext, loConfiguration);
```
#####Job Configuration
我们可以给一个任务设置一些有用的参数，例如

* 它的优先级
* 它是否需要网络
* 如果不执行，是否需要持久化
* 延时运行
* 重试机制

这个库的持久化引擎非常强大。例如，网络不可用时，任务会被持久化到设备上。一旦网络可用``JobManager``获取持久化的任务并执行它们

```
public abstract class AbstractQuery extends Job {
    private static final String TAG = AbstractQuery.class.getSimpleName();
    private static final boolean DEBUG = true;

    protected enum Priority {
        LOW(0),
        MEDIUM(500),
        HIGH(1000);
        private final int value;

        Priority(final int piValue) {
            value = piValue;
        }
    }

    protected boolean mSuccess;
    protected Throwable mThrowable;
    protected AbstractEventQueryDidFinish.ErrorType mErrorType;

    //region Protected constructor
    protected AbstractQuery(final Priority poPriority) {
        super(new Params(poPriority.value).requireNetwork());
    }

    protected AbstractQuery(final Priority poPriority, final boolean pbPersistent, final String psGroupId, final long plDelayMs) {
        super(new Params(poPriority.value).requireNetwork().setPersistent(pbPersistent).setGroupId(psGroupId).setDelayMs(plDelayMs));
    }
    //endregion

    //region Overridden methods
    @Override
    public void onAdded() {
    }

    @Override
    public void onRun() throws Throwable {
        try {
            execute();
            mSuccess = true;
        } catch (Throwable loThrowable) {
            if (BuildConfig.DEBUG && DEBUG) {
                Logger.t(TAG).e(loThrowable, "");
            }
            mErrorType = AbstractEventQueryDidFinish.ErrorType.UNKNOWN;
            mThrowable = loThrowable;
            mSuccess = false;
        }

        postEventQueryFinished();
    }

    @Override
    protected void onCancel() {
    }

    @Override
    protected int getRetryLimit() {
        return 1;
    }
    //endregion

    //region Protected abstract method for specific job
    protected abstract void execute() throws Exception;

    protected abstract void postEventQueryFinished();

    public abstract void postEventQueryFinishedNoNetwork();
    //endregion
}
```

 * 枚举描述了我需要的优先级
 * 提供两个构造函数
 * 异常处理
 * 默认重试次数

 有三个方法被实现
 
 * ``execute``: 执行特定的代码
 * ``postEventQueryFinished ``: 通知任务结果
 * ``postEventQueryFinishedNoNetwork ``: 通知网络不可用

 后两个通常基于总线
 
 下面是我定义的抽象事件
 
 ```
 public abstract class AbstractEventQueryDidFinish<QueryType extends AbstractQuery> extends AbstractEvent {
    public enum ErrorType {
        UNKNOWN,
        NETWORK_UNREACHABLE
    }

    public final QueryType query;

    public final boolean success;
    public final ErrorType errorType;
    public final Throwable throwable;

    public AbstractEventQueryDidFinish(final QueryType poQuery, final boolean pbSuccess, final ErrorType poErrorType, final Throwable poThrowable) {
        query = poQuery;
        success = pbSuccess;
        errorType = poErrorType;
        throwable = poThrowable;
    }
}
 ```
 
 * 查询刚完成
 * 终端状态
 * 异常处理

 下面是我的查询用户仓库的代码
 
 ```
 public class QueryGetRepos extends AbstractQuery {
    private static final String TAG = QueryGetRepos.class.getSimpleName();
    private static final boolean DEBUG = true;

    //region Fields
    public final String user;
    //endregion

    //region Constructor matching super
    protected QueryGetRepos(@NonNull final String psUser) {
        super(Priority.MEDIUM);
        user = psUser;
    }
    //endregion

    //region Overridden method
    @Override
    protected void execute() throws Exception {
        final GitHubService gitHubService = // specific code to get GitHubService instance

        final Call<List<DTORepo>> loCall = gitHubService.listRepos(user);
        final Response<List<DTORepo>> loExecute = loCall.execute();
        final List<DTORepo> loBody = loExecute.body();

        // TODO deal with list of DTORepo
    }

    @Override
    protected void postEventQueryFinished() {
        final EventQueryGetRepos loEvent = new EventQueryGetRepos(this, mSuccess, mErrorType, mThrowable);
        busManager.postEventOnMainThread(loEvent);
    }

    @Override
    public void postEventQueryFinishedNoNetwork() {
        final EventQueryGetRepos loEvent = new EventQueryGetRepos(this, false, AbstractEventQueryDidFinish.ErrorType.NETWORK_UNREACHABLE, null);
        busManager.postEventOnMainThread(loEvent);
    }
    //endregion

    //region Dedicated EventQueryDidFinish
    public static final class EventQueryGetRepos extends AbstractEventQueryDidFinish<QueryGetRepos> {
        public EventQueryGetRepos(final QueryGetRepos poQuery, final boolean pbSuccess, final ErrorType poErrorType, final Throwable poThrowable) {
            super(poQuery, pbSuccess, poErrorType, poThrowable);
        }
    }
    //endregion
}
```
 
现在，通过``QueryFactor``这个简单的代理类来进行请求
 
```
public class QueryFactory {
    //region Build methods
    public QueryGetRepos buildQueryGetRepos(@NonNull final String psUser) {
        return new QueryGetRepos(psUser);
    }
    //endregion

    //region Start methods
    public void startQuery(@NonNull final Context poContext, @NonNull final AbstractQuery poQuery) {
        final Intent loIntent = new ServiceQueryExecutorIntentBuilder(poQuery).build(poContext);
        poContext.startService(loIntent);
    }

    public void startQueryGetRepos(@NonNull final Context poContext, @NonNull final String psUser) {
        final QueryGetRepos loQuery = buildQueryGetRepos(psUser);
        startQuery(poContext, loQuery);
    }
    //endregion
}
```

``Service``处理如下声明的查询:

```
public class ServiceQueryExecutor extends IntentService {
    private static final String TAG = ServiceQueryExecutor.class.getSimpleName();

    //region Extra fields
    AbstractQuery query;
    //endregion

    MerlinsBeard merlinsBeard;
    JobManager jobManager;

    //region Constructor matching super
    /**
     * Creates an IntentService.  Invoked by your subclass's constructor.
     */
    public ServiceQueryExecutor() {
        super(TAG);
    }
    //endregion

    //region Overridden methods
    @DebugLog
    @Override
    protected void onHandleIntent(final Intent poIntent) {
        // TODO get AbstractQuery from Intent 
        // TODO get MerlinsBeard and JobManager instances

        // If query requires network, and if network is unreachable, and if the query must not persist
        if (query.requiresNetwork() &&
                !merlinsBeard.isConnected() &&
                !query.isPersistent()) {
            // then, we post an event to notify the job could not be done because of network connectivity
            query.postEventQueryFinishedNoNetwork();
        } else {
            // otherwise, we can add the job
            jobManager.addJobInBackground(query);
        }
    }
    //endregion
}
```

###数据持久化

####ORM 方式
对象关系映射是软件开发中经常使用的技术

####[OrmLite](http://ormlite.com/sqlite_java_android_orm.shtml)
第一步设置要被映射的``POJO``
我创建抽象类来匹配``_id``列

```
public abstract class AbstractOrmLiteEntity {
    @DatabaseField(columnName = BaseColumns._ID, generatedId = true)
    protected long _id;

    //region Getter
    public long getBaseId() {
        return _id;
    }
    //endregion
}
```

现在创建一个``POJO``类

```
@DatabaseTable(tableName = "REPO", daoClass = DAORepo.class)
public class RepoEntity extends AbstractOrmLiteEntity {
    @DatabaseField
    public Integer id;

    @DatabaseField
    public String name;

    @DatabaseField
    public String location;

    @DatabaseField
    public String url;
}
```

我们来看一下DAO。这个设计目的在于通过抽象的接口来访问具体的数据。

``OrmLite``提供了``Dao``接口以及其实现``BaseDaoImpl``。CURD所需的操作都具备。

然而，这些都是同步执行的。使用``RxJava``来异步执行这些操作。

因此我使用``RxJava``重写了所有的方法
我创建了如下接口

```
public interface IRxDao<T, ID> extends Dao<T, ID>
```

所有的方法名以"rx"为前缀。返回一个特定类型的``Observable``对象

```
public abstract class RxBaseDaoImpl<DataType extends AbstractOrmLiteEntity, IdType> extends BaseDaoImpl<DataType, IdType> implements IRxDao<DataType, IdType>
```

使用``long``作为ID类型

```
public interface IOrmLiteEntityDAO<DataType extends AbstractOrmLiteEntity> extends Dao<DataType, Long> {
}
```

抽象类

```
public abstract class AbstractBaseDAOImpl<DataType extends AbstractOrmLiteEntity> extends RxBaseDaoImpl<DataType, Long> implements IOrmLiteEntityDAO<DataType> {
    //region Constructors matching super
    protected AbstractBaseDAOImpl(final Class<DataType> poDataClass) throws SQLException {
        super(poDataClass);
    }

    public AbstractBaseDAOImpl(final ConnectionSource poConnectionSource, final Class<DataType> poDataClass) throws SQLException {
        super(poConnectionSource, poDataClass);
    }

    public AbstractBaseDAOImpl(final ConnectionSource poConnectionSource, final DatabaseTableConfig<DataType> poTableConfig) throws SQLException {
        super(poConnectionSource, poTableConfig);
    }
    //endregion
}
```

``DAORepo``变成

```
public class DAORepo extends AbstractBaseDAOImpl<RepoEntity> {
    //region Constructors matching super
    public DAORepo(final ConnectionSource poConnectionSource) throws SQLException {
        this(poConnectionSource, RepoEntity.class);
    }

    public DAORepo(final ConnectionSource poConnectionSource, final Class<RepoEntity> poDataClass) throws SQLException {
        super(poConnectionSource, poDataClass);
    }

    public DAORepo(final ConnectionSource poConnectionSource, final DatabaseTableConfig<RepoEntity> poTableConfig) throws SQLException {
        super(poConnectionSource, poTableConfig);
    }
    //endregion
}
```

``ORMLite``提供``SQLiteOpenHelper``的抽象子类``OrmLiteSQLiteOpenHelper``。

```
public class DatabaseHelperAndroidStarter extends OrmLiteSqliteOpenHelper {
    private static final String DATABASE_NAME = "android_starter.db";
    private static final int DATABASE_VERSION = 1;

    //region Constructor
    public DatabaseHelperAndroidStarter(@NonNull final Context poContext) {
        super(poContext, DATABASE_NAME, null, DATABASE_VERSION);
    }
    //endregion

    //region Methods to override
    @Override
    @SneakyThrows(SQLException.class)
    public void onCreate(@NonNull final SQLiteDatabase poDatabase, @NonNull final ConnectionSource poConnectionSource) {
        TableUtils.createTable(poConnectionSource, RepoEntity.class);
    }

    @Override
    @SneakyThrows(SQLException.class)
    public void onUpgrade(@NonNull final SQLiteDatabase poDatabase, @NonNull final ConnectionSource poConnectionSource, final int piOldVersion, final int piNewVersion) {
        TableUtils.dropTable(poConnectionSource, RepoEntity.class, true);
        onCreate(poDatabase, poConnectionSource);
    }
    //endregion
}
```

``ORMLite``提供``TableUtils``，其可以根据映射的类文件创建或者删除表

现在，我们需要一个``DatabaseHelperAndroidStarter``来处理数据

```
public DatabaseHelperAndroidStarter getDatabaseHelperAndroidStarter(@NonNull final Context poContext) {
    return new DatabaseHelperAndroidStarter(poContext);
}
```

我们能通过下面的方式获得一个``DAORepo``实例

```
public DAORepo getDAORepo(@NonNull final DatabaseHelperAndroidStarter poDatabaseHelperAndroidStarter) {
    return new DAORepo(poDatabaseHelperAndroidStarter.getConnectionSource());
}
```

在``Fragment``中，通过以下方式获取仓库

```
private void rxGetRepos() {
    mSubscriptionGetRepos = daoRepo.rxQueryForAll()
            .subscribeOn(Schedulers.newThread())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(
                    (final List<RepoEntity> ploRepos) -> { // onNext
                        // TODO deal with repos
                    },
                    (final Throwable poException) -> { // onError
                        mSubscriptionGetRepos = null;
                        // TODO deal with error
                    },
                    () -> { // onCompleted
                       mSubscriptionGetRepos = null;
                    }
            );
}
```

但仍然有个问题：如何从网络上获取一个仓库并把它解析然后存储到本地?

我的目标是使用``DTORepo``来与网络通讯,``RepoEntity``映射到数据库。它们有相同名字的相同的字段。因此我需要一个工具用来把DTO转换成实体。这时候，我们会用到[Android Transformer](https://github.com/txusballesteros/android-transformer)

它提供两个主要的注解

 * ``@Mappable``来代表要映射的类
 * ``@Mapped``代表要映射的成员

因此，``RepoEntity``变成:
 
```
@Mappable(with = DTORepo.class)
@DatabaseTable(tableName = "REPO", daoClass = DAORepo.class)
public class RepoEntity extends AbstractOrmLiteEntity implements Serializable {
    @Mapped
    @DatabaseField
    public Integer id;

    @Mapped
    @DatabaseField
    public String name;

    @Mapped
    @DatabaseField
    public String location;

    @Mapped
    @DatabaseField
    public String url;
}
```

现在我们能通过以下方式将DTO转换成实体

```
final Transformer loTransformerRepo = new Transformer.Builder().build(RepoEntity.class);
final RepoEntity loRepo = loTransformerRepo.transform(loDTORepo, RepoEntity.class);
```

[ormgap](https://github.com/stephanenicolas/ormlite-android-gradle-plugin)插件

#### ContentProvider方式
``ContentProvider``和``Cursor``及其派生配合使用异常强大。  

使用[ProviGen](https://github.com/TimotheeJeannin/ProviGen)有以下优势：

 * 声明``ContentProvider``相关类方便
 * 使用``ProviGenProvider``，其是``ContentProvider``的一个子类
 * 提供``SQLiteOpenHelper``默认实现
 * 能够自定义``SQLiteOpenHelper``的实现
 * ``TableBuilder``能够使用流畅的API创建``SQL``表

[MicroOrm](https://github.com/chalup/microorm)

###依赖注入

依赖注入优点：

 * 便于阅读
 * 便于维护
 * 便于测试
 
 ...
 
###MVP架构
[文章](http://hannesdorfmann.com/mosby/)  
你能在这了解到MVP基础，VIEW状态和Loading-Content-Error

另一个有用的工具是[DataBinding](http://developer.android.com/tools/data-binding/guide.html)。它能使``View``和``Model``紧密耦合，双向绑定。

使用``Mosby``来解释MVP

首先设计我们要展示内容的模型类

```
public final class ModelRepoDetail {
    public final RepoEntity repo; // the repo to display

    public ModelRepoDetail(final RepoEntity poRepo) {
        repo = poRepo;
    }
}
```

现在我们定义了对应的``View``接口

```
public interface ViewRepoDetail extends MvpLceView<ModelRepoDetail> {
    void showEmpty();
}
```

下一步是定义``Presenter``

```
@AutoInjector(ApplicationAndroidStarter.class) // to automatically add inject method in component
public final class PresenterRepoDetail extends MvpBasePresenter<ViewRepoDetail> {

    //region Injected fields
    @Inject
    DAORepo daoRepo; // we need the DAO to load the repo from its ID
    //endregion

    //region Fields
    private Subscription mSubscriptionGetRepo; // the RxJava subscription, to destroy it when needed
    //endregion

    //region Constructor
    public PresenterRepoDetail() {
        // inject necessary fields via the component
        ApplicationAndroidStarter.sharedApplication().componentApplication().inject(this);
    }
    //endregion

    //region Visible API
    public void loadRepo(final long plRepoId, final boolean pbPullToRefresh) {
        if (isViewAttached()) {
            getView().showLoading(pbPullToRefresh);
        }
        // get repo asynchronously via RxJava
        rxGetRepo(plRepoId);
    }

    public void onDestroy() {
        // destroy the RxJava subscribtion
        if (mSubscriptionGetRepo != null) {
            mSubscriptionGetRepo.unsubscribe();
            mSubscriptionGetRepo = null;
        }
    }
    //endregion

    //region Reactive job
    private void rxGetRepo(final long plRepoId) {
        mSubscriptionGetRepo = getDatabaseRepo(plRepoId)
                .subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(
                        (final RepoEntity poRepo) -> { // onNext
                            if (isViewAttached()) {
                                getView().setData(new ModelRepoDetail(poRepo));
                                if (poRepo == null) {
                                    getView().showEmpty();
                                } else {
                                    getView().showContent();
                                }
                            }
                        },
                        (final Throwable poException) -> { // onError
                            mSubscriptionGetRepo = null;
                            if (isViewAttached()) {
                                getView().showError(poException, false);
                            }
                        },
                        () -> { // onCompleted
                            mSubscriptionGetRepo = null;
                        }
                );
    }
    //endregion

    //region Database job
    @RxLogObservable
    private Observable<RepoEntity> getDatabaseRepo(final long plRepoId) {
        return daoRepo.rxQueryForId(plRepoId);
    }
    //endregion
}
```

主要代码放在``rxGetRepo``方法中。它从数据库加载数据,然后刷新UI

我们来看一下``FragmentRepoDetail``

```
@FragmentWithArgs
public class FragmentRepoDetail
        extends MvpFragment<ViewRepoDetail, PresenterRepoDetail>
        implements ViewRepoDetail {

    //region FragmentArgs
    @Arg
    Long mItemId;
    //endregion

    //region Fields
    private Switcher mSwitcher;
    //endregion

    //region Injected views
    @Bind(R.id.FragmentRepoDetail_TextView_Empty)
    TextView mTextViewEmpty;
    @Bind(R.id.FragmentRepoDetail_TextView_Error)
    TextView mTextViewError;
    @Bind(R.id.FragmentRepoDetail_ProgressBar_Loading)
    ProgressBar mProgressBarLoading;
    @Bind(R.id.FragmentRepoDetail_ContentView)
    LinearLayout mContentView;
    //endregion

    //region Data-binding
    private FragmentRepoDetailBinding mBinding;
    //endregion

    //region Default constructor
    public FragmentRepoDetail() {
    }
    //endregion

    //region Lifecycle
    @Override
    public void onCreate(final Bundle poSavedInstanceState) {
        super.onCreate(poSavedInstanceState);
        FragmentArgs.inject(this);
    }

    @Override
    public View onCreateView(final LayoutInflater poInflater, final ViewGroup poContainer,
                             final Bundle savedInstanceState) {
        mBinding = DataBindingUtil.inflate(poInflater, R.layout.fragment_repo_detail, poContainer, false);
        return mBinding.getRoot();
    }

    @Override
    public void onViewCreated(final View poView, final Bundle poSavedInstanceState) {
        super.onViewCreated(poView, poSavedInstanceState);

        ButterKnife.bind(this, poView);

        mSwitcher = new Switcher.Builder()
                .withEmptyView(mTextViewEmpty)
                .withProgressView(mProgressBarLoading)
                .withErrorView(mTextViewError)
                .withContentView(mContentView)
                .build();

        loadData(false);
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();

        ButterKnife.unbind(this);

        if (mBinding != null) {
            mBinding.unbind();
            mBinding = null;
        }
    }
    //endregion

    //region MvpFragment
    @Override
    public PresenterRepoDetail createPresenter() {
        return new PresenterRepoDetail();
    }
    //endregion

    //region ViewRepoDetail
    @Override
    public void showEmpty() {
        mSwitcher.showEmptyView();
    }
    //endregion

    //region MvpLceView
    @Override
    public void showContent() {
        mSwitcher.showContentView();
    }

    @Override
    public void showLoading(final boolean pbPullToRefresh) {
        mSwitcher.showProgressView();
    }

    @Override
    public void showError(final Throwable poThrowable, final boolean pbPullToRefresh) {
        mSwitcher.showErrorView();
    }

    @Override
    public void setData(final ModelRepoDetail poData) {
        mBinding.setRepo(poData.repo);

        final Activity loActivity = this.getActivity();
        final CollapsingToolbarLayout loAppBarLayout = (CollapsingToolbarLayout) loActivity.findViewById(R.id.ActivityRepoDetail_ToolbarLayout);
        if (loAppBarLayout != null) {
            loAppBarLayout.setTitle(poData.repo.url);
        }
    }

    @Override
    public void loadData(final boolean pbPullToRefresh) {
        if (mItemId == null) {
            mSwitcher.showErrorView();
        } else {
            getPresenter().loadRepo(mItemId.longValue(), pbPullToRefresh);
        }
    }
    //endregion
}
```

当中的一些注解会在下一节中解释

我们来看一下对应的布局

```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="repo"
            type="fr.guddy.androidstarter.database.entities.RepoEntity"/>
    </data>

    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:id="@+id/FragmentRepoDetail_TextView_Empty"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:gravity="center"
            android:text="@string/empty_repo"/>

        <TextView
            android:id="@+id/FragmentRepoDetail_TextView_Error"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:gravity="center"
            android:text="@string/error_repo"/>

        <ProgressBar
            android:id="@+id/FragmentRepoDetail_ProgressBar_Loading"
            style="?android:attr/progressBarStyleLarge"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"/>

        <LinearLayout
            android:id="@+id/FragmentRepoDetail_ContentView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical">

            <TextView
                android:id="@+id/FragmentRepoDetail_TextView_Name"
                style="?android:attr/textAppearanceLarge"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:padding="16dp"
                android:text="@{repo.name}"
                android:textIsSelectable="true"/>

            <TextView
                android:id="@+id/FragmentRepoDetail_TextView_Location"
                style="?android:attr/textAppearanceMedium"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:padding="16dp"
                android:text="@{repo.location}"
                android:textIsSelectable="true"/>

            <TextView
                android:id="@+id/FragmentRepoDetail_TextView_Url"
                style="?android:attr/textAppearanceSmall"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:padding="16dp"
                android:text="@{repo.url}"
                android:textIsSelectable="true"/>
        </LinearLayout>
    </FrameLayout>
</layout>
```

多亏了MVP架构，我们收获良多
 * 独立的类来加载和展示对应的数据：``Presenter``
 * 要展示的数据: ``Model``
 * 展示的方式: ``View``

###写更轻量的类

####注解和注解处理器的好处
``android-apt``是``Android`` 开发的一大进步。

它允许开发者在``gradle``文件中配置编译时的注解处理。很多库用它来生成模板代码。

####Butter Knife
####FragmentArgs
####IntentBuilder
####Icepick
####OnActivityResult
####Project Lombok
####Switcher

###Testing
[frutilla](https://github.com/ignaciotcrespo/frutilla)  
####Fluent assertions
 * truth
 * AssertJ
 * AssertJ Android
 
####Mocking
####UI testing
####Code coverage

```
android {
   buildTypes {
      debug {
         testCoverageEnabled = true
      }
   }
}
```

###Code quality
 * [CheckStyle](http://checkstyle.sourceforge.net/)
 * [FindBugs](http://findbugs.sourceforge.net/)
 * [PMD](http://pmd.github.io/)
 * [Android Lint](http://tools.android.com/tips/lint)

###Relevant libraries
 