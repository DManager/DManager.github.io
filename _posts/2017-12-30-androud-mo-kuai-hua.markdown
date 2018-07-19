---
layout: post
title: "二维火掌柜Android模块化架构实践"
date: 2017-12-30 18:18:59 +0800
comments: true
categories: android
author: [石胆, 牛轧糖]
---



# 模块化工作之前思考的问题



## 1. 模块化结构图

模块化之前项目结构：

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/1.png)



模块化调整后项目结构：

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/2.png)



模块化拆分原则：

1. 依赖关系只能上层依赖下层，不能反向依赖。

   上下层即上图中纵向关系，上层模块只能单向依赖下层的模块，如果两个模块间配置的是双向依赖，会出现循环依赖问题。例如：`A -> B`A模块依赖B模块， `B ->A`  B模块又依赖A模块，`A -> B`A模块又依赖B模块.... 陷入死循环了。

<!-- more -->

2. 平级模块不能相互依赖（不能单向依赖，更不能双向依赖）。

|      | 模块间相互依赖              | 模块不相互依赖             |
| ---- | -------------------- | ------------------- |
| 代码关系 | 代码相互耦合，没有清晰的边界       | 代码相互隔离              |
| 打包速度 | 每次都需要编译所有代码，速度慢      | 任意卸载无关模块，可以提高编译速度   |
| 定制功能 | 代码都耦合在一起，很难删除指定功能再打包 | 可以编译一个不包含任意功能模块的安装包 |

​    可能要疑问了，为什么平级模块(上图中横向的几个模块)间不让相互依赖，甚至单向依赖都不行？

​    拿业务层举例，如果`Member`，`Recharge`，`Shop`几个模块间允许单向依赖，模块间的依赖图会变成树形结构而不是平级结构。

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/12.png)

​     只要允许平级模块间可以相互依赖，一定会变成类似上图的结构，其实上图画的还是比较清晰的关系，假设G模块依赖B模块，整个依赖关系更无法直视了。也许你会说：“我任性就让平级模块间可以互相依赖，整个项目运行完全没问题”

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/13.png)

​    那如果新的需求要去除C模块的功能呢？平级模块间如果没有依赖关系，可以直接删除掉C模块。但是如果按照上图的依赖关系来操作，需要重新设置`A，F，G`模块间的依赖关系，A模块依赖C模块（`A ->C `）肯定存在A直接依赖C模块中的代码，删除掉C模块之后，这些A模块中依赖的代码也需要修改。

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/4.png)

​    模块间代码隔离在架构上更清晰，但是也会带来一些挑战，平级模块间代码隔离也导致无法直接访问到对方（例如：`Member`要访问`Recharge`的功能，抱歉因为代码是隔离的不能直接访问），下文中`模块生命周期管理`，`模块间通信`，`模块间跳转` 3个部分，都是围绕代码隔离带来的问题，提出的对应解决方案。



3. 先拆分模块，再针对每个模块逐步优化

   模块化工作是属于架构优化，执行的过程中会看到很多之前设计不合理的代码细节，建议按照模块化结构先把整个项目拆分开，在WIKI上创建`优化清单`，大家在模块化过程中发现的待优化点，都可以统一记录在此清单中，模块化完成后，再针对每一个模块逐步进行优化。




## 2. 模块生命周期管理

什么是模块生命周期？

> 生命周期就是指一个对象的生老病死。

​    模块生命周期的目的是在应用的启动与关闭时，模块可以获知到这两种状态。

何时需要管理模块的生命周期？

​    在应用启动时，如果模块需要在此时初始化一些对象，这种场景需要使用模块生命周期管理，例如：统计崩溃的第三方框架已经单独放到一个仓库中，但是此框架需要在Application中进行初始化。如果没有类似的需求，项目中不需要管理模块的生命周期。



![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/3.png)



