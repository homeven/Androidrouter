# Androidrouter
Androdid组件
简书地址：https://www.jianshu.com/p/b76b85da5740
### 概述

#### 组件化缘由

记得刚开始接触Android开发的时候，只知道MVC分层架构，而且感觉Model，View以及Controller太简单了，也能称之为分层架构，随便写就是MVC。就像在接触设计模式之前，你可能已经写了无数个单例模式，只是那个时候你可能并不知道，你已经在用设计模式了，你不会去想是用DCL还是使用内部类实现的单例优雅。

后来当一个类中的代码上千行之后，就开始想着抽取公共方法作为工具类，使用封装、继承以及多态来优化自己的代码，直到随着业务的发展，在View层的逻辑越来越多，无法抽取时，发现MVC的天花板其实很低，Activity跟Fragment作为View层经常会跟Model层纠缠不清，及时进行抽取之后，也还是很臃肿。MVP的出现，彻底解决了这个问题，解耦Model层跟View层，使得整个项目的代码显得更加简洁。

在项目初期的的时候，感觉MVP还是很不错的，当项目逐渐变大的时候，每次你改动了很小的一部分，你也需要重新编译整个APP，举个例子，就拿购物车来说，我修改了数量比狂的样式，我需要重新编译整个APP，为了加快速度，我可能要开启InstantRun，可能要使用Freeline来加速编译，这并不是我想要的，而且在使用InstantRun之后，output目录下生成的apk是差量包，只能供开发调试，给测试是无法安装的，我要是想通过脚本上传到fir给测试人员，那又得打一个全量的包，并且InstantRun也不是很稳定。

#### 组件化效果

毫无悬念，组件化势在必行，在网上看了很多相关的资料，对组件化有一个初步的了解，然后就开始组件化了，下面1以我自己的项目为例，放两张组件化之前跟之后的图对比一下。