```java
public class RestApplication extends Application {

    private ModuleManager mModuleManager;

    @Override
    public void onCreate() {
        super.onCreate();
        mModuleManager = new ModuleManager();
        mModuleManager.addModule(new MemberModule());
        mModuleManager.addModule(new RechargeModule());
        mModuleManager.onCreate();
    }

    @Override
    public void onTerminate() {
        super.onTerminate();
        // NOTE: onTerminate函数并不保证app关闭时一定会被调用
        // 此处逻辑可以移到app关闭逻辑中（通常是首页监听back的地方）
        mModuleManager.onDestory();
    }
}
```



```java
public class ModuleManager {

    private List<ModuleInterface> mModuleList;

    public ModuleManager() {
        mModuleList = new ArrayList<ModuleInterface>();
    }

    public void addModule(ModuleInterface module) {
        if (mModuleList == null) {
            return;
        }
        mModuleList.add(module);
    }

    public void onCreate() {
        if (mModuleList == null) {
            return;
        }

        for (ModuleInterface module : mModuleList) {
            module.registerFacade();
            module.onApplicationCreate();
        }
    }

    public void onDestory() {
        if (mModuleList == null) {
            return;
        }

        for (ModuleInterface module : mModuleList) {
            module.onApplicationDestory();
        }
    }
}
```



```java
public interface ModuleInterface {
    /**
     * 启动应用时触发
     */
    void onApplicationCreate();

    /**
     * 退出应用时触发
     */
    void onApplicationDestory();
}
```



```java
public class RechargeModule implements ModuleInterface {

    @Override
    public void onApplicationCreate() {
        ......
    }

    @Override
    public void onApplicationDestory() {
       ......
    }
}

```



如果卸载掉`Member`会发现找不到`MemberModule`类，不过有以下几种方案可以解决：

1. 通过反射获取`MemberModule`。
2. `MemberModule`添加自定义注解，编译期查如果能够查找到再进行注册（可以避免使用反射）。
3. 再创建一个空的`Member`仓库，里面只放一个`MemberModule`类。



## 3. 模块间通信

什么是模块间通信？

> 通信，指人与人或人与自然之间通过某种行为或媒介进行的信息交流与传递，从广义上指需要信息的双方或多方在不违背各自意愿的情况下采用任意方法，任意媒质，将信息从某方准确安全地传送到另方。

​    模块间通信就是为了获取对方模块中的信息，或者通知对方模块去处理某件事情，但是上面也提到了模块间代码是隔离的。后续就是展示讨论下代码隔离的情况下，通过模块间通信手段达到访问对方模块功能的目的。    



1. 把需要调用的代码下沉



![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/5.png)



​    这个解决方案是最简单的，但是架构设计上也是最不合理的。原本划分到不同业务模块的功能，仅仅其他模块需要访问，就移动到不合理的层级上，而且会导致下层越来越臃肿。



2. 系统提供的通信方式

​    Android系统提供的通信方式有很多，例如：*LocalBroadcastReceiver*，*Socket*，*ContentProvider*，*AIDL*。其中`LocalBroadcastReceiver`的好处是比`BroadcastReceiver` 效率更高，且仅针对应用内广播，外部无法访问所以也会更安全。

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/6.png)



```java
public class SettingReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
        if(intent != null && "package.SETTING_ACTION".equals(intent.getAction())){
              ......
        }
    }
}
```



```xml
        <receiver android:name=".SettingReceiver">
            <intent-filter>
                <action android:name="package.SETTING_ACTION"></action>
            </intent-filter>
        </receiver>
```



```java
public class SettingModule implements ModuleInterface {
  
    private SettingReceiver mReceiver = new SettingReceiver();
    
    @Override
    public void onApplicationCreate() {
        LocalBroadcastManager broadcastManager = LocalBroadcastManager.getInstance(this);
        IntentFilter intentFilter = new IntentFilter("package.SETTING_ACTION");
        broadcastManager.registerReceiver(mReceiver, intentFilter);
    }

    @Override
    public void onApplicationDestory() {
        LocalBroadcastManager broadcastManager = LocalBroadcastManager.getInstance(this);
        broadcastManager.unregisterReceiver(mReceiver);
    }
}
```

优点：Android系统原生支持，直接使用相应API就可以进行通信。

缺点：如果需要返回值处理起来比较麻烦，需要使用回调函数等方式实现。



3. 消息总线

​    可选第三方框架`EventBus`，`Otto`，`RxBus`，也可以自己开发一个消息总线，如果不考虑线程安全，消息量很小的前提下，一个类2～3个函数就能满足需求。



![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/7.png)



优点：把Event接口定义放在Common层中，直接post即可，非常方便。

缺点：最大的问题是，通信总线框架太好用，原本可以使用`startActivityForResult`或者回调能解决的问题，都可能会直接使用消息总线框架处理。很多项目到最后都变成Event满天飞，容易出现各种问题。



4. Facade接口

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/8.png)



一个模块对外提供一个Facade接口，通过反射获取此接口的实例，模块对外提供的接口统一成一个。



5. 依赖注入

​      通过ARouter的@Autowired实现对外提供服务。



## 4. 模块间页面跳转

​    模块间页面跳转是A模块一个页面希望跳转到B模块中的一个页面，因为AB模块代码独立，所以A模块无法访问到B模块中的类。模块间页面跳转也可以使用模块间通信解决，模块A调用模块B的一个函数，模块B的这个函数用于跳转到此模块中的指定页面，这并不是最好的解决方案，以下提供三种解决方案供参考：



1. 通过scheme跳转


```xml
<activity android:name=".SettingActivity">
  <intent-filter>
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <data android:scheme="2dfire"
          android:host="rest"
          android:path="/Setting"/>
  </intent-filter>
</activity>
```



```java
   Uri data = Uri.parse("2dfire://rest/Setting");
   Intent intent = new Intent(Intent.ACTION_VIEW, data);
   startActivity(intent);
```

​    通过scheme进行模块间跳转是成本最小的，核心是关注各模块间path不能重复，可以通过`模块/功能/页面功能`的形式避免重复。    



scheme 跳转方案考虑

+ 每个 Activity 都在 Manifest 中配置一个 scheme 值

该方法就是通常的 scheme uri 配置，每个唤起都可以通过 uri 跳转

+ 都通过着陆页面进行分发，该 Activity 直接透明即可

该方法的核心是，着陆页面 `SchemeFilterActivity` 的配置，只写 scheme 和 host，不写 path，这样就会接管所有的跳转

```xml
 <activity android:name=".SchemeFilterActivity">
    <intent-filter>
      <action android:name="android.intent.action.VIEW"/>
      <category android:name="android.intent.category.DEFAULT"/>
      <data android:scheme="2dfire"
            android:host="rest.com"/>
    </intent-filter>
</activity>
```

​  然后内部再通过如下方式进行分发，当然也可以用第三方库的例如 ARouter 进行跳转

```java
Uri data = getIntent().getData();
Intent intent = new Intent(Intent.ACTION_VIEW, data);
startActivity(intent);
```



第二种方法的好处是，外部唤起时可以通过着陆页面进行统一分发，而不需要每个 Activity 都去配置 Manifest，具有更强的可更改性

同时后期也可以进行更多的逻辑控制，包括权限控制 ，是否跳转至登录界面等。



2. 通过路由框架跳转


```java
// 定义路径
public static final String SETTING = "/rest/Setting";

// 定义路径对应页面
@Route(path = Paths.SETTING)
public class SettingActivity extends Acitvity

// 跳转到指定路径页面
ARouter.getInstance().build(SETTING).navigation();
```

​    通过路由框架进行模块间跳转，需要引入第三方框架。如果跳转过程中不需要进行拦截处理（即不需要使用`拦截器`），还是建议使用第一种方案。