![image](//upload-images.jianshu.io/upload_images/1217209-7f1cdfebfe8db80b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

可以明显的发现我们的Module变多了，就像MVC切换到MVP之后，需要写很多的Presenter组件化最大的好处就是可以模块可以单独开发调试，这样效率一下子就上来了，还是拿购物车举例，购物车实际上就只有一个界面，也就是一个Fragment，加上启动页跟Fragment的父Activity，也就两个界面，可以说想慢都慢不下来，下面就我在组件化过程中遇到的一些问题进行总结一下。

### 正文

#### 指导思想

##### 组件拆分

组件化的目的在于将一个project划分成业务组件、基础组件、路由组件。其中业务组件是相互隔离的，可以单独调试，基础组件提供业务组件所公用的功能，路由组件为业务组件之间通信提供支持。

一般来讲，一个APP可以由一个app壳，然后集成多个Module，这是理想的情况，但是从运营的需求到产品的设计到UI出图，可能你就会对组件化很绝望，并不是那么的理想，很多时候我们程序入口所在的Module实际上跟其它很多Module是关联的，实际上没法拆分，本文将会以这种比较复杂的情况进行组件化分析。

##### 组件隔离

组件化的一个很大的特性在于可以单独调试，但是由于业务组件之间的隔离，所以导致了多个组件之间无法进行通信，其实我觉得是很正常的，既然是单独调试，就必然不应该跟其它的Module间进行依赖，不管是编译期还是运行期都应如此，不然组件化就没有任何意义了，但是由于我们的业务组件都是相互关联的，如果不依赖其他的组件的话，作为一个单独的APP运行有时候是需要参数的，鉴于此，我们可以在Application初始化的时候，新增一个页面作为参数配置，或者直接在Application中固定写死。

##### 核心法则

不管我们如何划分，如何依赖，组件间的关系都要严格遵守一个准则：**编译器隔离，运行期按需依赖**。

#### 整体架构

![image](//upload-images.jianshu.io/upload_images/1217209-10d354c27c803bfa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/784/format/webp)

通过组件化将项目按照业务进行化分成**GoodsModule**，**CartModule**，**UserModule**，**OrderModule**四个模块，模块间通过**RouterModule**进行通信，也就是说业务组件依赖于路由组件，**RouterModule**依赖于**Base**，也就是**BaseModule**，**LibraryModule**，基础库跟第三方库，然后MainModule实际上相当于程序的入口跟容器，通过**MainModule**依赖上述四个Module，完成整个APP的打包。

当然在单独调试的时候，**GoodsModule**，**CartModule**，**UserModule**，**OrderModule**又各自成为一个APP，可以单独进行调试，这样就实现了APP的组件化，下面就组件化过程中遇到的一些问题总结一下。

#### 组件化分析

在组件化的过程中，由于Module之间是隔离的，所以就产生了一系�列问题，现在就组件化前后的遇到的问题总结如下：

*   组件划分：如何根据业务对项目进行Module划分
*   模式切换：如何使得APP在单独调试跟整体调试自由切换
*   资源冲突：当我们创建了多个Module的时候，如何解决相同资源文件名合并的冲突
*   依赖关系：多个Module之间如何引用一些共同的library以及工具类
*   组件通信：组件化之后，Module之间是相互隔离的，如何进行UI跳转以及方法调用
*   入口参数：我们知道组件之间是有联系的，所以在单独调试的时候如何拿到其它的Module传递过来的参数

接下来会根据�这几个问题，提出对应的解决方法

#### 组件划分

##### 业务划分

由于我们做的是一个电商项目，网上也查找了很多资料，感觉他们举的例子都有些过于简单，因为模块间基本上没有什么耦合，所以很好拆分，不过还是很感谢他们提供了一种解决思路。玩过京东，淘宝都知道，大致分为几个大的模块：商品模块，购物车模块，订单模块，用户模块。没错，我也是这么拆分我们APP的。但是拆着拆着就发现问题了，模块间耦合性太高，我们过了SplashActivity之后就是MainActivity，看图说话

![image](//upload-images.jianshu.io/upload_images/1217209-abf5c751599d1bba.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/720/format/webp)

所以网上的一些一进来就是一个空的APP壳的方法并不适用，从一开始就遇到了这个棘手的问题，有点尴尬，按照之前的模块划分，在用户登陆的情况下**MainModule**一进来就必须拿到**GoodsModule**，**CartModule**以及**UserModule**中的三个Fragment。所以首先必须得解决这个问题，很显然之前的使用一个APP壳来合并多个Module的情况并不适用，起初我直接定义了一个**MainModule**，然后让他直接引用多个Module，那么MainModule就承担了APP壳的功能，这样一来，就可以解决**MainModule**对其它Module的引用问题，但是违背了组件化的业务组件隔离的原则。

所以不能让**MainModule**依赖另外三个Module，但是如果我不引用其他的Module，那么很显然我无法拿到这四个Fragment的引用，有一点可以很明确，那就是编译期业务Module之间必须不可见，这点是毫无疑问的。但是运行期是可见的，因为所有的Module在运行期间肯定都是通过直接或者间接依赖，不然有些Module就没用了，在运行时获取实例，那么很自然地就会想到反射了，没错就是反射。

##### 依赖划分

除了业务模块之外，我们还会有一些公用的工具类以及资源文件，也就是Base类，比如说多个Module共同使用的资源文件，我们都可以放在一个Module里面，另外就是还有第三方依赖，这里我新建了两个Module一个是BaseModule，一个是LibraryModule。整体关系如下

```
业务组件——>路由组件——>基础组件

```

#### 模式切换

##### 定义开关

切换的时候需要一个开关，来表示是单个Module间运行还是多个Module间运行，很容易想到是一个布boolean类型的标志，可能你也想到了，在gradle.properties中来定义，网上好像都是这么做的，实际上我们还可以在BaseModule以及LibraryModule定义，原因很简单，只需要所有的Module中都能够访问就行了，只要遵循这个原则都是OK的，只是在gradle.properties中定义跟使用都比较方便。

```
isDebug=false//Debug还是Release
isModuleRun=true//是否单Module运行

```

这里我不仅仅定义了**isModuleRun**，还定义了isDebug，是不是感觉有些奇怪，不是可以通过BuildConfig.Debug来判断当前是否是Debug模式么，因为我们的url配置信息都是写在BaseModule中以便于所有的Module调用，他是一个Library，关于Library这里还有一个问题注意下，由于Library的Module打包方式是使用release模式打包的，所以BuildConfig.Debug永远是false，所以我们需要额外定义一个变量isDebug，然后手动在Debug跟Release中进行切换，然后在BaseModule的gradle中进行判断

```
if (isDebug.toBoolean()) {
    //debug模式
    buildConfigField "String", "AlphaUrl", "\"${url["debug"]}\""

} else {
    //release模式
    buildConfigField "String", "AlphaUrl", "\"${url["release"]}\""

}

```

##### 使用开关

###### Application

isModuleRun为false的时候，Application跟AndroidManifest都是以Library的形式参与编译，不需要启动的Activity以及自定义的Application反之则需要。

**isModuleRun=false**

无序修改

```
<application
    android:allowBackup="true"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">
 </application>

```

**isModuleRun=false**

在main/debug目录下新建一个AndroidManifest.xml文件

```
<application
    android:name=".debug.GoodsApplication"
    android:allowBackup="true"
    android:label="@string/goods_name"
    android:supportsRtl="true"
    tools:replace="android:label"
    android:theme="@style/AppTheme">
    <activity android:name=".GoodsActivity">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
 </application>

```

**引用方式**

在Module的gradle目录下进行引用

修改插件

```
if (isModuleRun.toBoolean()) {
    apply plugin: 'com.android.application'
} else {
    apply plugin: 'com.android.library'
}

```

新增applicationId

```
if (isModuleRun.toBoolean()) {
    applicationId "com.wustor.cartmoudle"
}

```

切换AndroidManifest文件

```
sourceSets {
    main {
        if (isModuleRun.toBoolean()) {
            manifest.srcFile 'src/main/debug/AndroidManifest.xml'
        } else {
            manifest.srcFile 'src/main/AndroidManifest.xml'
            java {
                //全部Module一起编译的时候剔除debug目录
                exclude '**/debug/**'
            }
        }
    }
}

```

#### 资源冲突

假如我们在CartModule中定义了一个Application，然后在当前Module中的strings.xml中定义了app_name，同时在OrderModule中的strings.xml中也定义了这个app_name，那么合并你的时候就会出现冲突，我们只可以通过将上述字段分别改成cart_name跟order_name来解决这个问题，在严格的开发规范下，可以通过这种差异化命名来解决，因为不同的Module基本上资源文件的名称基本都不一样，即时冲突也是少量的冲突，很容易解决。

当然除了这种方式之外可以在build.gradle中给资源文件名添加前缀

```
resourcePrefix "cart_"

```

可以强行检查，命名都需要价格前缀，这样反而违背了组件化的初衷，使得操作变麻烦了，不过感觉这种方式不是很有必要，当然有时候还可能出现图片名字相同，这个其实可以还原到组件化之前的项目中分析，是不可能发生的事情，所以归根到底还是没有良好的开发规范跟开发习惯造成，没必要为这种去做一些修改，毕竟**约定大于配置**。

#### 依赖配置

通过最开始的整体架构图可以看出来，凡是能够在Library跟Application之间进行切换的Module毫无疑问是需要依赖我们Base的两个Module的，其实可以合并成一个Module，我这里分了两个，一个是BaseModule，一个是LibraryModule。下面通过build.gradle中的配置来梳理一下他们的依赖关系：

##### MainModule

```
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    compile project(':routermodule')
}

```

编译期间组件进行隔离，所以MainModule只依赖了RouterModule，刚才说的还有在运行期按需依赖，这里是通过gradle的脚本实现控制的

```
//编译期组件隔离，运行期组件按需依赖
//mainModule需要跟cartModule,goodsModule,usersModule进行交互，所以在运行期添加了依赖
def tasks = project.gradle.startParameter.taskNames
for (String task : tasks) {
    def upperName = task.toUpperCase()
    if (upperName.contains("ASSEMBLE") || upperName.contains("INSTALL")) {
        dependencies.add("compile", project.project(':' + 'cartmodule'))
        dependencies.add("compile", project.project(':' + 'goodsmodule'))
        dependencies.add("compile", project.project(':' + 'usermodule'))
        dependencies.add("compile", project.project(':' + 'ordermodule'))
    }
}

```

##### BusinessModule

这里指的是Goods/Cart/User/OrderModule，其实是平行的

```
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    compile project(':routermodule')
}

```

业务Module依赖于RouterModule

##### RouterModule

```
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    compile project(':modulelib')
    compile 'com.alibaba:arouter-api:1.2.1.1'
}

```

RouterModule依赖了LibraryModule

##### BaseModule

```
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    testCompile 'junit:junit:4.12'
    compile project(':librarymodule')
}

```

BaseModule作为一个基础库，依赖了LibraryModule

##### LibraryModule

这个作为最底层的劳苦大众，实际上就是提供了一个依赖，所以就没有什么好依赖，只能自己跟自己玩儿。

> 所以到这里的话，基本的依赖关系已经很清楚了，知道了整个架构图，接下来进行施工也就很简单了

#### 组件通信

其实在当初进行模块划分的时候，是根据业务来的，所以当我们进入到一个模块之后，大部分逻辑应该还是在这个模块内进行处理的，但是偶尔还是会跟别的Module进行打交道，看一个界面

![image](//upload-images.jianshu.io/upload_images/1217209-d1746e18f2863ca5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

就拿**GoodsModule**跟**CartModule**来说，这两个Module是可以进行相互跳转的，在**GoodsModule**的列表页面点击购物车图标可以进入到**CartModule**的购物车列表，购物车列表点击商品也可以进入**GoodsModule**的商品详情页。除了这个跳转实际上还有变量的获取，比如在首页，我需要同时获取到**GoodsModule**中的HomeFragment、SortFragment，**CartModule**中的CartFragment，**UserModule**中的MineFragment。我是在MainModule中直接依赖了四个业务Module，实际上可以不这样，我们也可以使用Arouter来进行获取Fragment的实例。

##### 获取实例

其实这里的实例大多数情况下指的就是Fragment，下面以Fragment为例，别的实例如法炮制即可

*   反射获取

由于模块间是隔离的，所以我们没办法直接创建Fragment的实例，那么这个时候其实很容易想到的就是反射，发射可谓无所不能，下面贴一下代码。

```
//获取Fragment实例
public static Fragment getFragment(String className) {
    Fragment fragment;
    try {
        Class fragmentClass = Class.forName(className);
        fragment = (Fragment) fragmentClass.newInstance();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    return fragment;
}

```

*   Arouter

[Arouter](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Falibaba%2FARouter)是阿里巴巴退出的一款路由框架，在组件中进行路由操作表方便，下面举例说明

目标Fragment中加入注解

```
@Route(path = "cart/fragment")
public class CartFragement extends BaseFragment{
}

```

在任何地方获取实例

```
Fragmetn fragment = (Fragment) ARouter.getInstance().build("/cart/fragment").navigation();

```

##### 方法调用

在不同的Module之间都存在方法的调用，我们可以在每个Module里面定义一个接口，并且实现这个接口，然后在需要调用的地方获取到这个接口，然后进行方法调用即可。为了统一管理，我们把每个Module的接口都定义在RouterModule里面，然后由于各个业务Module都依赖于这个RouteModule，然后只需要通过反射获取到这个接口，进行方法调用就可以了。

![image](//upload-images.jianshu.io/upload_images/1217209-25719f5ffb1f40bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

ModuleCall�

Module之间回调的接口

```
public interface ModuleCall {
   //调用init方法可以传递Context参数
    void initContext(Context context);
}

```

Service接口继承自ModuleCall可以定义一些回调方法供本身之外的其他Module进行调用

```
public interface AppService extends ModuleCall {
    //TODO 调用方法自定义
    void showHome();
    void finish();

}

```

Impl实现类则是对应在每个Module中的具体回调，是实现Service接口的直接子类

```
public class AppServiceImpl implements AppService {
    @Override
    public void showHome() {
    }
    @Override
    public void finish() {
    }
    @Override
    public void initContext(Context context) {
    }
}

```

下面还是通过反射跟Arouter两种方式进行说明

*   反射调用

    ```
    public static Object getModuleCall(String name) {
        T t;
        try {
            Class aClass = Class.forName(name);
            t = (T) aClass.newInstance();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        return t;
    }

    ```

获取接口

```
AppService appService = (AppService) ReflectUtils.getModuleCall(Path.APP_SERVICE);

```

其实跟获取Fragment实例一样，通过类名来获取对应的接口，然后调用对应的方法就行，有一点需要注意的就是，如果获取的接口之后调用的方法需要传入Context参数，那么在调用接口方法之前必须先调用initContext方法才能使用传入的Context，不然会报空指针异常。

*   Arouter

Arouter中有一个IProvider接口，如下

```
public interface IProvider {
    void init(Context var1);
}

```

其实IProvider跟上面的ModuleCall是一样的，只不过他在获取到接口实例之后，就会调用initContext方法，其中的Context来自ARouter.init(this)中传入的参数，不需要我们再手动调用initContext。

目标类中注入路径

```
@Route(path = Path.APP_SERVICE)
public class AppServiceImpl implements AppService {
    private Context mContext;
    @Override
    public void showHome() {
        Log.d("go--->", "home--->");
    }

    @Override
    public void finish() {
    }

    @Override
    public void init(Context context) {
        mContext = context;
    }
}

```

任意地方获取目标类

```
AppService appService = (AppService) RouterUtils.navigation(Path.APP_SERVICE);

```

然后调用方法即可

##### UI跳转

跳转基本上指的就是Activity之间的跳转，废话不多说，依旧是Arouter跟反射

*   反射

    ```
    //将类名转化为目标类
    public static void startActivityWithName(Context context, String name) {
        try {
            Class clazz = Class.forName(name);
            startActivity(context, clazz);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
    }
    //获取Intent
    public static Intent getIntent(Context context, Class clazz) {
        return new Intent(context, clazz);
    }
    //启动Activity
    public static void startActivity(Context context, Class clazz) {
        context.startActivity(getIntent(context, clazz));
    }

    ```

*   Arouter

    将目标Activity注册到Arouter

    ```
    @Route(path = Path.CART_MOUDLE_CART)
    public class CartActivity extends BaseActivity<UselessPresenter, UselessBean> {
    }

    ```

    启动目标Activity

    ```
     ARouter.getInstance().build(Path.CART_MOUDLE_CART).navigation()

    ```

#### 入口参数

##### Application

当组件单独运行的时候，每个Module自成一个APK，那么就意味着会有多个Application，很显然我们不愿意重复写这么多代码，所以我们只需要定义一个ModuleApplication即可，其它的Application直接继承此ModuleApplication就OK了，看一下结构图：

![image](//upload-images.jianshu.io/upload_images/1217209-15241de73c1c31a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

实际上所有的逻辑都是在ModuleApplication中，业务Module分别有自己的子类，通过子类可以对Application做一些自己的定制化操作。

##### 无参原因

之前在网上看到过携程以及得到的组件化，他们从MainModule进入到别的Module貌似都是不需要传参数的，所以不管是组件单独调试还是所有的Module一起远行对于从ModuleA跳转到ModuleB都是不需要传参的。但是很多时候不同的Module间跳转是需要传参的，就拿购物车来说，我单独调试的时候是需要知道用户的加密的userId，才能向服务器请求数据，如果是多个Module一起运行，访问购物车的时候，是可以从别的Module取到userId的，单独调试的时候就没法获取到，也就是入口的时候没有参数对购物车进行初始化。

##### 解决方式

因为当我们在组件化进行调试的时候，我们每个Module在cartmodule/src/main/debug目录下有自己的Application，对于入口参数比较简单的情况，我们可以直接在Application中写死，而对于一些比较复杂的或者动态的参数，我们可以继续在此目录下新疆一个Activity来配置我们单Module调试所需要的参数，然后在整个项目进行编译的时候剔除debug目录下的文件。

```
sourceSets {
    main {
        if (isModuleRun.toBoolean()) {
            manifest.srcFile 'src/main/debug/AndroidManifest.xml'
        } else {
            manifest.srcFile 'src/main/AndroidManifest.xml'
            java {
                //release 时 debug 目录下文件不需要合并到主工程
                exclude '**/debug/**'
            }
        }
    }
}

```

### 总结

项目组件化运行了一段时间，通过划分Module，单独调试，确实大大提升了开发效率，随着使用的时间的推移，也在对组件化的理解也进一步加深，也在不断地完善，下面几点是在组件化过程中总结的一些经验。

*   **Module划分**：在划分Module的时候没必要划分地太细，但是要严格按照业务来划分，这样单独调试对于习作开发才有意义。
*   **Module隔离**：业务Module之间应该是相互隔离不可见的，不能相互依赖，如果相互之间需要通信，则必须经过路由转发，便于统一管理。
*   **面向接口编程**：不管是也业务Module还是BaseModule、LibraryModule以及RouterModule，在对外提供服务的时候尽可能的以接口的形式，不同的Module对外提供的服务接口应该都有一个共同的抽象父类，便于管理。
*   **防止循环依赖**：循环依赖就是A依赖B，B依赖A,在运行期间动态添加依赖的时候，一定要考虑这个依赖是否被添加到项目中去了，所谓添加到项目中就是但凡被其它的Module进行依赖过就算添加进项目中，不然很容易造成循环依赖。