3. 自定义规则

   上面第一种方案是通过scheme进行隐式跳转，不使用显示跳转的原因是因为模块间代码隔离，无法直接访问到指定的其他模块的类。但是可以为每个页面定义一个唯一标记（例如：字符串，数字），

   > 通过在`模块生命周期管理`应用启动时各模块统一进行注册，调用跳转函数时可以从注册中的规则中进行遍历，并执行相应规则对应的代码进行跳转。




# 模块化工作推进



## 1. 模块化工作可执行的四种方案

一 所有人停止业务需求开发

​    所有人都停止业务需求的开发，单独抽出一段时间进行模块化工作，不会遇到任何新增业务代码与模块化工作导致的代码冲突问题，缺点是模块化过程中不会新增任何功能。



二 部分人停止业务需求开发

1. 整个团队抽出部分人员完成模块化工作

   假设团队有3个小组，3个小组分别负责A，B，C功能的开发。

   （1）可以一个小组完成A功能的模块后，下一个小组再开始，这样避免小组间新增代码导致的互相冲突。

   （2）第一个小组抽出一个或多个人进行A功能的模块化工作，这样即使代码冲突也是在小组范围内的功能代码，比较容易处理解决。



2. 整个团队只抽出一个人完成模块化工作

   只有一个人停止业务需求开发，独自一人完成所有模块化工作，人力有限模块化进度会非常慢。

   ​


三 所有人都不停止业务需求开发

​    这种方式是团队中每个人都要进行模块化工作。需要把模块化工作拆分为更细的粒度，可以在团队开发业务需求之余进行模块化工作。



## 2. 我们团队选择的模块化工作方案

​    一切脱离实际情况的讨论都是耍流氓，所以每个团队应该因地制宜的进行选择。我们团队选择的是最后一种 “ 所有人都不停止开发 ”。在团队WIKI上列出所有模块涉及功能的页面清单。

> | 所属模块 |  功能  |   入口   |        类名        |  状态  |     时间     |
> | :--: | :--: | :----: | :--------------: | :--: | :--------: |
> | 设置模块 | 功能设置 | 首页-左侧栏 | SettingsActivity | 已完成  | 2017-06-01 |



页面清单的好处：

1. 可以统计总工作量
  在模式化实施前，通过列出需要拆分的9个模块所有页面的清单，可以直接统计出此次工作的总工作量。

2. 任务分配与进度统计
  大家可以相对平均的分配任务，每完成一个页面可以在清单中标记状态，也便于统计整体进度。

3. 减少与新增代码的冲突
  因为可以清晰展示每个页面的状态，所以有新增需求时，如果此页面未模块化，可以认领模块化工作并完成新增需求，如果已经模块化可以查看到相关处理人员，并当面沟通解决方案。

4. 便于回归测试

   所有改动都记录在页面清单中，测试人员仅针对标记为已完成的页面进行测试，不需要对整个应用进行全覆盖测试，减少测试人员的工作量。




## 3. 模块化工作流程

如何有效避免模块化工作代码合并冲突？

团队开发是基于git flow，所以后续流程也是基于git flow设计的。



原始git flow流程图（模块化过程中，业务流畅依然按照此流程进行）：

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/git-workflow.png)

模块化工作git flow流程图：

![git-workflow](/assets/images/2017-12-30-androud-mo-kuai-hua/9.png)





1. 全组12个人，每3人一组，共4个小组，每个小组对应一个Feature。
2. Feature中标记的数字是每个成员提交到小组Feature的代码节点。
3. Module Develop是新增分支，各小组Module Feature每周统一合并到此分支，并且合并后开发人员需要进行自测并修复发现的问题。如果有项目发版Release分支合并到develop之后，此Release也会合并到Module Develop，每次发版后也会合并到Module Develop，减少最终Module Develop时的冲突，上图未画出此场景，避免图画的过于复杂。
4. Module Develop存在的核心目的是Module Feature可以多次提交到此分支，然后能够合并到任意业务需求分支一起发布到线上。





# 工具使用



## 1. 模块的加载与卸载



1. `gradle.properties`文件

```groovy
# 会员模块
ImportMember=true
# 导入源码形式还是Maven形式
ImportTheSourceCode=true
```

项目`gradle.properties`中设置常量，true为加载指定模块，反之卸载。

2. `settings.gradle`文件

```groovy
if (ImportMember.toBoolean() && ImportTheSourceCode.toBoolean()) {
    include ':member'
    project(':member').projectDir = new File('../ManagerMemberModule')
}
```

根据常量判断是否引入指定的子工程（member模块）

3. `build.gradle`文件

```groovy
if (ImportMember.toBoolean()) {
    if (ImportTheSourceCode.toBoolean()){
        compile project(':member')
    } else {
        compile rootProject.ext.tdfDependencies["tdfMember"]
    }
}
```

根据常量判断是否引入本地Android模块作为依赖项。

```groovy
tasks.all {
    if ("assembleRelease".equalsIgnoreCase(it.name)) {
        it.doFirst() {
            if (!ImportMember.toBoolean() || .....) {
              throw new GradleException('Failed to import model. Please check.');
            }
        }
    }
}
```

编译release包时自动检测，避免编译出的apk存在遗漏部分模块功能的情况。





## 2. 查看框架依赖情况

​    如果发现项目中第三方框架比较多或者出现有同框架不同版本的情况，可以通过以下命令查看项目中都存在哪些第三方框架及其依赖关系。

```shell
./gradlew app:dependencies > dependencies.txt
```

 ![dependencies](/assets/images/2017-12-30-androud-mo-kuai-hua/dependencies.png)



## 3. 善用重构工具

​    模块化工作过程中，工作量最大的是移动文件到不同的仓库中，并解决因移动而导致的各种引用问题。文件移动常见的做法是自己手动把文件移动到指定文件夹下，不过通过Andorid Studio提供的Refactor Move工具，只需要设置目标位置，工具会帮你自动处理所有引用问题，避免很多枯燥且容易出错的细节操作。

![refactor_move](/assets/images/2017-12-30-androud-mo-kuai-hua/refactor_move.png)



## 4. 深入分析模块间依赖关系



![analyze](/assets/images/2017-12-30-androud-mo-kuai-hua/analyze.png)



![Analyze_info](/assets/images/2017-12-30-androud-mo-kuai-hua/Analyze_info.png)

  使用Analyze -> Analyze Dependencies分析模块间的代码与资源依赖，通过一段时间的分析后，可以获取到所有模块间依赖的文件清单，通过此清单可以很容易进行模块间解耦。



## 5. Git多仓库管理工具

​    每个模块放入相应的仓库之后，整个项目的仓库会增加很多。

>1. 把不需要变动源码的仓库改成`Maven aar`形式进行依赖
>2. 使用多仓库管理工具更高效一些，团队尝试过`git submodule`， `Google Repo`，团队小伙伴自定义工具[mgit](https://github.com/FamliarMan/MGit)



# 参考资料

[[*微信*Android*模块化*架构重构实践](https://mp.weixin.qq.com/s/6Q818XA5FaHd7jJMFBG60w)]

[Android 模块化探索与实践](https://zhuanlan.zhihu.com/p/26744821)



总结：

​        模块化工作在设计阶段把所有细节都考虑到不太可能，所以制定一个规划之后就可以执行了，如果执行过程中遇到新的问题，再及时分析并解决问题。模块化过程中遇到`Butter Knife`，`Dagger 2`，`ARouter`，`Gradle`，`DataBinding` 框架相关的问题，只要熟悉框架流程与源码都可以很快解决，就不再一一列举。
