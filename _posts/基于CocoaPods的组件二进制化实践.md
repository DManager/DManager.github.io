---
title: 基于 CocoaPods 的组件二进制化实践
date: 2019-01-21 19:49:08
tags: [CocoaPods,Ruby,二进制化]
categories: iOS
author: 青木
---

火掌柜 iOS 客户端经过近两年的组件化推进，组件数量已经颇具规模，达到了近 100 个。随着组件数量和代码量越来越多，主工程的打包时间从最初的十几分钟，增加到了现在的四十分钟左右。依赖组件较多，改动相对频繁的上层业务组件，其发布时间也较为漫长。编译时长的困扰，已经明显影响了日常开发体验，同时也造成 CI pipeline 执行时间过长，在 runner 资源匮乏的情况下，不利于内部 CI 的推广。当前时间节点下，如何减少编译时长，已经成为开发团队较为迫切的需求。

<!--more-->

## 前言

> 组件化除了让模块复用更加便捷，业务开发更加轻量，还有一个不可忽视的优势———组件二进制化，即可通过将非开发中的组件预先编译打包成静态 / 动态库并存放至某处，待集成此组件时，直接使用二进制包，从而提升集成此组件的 App 或者上层组件的编译速度。

对比源码依赖，二进制依赖的组件只需要进行链接而无需编译，可以极大地提升集成效率。掌柜主工程在大部分组件都二进制化的情况下，打包时长从四十分钟左右，下降到最快十二分钟，整整减少了三倍多， CI pipeline 涉及到编译环节的 lint、打包、发布，其耗时也成数倍减少，二进制化所带来的好处不言而喻。

在实践二进制化过程中，由于没有找到较为成熟的依赖切换工具，我们编写了 [cocoapods-bin](https://github.com/tripleCC/cocoapods-bin.git) 通用插件，有需要的开发者可以尝试下。

需要说明的是有些二进制方案是在首次编译后，保留组件生成的二进制包，后续编译直接使用此二进制包。在大多数情况下，比如 App 打包，组件 lint 与发布，这类只进行一次编译的操作，首次编译才是主要关注点。本文所说的二进制化和此类方案的最大区别，就是将组件二进制包制作放到首次编译前，更多的是在组件发布时，同时生成二进制包。

另外，鉴于 CocoaPods 在 1.3.0 后的版本，增加了类似[增量编译的功能](http://blog.cocoapods.org/CocoaPods-1.3.0/)，在首次 install / update 编译之后，后续再进行 install / update 操作，会根据更改结果进行增量编译，个人感觉针对 “非首次 install / update 后的编译“ 优化，并不是必须的，因为 CocoaPods 已经帮我们做好了。

## 二进制化需求

以下是根据掌柜团队日常开发情况，提出的二进制化需求点：

1. 不影响未接入二进制化方案的业务团队
2. 组件级别的源码 / 二进制依赖切换功能
3. 无二进制版本时，自动采用源码版本
4. 接近原生 CocoaPods 的使用体验 （为了满足此需求，我们决定开发自定义的 CocoaPods 插件。）
5. 不增加过多额外的工作量


下面我会参照这几个需求点，逐步说明掌柜 iOS 团队的二进制化过程。

## 宏定义处理
预编译阶段处理的宏定义，在组件进行二进制化后会失效，特别是某些依赖 DEBUG 宏的调试工具，在二进制化之后就不可见了。为了方便处理，我把使用宏的地方分为两种：

- 方法内部
- 方法外部

针对方法内部，我们创建了 TDFMacro 类来替换宏，将逻辑挪到运行时处理：

```objc
// TDFMacro.h
@interface TDFMacro : NSObject
+ (BOOL)enterprise;
+ (BOOL)debug;

+ (void)debugExecute:(void(^)(void))debugExecute elseExecute:(void(^)(void))elseExecute;
+ (void)enterpriseExecute:(void(^)(void))enterpriseExecute elseExecute:(void(^)(void))elseExecute;
@end

// TDFMacro.m
@implementation TDFMacro
+ (BOOL)enterprise {
#if ENTERPRISE
    return YES;
#else
    return NO;
#endif
}

+ (BOOL)debug {
#if DEBUG
    return YES;
#else
    return NO;
#endif
}

+ (void)debugExecute:(void (^)(void))debugExecute elseExecute:(void (^)(void))elseExecute {
    if ([self debug]) {
        !debugExecute ?: debugExecute();
    } else {
        !elseExecute ?: elseExecute();
    }
}

+ (void)enterpriseExecute:(void (^)(void))enterpriseExecute elseExecute:(void (^)(void))elseExecute {
    if ([self enterprise]) {
        !enterpriseExecute ?: enterpriseExecute();
    } else {
        !elseExecute ?: elseExecute();
    }
}
@end
```
这样一来，只需要确保 TDFMacro 组件中的宏有效就可以了————不对其进行二进制化。

针对方法外部，我们尽量将能改写到方法内部的代码改写后按第一种情况处理，不能处理的对代码进行重构以消除宏定义，比如我们网络层的常量，重写前后：

```objc
// 前
#if DEBUG 
NSString * kTDFRootAPI = @"xxx";
#else
NSString * const kTDFRootAPI = @"xxx";
#end
// 后
NSString * kTDFRootAPI = @"xxx";
```

## 制作二进制包

二进制化第一步，先要把组件的二进制包打出来。这里说下比较通用的打包工具 [cocoapods-packager](https://github.com/CocoaPods/cocoapods-packager) 和 [Carthage](https://github.com/Carthage/Carthage/issues) ，目前我们使用  [cocoapods-packager](https://github.com/CocoaPods/cocoapods-packager) 将组件构建 static-framework 。

 [cocoapods-packager](https://github.com/CocoaPods/cocoapods-packager)  的工作原理和 `pod spec/lib lint` 差不多，都是通过 podspec 动态生成 Podfile ，然后 install 出目标工程，最后通过 `xcodebuild` 命令构建出二进制包。这种方式有一个好处，只要保证组件 lint 通过了，就可以打出二进制包，不需要和 Example 工程挂钩，很方便。但是这个插件作者几乎不维护了，很多较久之前的 issue 和 pull request 都是未处理状态。

以下是我们用来构建 static-framework 的命令：

```shell
pod package TDFNavigationBarKit.podspec --exclude-deps --force --no-mangle --spec-sources=http://git.2dfire.net/ios/cocoapods-spec.git
```

在使用过程中，我遇到了两个关于组件资源的问题 ：

- 使用了 `--exclude-deps` option 后，虽然没有把 dependency 的符号信息打进可执行文件，但是它把 dependency 的 bundle 给拷贝过来了 (见 `builder.rb 229 copy_resources` 方法)
- subspec 声明的 resource 不会被拷贝进 framework 中

鉴于 cocoapods-packager  [近期没有发布新版本的计划](https://github.com/CocoaPods/cocoapods-packager/issues/200)，我只能 fork 并更新代码之后，重新发布 [cocoapods-packager-pro](https://github.com/tripleCC/cocoapods-packager) 来修复这两个问题。使用  cocoapods-packager-pro  之后，构建  static-framework 的命令变为：

```shell
pod package-pro TDFNavigationBarKit.podspec --exclude-deps --force --no-mangle --spec-sources=http://git.2dfire.net/ios/cocoapods-spec.git
```

二级命令 package 改成 package-pro 即可。

CocoaPods 目前发布了 1.6.0 beta 版本，试用之后，发现由于某些类的构造函数参数发生了变更， 导致 cocoapods-packager 现有代码已经无法正常工作了，所以 cocoapods-packager  只适用低于 1.6.0 版本的 CocoaPods，后期如果官方 cocoapods-packager 还是没有更新的话，我们应该会在 cocoapods-packager-pro 中适配新版本 CocoaPods。

cocoapods-packager 作者最近还创建了插件 [cocoapods-generate](https://github.com/square/cocoapods-generate) ，此插件可以直接根据 podspec 生成目标工程，相当于 cocoapods-packager 前半部分功能的增强版。目前这个插件支持 CocoaPods 1.6.0 beta 版本，不想用 cocoapods-packager 的开发者，可以先利用 cocoapods-generate 创建目标工程，然后接管构建二进制包的后续操作，可以选择自己实现打包脚本，也可以选择使用 Carthage。

关于 Carthage 如何打 static-framework ，可以参照 [Build static frameworks to speed up your app’s launch times](https://github.com/Carthage/Carthage#build-static-frameworks-to-speed-up-your-apps-launch-times) 。其中有一步是将需要打包的 scheme 设置为 shared ，这个 scheme 对应 CocoaPods 组件的 develpement pod ，一般来说通过 CocoaPods 模版工程或者 cocoapods-generate 插件生成目标工程的 scheme 都是 shared 的，如果没有 shared ，可参照[让 CocoaPods 组件支持 Carthage 打包](https://triplecc.github.io/2018/04/07/2018-04-07-rang-cocoapodszu-jian-zhi-chi-carthageda-bao/)一文进行设置。

构建出 `.framework` 文件后，需要对其进行压缩，我们使用以下命令将文件压缩成 zip 格式：

```shell
zip --symlinks -r TDFNavigationBarKit.framework.zip TDFNavigationBarKit.framework
```

通过上述两个步骤，我们就得到了组件的二进制 zip 包。

需要注意的是，如果使用 cocoapods-packager 打包，其 `.framework` 中的目录结构如下 ：

```shell
TDFNavigationBarKit.framework/
├── Headers -> Versions/Current/Headers
├── Modules
│   └── module.modulemap
├── Resources -> Versions/Current/Resources
├── TDFNavigationBarKit -> Versions/Current/TDFNavigationBarKit
└── Versions
    ├── A
    │   ├── Headers
    │   │   ├── TDFNavigationBarKit.h
    │   │   ├── UIViewController+BackgroundConfigure.h
    │   │   └── UIViewController+NavigationBarConfigure.h
    │   ├── Resources
    │   │   └── Media.xcassets
    │   │       ├── Contents.json
    │   │       ├── common_nbc_back.imageset
    │   │       │   ├── Contents.json
    │   │       │   └── common_nbc_back.png
    │   │       ├── common_nbc_cancel.imageset
    │   │       │   ├── Contents.json
    │   │       │   └── common_nbc_cancel.png
    │   │       ├── common_nbc_ok.imageset
    │   │       │   ├── Contents.json
    │   │       │   └── common_nbc_ok.png
    │   └── TDFNavigationBarKit
    └── Current -> A
```

可以看到，其中的 `Headers`、`Resources`、`Versions/Current` 都是软链接。podspec 中涉及到文件匹配的字段，如 `source_files`、`public_header_files`、`resources` 等，对软链接是无效的，所以需要设置为文件实际存放的路径：

```ruby
s.source_files = TDFNavigationBarKit.framework/Versions/A/Headers/*.h
s.public_header_files = TDFNavigationBarKit.framework/Versions/A/Headers/*.h

# 或者更全面一点
s.source_files = TDFNavigationBarKit.framework/Versions/A/Headers/*.h, TDFNavigationBarKit.framework/Headers/*.h
s.public_header_files = TDFNavigationBarKit.framework/Versions/A/Headers/*.h, TDFNavigationBarKit.framework/Headers/*.h
```

针对二进制包的制作，我们创建了以下命令供团队内部使用：

```shell
# 将源码打包成二进制，并压缩成 zip 包
pod binary package
```

## 存储二进制包

通常二进制包存放的地址有两种，目前我们使用的是第二种 ( 服务器代码可参照 [binary-server](https://github.com/tripleCC/binary-server)  )：

- 组件所在 git 仓库
- 静态文件服务器

相较于 git 仓库，我认为存放至静态文件服务器的优势如下：

- 接口访问，易于扩展与自动化处理
- 源码和二进制分离，依赖二进制时，只下载二进制包比 clone 仓库快
- 不会增大 git 仓库大小，这点也涉及到源码依赖的下载速度

这里说下为什么我们对组件的下载速度这么敏感。

首先，CocoaPods 针对下载的组件是有缓存的，在第一次下载后，CocoaPods 会将组件存放在 Caches 文件夹中，后续 install 操作会先从 Caches 中查找是否有此组件的缓存，如果没有的话，再执行下载流程（是不是感觉和 SDWebImage 有点像）。但是目前 CocoaPods 在同一台机器上，只能有一个版本的缓存 （ ~/Library/Caches/CocoaPods/Pods 下的 VERSION 记录着当前缓存对应的 CocoaPods 版本 ），也就是说我第一次使用 `pod _1.5.3_ install` 下载了所有组件，再执行 `pod _1.4.1_ install` ， CocoaPods 会把 1.5.3 版本的所有组件缓存清空，然后重新下载 。

由于团队内部只有 5 台 Mac mini 机器，我们只能在机器上同时部署 GitLab CI Runner 和 Jenkins Slaver ，CI 脚本中使用的 CocoaPods 版本可以统一控制成 1.4.0 ( 这里不使用最新的 1.5.3 是由于[这个 bug](https://github.com/CocoaPods/CocoaPods/issues/7617) 会导致 lint 失败)，但是其他业务线打包时使用的 CocoaPods 版本就没法统一了，有 1.5.3 的，有 1.6.0.beta 的，加上各业务线的打包频率还比较高，导致机器频繁地在不同 CocoaPods 版本中切换 。

结合上诉两个原因，我们趋向采用下载速度更快的方案。

针对二进制包的增删查，我们创建了以下命令供团队内部使用：

```shell
# 查看所有二进制版本信息
pod binary list 
# 查找组件二进制版本信息
pod binary search NAME
# 下载二进制 zip 包
pod binary pull NAME VERSION
# 推送二进制 zip 包 
pod binary push [PATH] [-name=组件名] [--version=版本号] [--commit=版本日志]     
```

## 切换依赖方式

二进制化后，整体构建速度变快了，但是不利于开发人员跟踪调试，所以就需要有依赖切换功能。这里所说的依赖切换功能包括整个工程、单个组件的切换，以及二进制版本的使用封装，这也是组件二进制化耗费时间和精力最多的地方。

在整个过程中，我总共尝试了三种方案，分别是单私有源单版本、单私有源双版本以及最终采用的**双私有源单版本**。下面我会简单地说下各方案以及实践中遇到的问题。


### 单私有源单版本

> 在不更改私有源和组件版本的前提下，通过动态变更源码 podspec，达到切换依赖的目的

单私有源单版本是我第一次实践采用的方案，也创建了对应的插件 [cocoapods-tdfire-binary](https://github.com/tripleCC/cocoapods-tdfire-binary) ，这里结合插件的实现过程，聊聊实现这类方案时遇到的坑。

前期调研二进制化方案时，我主要参考了 [iOS CocoaPods组件平滑二进制化解决方案](https://www.jianshu.com/p/5338bc626eaf) 一文，所以整体思路和这篇文章差不多，也是通过环境变量加判断语句实现 podspec 的内容变更（虽说 podspec 支持使用 ruby 语法定制，我还是建议最终以 json 格式发布到私有源上，因为 CocoaPods 内部会将 podspec json 化后再执行一些操作，比如缓存，如果这一动作不幂等，操作结果便是不可预知的，从而破坏 CocoaPods 自身的运行机制）。

这种方案最大的困扰在于切换依赖时，如何规避组件缓存带来的负面影响，处理不当容易出现工程组件目录为空的情况，以下是我实践过的两种方案：

1. 确保缓存中同时存在源码和二进制的资源及文件（设置 preserve_paths）

2. 切换依赖前，删除目标组件缓存以及本地 Pods 下的组件目录

在使用二进制服务器的前提下，方案一的常见实现方式为，在 pre_command 中设置下载二进制包脚本，并设置 preserve_paths ，让 CocoaPods 同时保留两种依赖方式所需要的文件即可。考虑到组件本身有二进制版本，组件 Cache 还没有下载的情况，这种方案通常辅以方案二。由于需要同时下载两种依赖的资源，个人并不是很喜欢这种方案，这也是我们弃用  cocoapods-tdfire-binary 的主要原因。

方案二需要 hook  Pod::Installer 类的 resolve_dependencies 方法，在这个方法中清除缓存及本地资源，并且设置组件的沙盒变动标记，这样 CocoaPods 就会重新下载对应的组件了：

```ruby
def cache_descriptors
  @cache_descriptors ||= begin
    cache = Downloader::Cache.new(Config.instance.cache_root + 'Pods')
    cache_descriptors = cache.cache_descriptors_per_pod
  end
end

def clean_local_cache(spec)
  pod_dir = Config.instance.sandbox.pod_dir(spec.root.name)
  framework_file = pod_dir + "#{spec.root.name}.framework"
  if pod_dir.exist? && !framework_file.exist?
    # 设置沙盒变动标记，去 cache 中拿
    # 只有 :changed 、:added 两种状态才会重新去 cache 中拿
    @analysis_result.sandbox_state.add_name(spec.name, :changed)
    begin
      FileUtils.rm_rf(pod_dir)
    rescue => err
      puts err
    end
  end
end

def clean_pod_cache(spec)
  descriptors = cache_descriptors[spec.root.name]
  return if descriptors.nil?
  descriptors = descriptors.select { |d| d[:version] == spec.version}
  descriptors.each do |d|
    # pod cache 文件名由文件内容的 sha1 组成，由于生成时使用的是 podspec，获取时使用的是 podspec.json 导致生成的目录名不一致
    # Downloader::Request slug
    # cache_descriptors_per_pod 表明，specs_dir 中都是以 .json 形式保存 spec
    slug = d[:slug].dirname + "#{spec.version}-#{spec.checksum[0, 5]}"
    framework_file = slug + "#{spec.root.name}.framework"
    unless (framework_file.exist?)
      begin
        FileUtils.rm(d[:spec_file])
        FileUtils.rm_rf(slug)
      rescue => err
        puts err
      end
    end
  end
end
```

需要注意的是，CocoaPods 在 podspec 不是 json 格式时，[缓存目录是有问题的](https://github.com/CocoaPods/CocoaPods/issues/8422)，所以需要我们自己去拼装缓存路径后再执行删除动作。

使用 cocoapods-tdfire-binary 时，我们需要在 podspec 文件中添加以下代码：

```ruby
....
tdfire_source_configurator = lambda do |s|
  # 源码依赖配置
  s.source_files = '${POD_NAME}/Classes/**/*'
  s.public_header_files = '${POD_NAME}/Classes/**/*.{h}'
end
unless %w[tdfire_set_binary_download_configurations tdfire_source tdfire_binary].reduce(true) { |r, m| s.respond_to?(m) & r }
  tdfire_source_configurator.call s
else
  # 内部生成源码依赖配置
  s.tdfire_source tdfire_source_configurator
  # 内部生成二进制依赖配置
  s.tdfire_binary tdfire_source_configurator
  # 设置下载脚本，preseve_paths
  s.tdfire_set_binary_download_configurations
end
```

然后在 Podfile 使用以下语句切换依赖：

```ruby
...
plugin 'cocoapods-tdfire-binary'

tdfire_use_binary!

# tdfire_third_party_use_binary!
tdfire_use_source_pods ['AFNetworking']
...
```

由于编写此插件时，我对 CocoaPods 源码以及 ruby 并不熟悉，导致我没有把 podspec 的配置放到插件内部，现在回过头看，更加合理的做法应该是在 podspec 中设置依赖标志，然后在 hook 的 resolve_dependencies  方法中，变更 podspec 的 source 及依赖相关的字段，这样的话，只需要采用上诉的方案二即可。

可以看到，单私有源单版本对 CocoaPods 缓存策略的侵入还是比较大的。

这里顺便说下 cocoapods-tdfire-binary 是如何处理 subspec 的，首先要说明的是，对于存在 subspec 的组件，我们将其整体打为一个二进制包，并没有分 subspec 构建。假设有组件 A ，B，他们对应的部分 podspec 如下：

```ruby
# A
Pod::Spec.new do |s|
  s.name             = 'A'
  ...
  s.subspec 'Core' do |ss|
    ss.source_files = 'A/Classes/A.{h,m}'
  end
  s.subspec 'Model' do |ss|
    ss.dependency 'A/Core'
    ss.dependency 'YYModel'
    ss.source_files = 'A/Classes/Next.{h,m}'
  end
  s.subspec 'Image' do |ss|
    ss.dependency 'A/Core'
    ss.dependency 'SDWebImage'
    ss.source_files = 'A/Classes/Prev.{h,m}'
  end
  ...
end

# B
Pod::Spec.new do |s|
  s.name             = 'B'
  ...
  s.dependency 'A/Model'
  ...
end
```

当 B 为源码版本，A 为二进制版本时，A 的 subspec 必须要包含 Model ，也就是说 A 的二进制 podspec 必须保证源码 podspec 中的 subspec 都存在，这样切换依赖时才不会出错。 cocoapods-tdfire-binary 在组件 A 为二进制版本时，会动态创建一个名为 TdfireBinary 的 default subspec ，然后将源码 subspec 的依赖上移至 TdfireBinary ：

```ruby
# A
Pod::Spec.new do |s|
  s.name             = 'A'
  ...
  s.subspec 'TdfireBinary' do |ss|
	ss.vendored_frameworks = "A.framework"
    ss.source_files = "A.framework/Headers/*", "A.framework/Versions/A/Headers/*"
    ss.public_header_files = "A.framework/Headers/*", "A.framework/Versions/A/Headers/*"
    ss.dependency 'YYModel'    
    ss.dependency 'SDWebImage'
  end
  s.subspec 'Core' do |ss|
    ss.dependency 'A/TdfireBinary'
  end
  s.subspec 'Model' do |ss|
    ss.dependency 'A/TdfireBinary'
  end
  s.subspec 'Image' do |ss|
	ss.dependency 'A/TdfireBinary'
  end
  ...
end
```

以下是我们实现过程中遇到的部分问题：

- 二进制版本时，依赖 subspec 会引入整个组件
- 需要拷贝 subspec 的属性至 TdfireBinary ，实现起来比较繁琐
- 由于是在插件内部对 podspec 进行转化，扩展性比较差

基于以上问题，我们后续创建 cocoapods-bin 插件时，就把这部分工作交给使用者处理了，如果组件拥有 subspec，那么就需要使用者提供一个模版二进制 podspec ，插件只负责同步 source 和 version。

### 单私有源双版本

>  在不更改私有源的前提下，通过变更组件版本（版本号加 `-binary`），达到切换依赖的目的

由于单私有源单版本要么需要同时下载两种版本的资源，要么切换依赖时需要重新下载目标版本的资源，我们决定以组件缓存为切入点，按照 CocoaPods 的设计规则，将二进制版本和源码版本从物理上区分开来。

最初我想到的就是使用双版本，在源码版本号后添加 `-binary`，即预发布版本，作为二进制版本的版本号。接下来只要在 CocoaPods 使用源码 podspec 下载资源前，将其替换为二进制 podspec 就可以实现二进制版本的切换了。

首先，我们来看下 Pod::Resolver 类，这个类会给 target 创建最终可用的 specifications ，只不过依赖分析工作并不在  Pod::Resolver 中进行，它扮演了类似 DataSource 的角色，将需要分析的数据提供给 Molinillo::Resolver 类处理。

这里说下我尝试从依赖分析切入时遇到的问题。要成为 Molinillo::Resolver 的数据源，需要实现/覆盖 Molinillo::SpecificationProvider 模块中的方法，以下是 Pod::Resolver 实现的 search_for ：

```ruby
def search_for(dependency)
  @search ||= {}
  @search[dependency] ||= begin
    locked_requirement = requirement_for_locked_pod_named(dependency.name)
    additional_requirements = Array(locked_requirement)
    specifications_for_dependency(dependency, additional_requirements)
  end
  @search[dependency].dup
end
def specifications_for_dependency(dependency, additional_requirements = [])
  requirement = Requirement.new(dependency.requirement.as_list + additional_requirements.flat_map(&:as_list))
  find_cached_set(dependency).
    all_specifications(installation_options.warn_for_multiple_pod_sources).
    select { |s| requirement.satisfied_by? s.version }.
    map { |s| s.subspec_by_name(dependency.name, false, true) }.
    compact
end
```

当时我通过 hook  specifications_for_dependency 方法，更改了 requirement ，以使方法返回我想要的 specification，最终也实现了替换 specification 的目的。但是在执行  lint， push 等操作时，由于 Podfile 为内部自动生成，很多组件都是间接依赖的，在目标组件的 podspec 中并没有声明版本，比如间接依赖了 YYModel ，requirement 为 ` ~> 1.0` ，如果替换 requirement 为 `= 1.0.1-binary` 就会出现以下错误：

```
Due to the previous naïve CocoaPods resolver, you were using a pre-release version of `YYModel`, 
without explicitly asking for a pre-release version, which now leads to a conflict. 
Please decide to either use that pre-release version by
adding the version requirement to your Podfile (e.g. `pod 'YYModel', '= 1.0.1-binary, ~> 1.0'`) or
revert to a stable version by running `pod update YYModel`
```

当然，我们也不可能会在 podspec 中显式依赖一个预发布版本，所以这条路最终失败了。实际上我们并不需要关心依赖是如何分析的，只需要等依赖分析完，将最终生成的 specification 替换掉即可，让我们看下 Pod::Resolver 的 resolve 方法：

```ruby
def resolve
  dependencies = @podfile_dependency_cache.target_definition_list.flat_map do |target|
    @podfile_dependency_cache.target_definition_dependencies(target).each do |dep|
      next unless target.platform
      @platforms_by_dependency[dep].push(target.platform)
    end
  end
  @platforms_by_dependency.each_value(&:uniq!)
  @activated = Molinillo::Resolver.new(self, self).resolve(dependencies, locked_dependencies)
  resolver_specs_by_target
rescue Molinillo::ResolverError => e
  handle_resolver_error(e)
end

def resolver_specs_by_target
  @resolver_specs_by_target ||= {}.tap do |resolver_specs_by_target|
    dependencies = {}
    @podfile_dependency_cache.target_definition_list.each do |target|
      specs = @podfile_dependency_cache.target_definition_dependencies(target).flat_map do |dep|
        name = dep.name
        node = @activated.vertex_named(name)
        (valid_dependencies_for_target_from_node(target, dependencies, node) << node).map { |s| [s, node.payload.test_specification?] }
      end

      resolver_specs_by_target[target] = specs.
        group_by(&:first).
        map do |vertex, spec_test_only_tuples|
          test_only = spec_test_only_tuples.all? { |tuple| tuple[1] }
          payload = vertex.payload
          spec_source = payload.respond_to?(:spec_source) && payload.spec_source
          ResolverSpecification.new(payload, test_only, spec_source)
        end.
        sort_by(&:name)
    end
  end
end
```

上面的 resolver_specs_by_target 方法返回就是最终结果，我们只需要变更其返回值就可以了。为了不污染源码私有源以及能更好地维护源码和二进制 podspec ，我们最终没有采用单私有源双版本，而是采用了双私有源单版本，不过两者的实现思路和入口差不多是一致的，这次尝试也给后续的实践铺了路。

### 双私有源单版本

> 在不更改组件版本的前提下，通过变更组件的私有源，达到切换依赖的目的

双私有源分别为源码私有源和二进制私有源，这两个私有源中有相同版本组件，只是 podspec 中的 source 和依赖等字段不一样，所以切换了组件对应的私有源即切换了组件的依赖方式。

以 YYModel 为例，现有源码私有源 cocoapods-spec 及 二进制私有源 cocoapods-spec-binary ，它们都有 YYModel 组件 1.0.4.2 版本的 podspec 如下：

```json
# cocoapods-spec 
{
  "name": "YYModel",
  "summary": "High performance model framework for iOS/OSX.",
  "version": "1.0.4.2",
  "license": {
    "type": "MIT",
    "file": "LICENSE"
  },
  "authors": {
    "ibireme": "ibireme@gmail.com"
  },
  "social_media_url": "http://blog.ibireme.com",
  "homepage": "https://github.com/ibireme/YYModel",
  "platforms": {
    "ios": "6.0",
    "osx": "10.7",
    "watchos": "2.0",
    "tvos": "9.0"
  },
  "source": {
    "git": "git@git.2dfire.net:cocoapods-repos/YYModel.git",
    "tag": "1.0.4.2"
  },
  "frameworks": [
    "Foundation",
    "CoreFoundation"
  ],
  "requires_arc": true,
  "source_files": "YYModel/*.{h,m}",
  "public_header_files": "YYModel/*.{h}"
}

# cocoapods-spec-binary 
{
  "name": "YYModel",
  "summary": "High performance model framework for iOS/OSX.",
  "version": "1.0.4.2",
  "authors": {
    "ibireme": "ibireme@gmail.com"
  },
  "social_media_url": "http://blog.ibireme.com",
  "homepage": "https://github.com/ibireme/YYModel",
  "platforms": {
    "ios": "6.0"
  },
  "source": {
    "http": "http://iosframeworkserver-shopkeeperclient.app.2dfire.com/download/YYModel/1.0.4.2.zip"
  },
  "frameworks": [
    "Foundation",
    "CoreFoundation"
  ],
  "requires_arc": true,
  "source_files": [
    "YYModel.framework/Headers/*",
    "YYModel.framework/Versions/A/Headers/*"
  ],
  "public_header_files": [
    "YYModel.framework/Headers/*",
    "YYModel.framework/Versions/A/Headers/*"
  ],
  "vendored_frameworks": "YYModel.framework"
}

```

当采用 YYModel 的源码版本时，我们从 cocoapods-spec 私有源获取组件的 podspec，那么下载地址为 ` git@git.2dfire.net:cocoapods-repos/YYModel.git`  的  `1.0.4.2`  tag ；当采用 YYModel 的二进制版本时，我们从 cocoapods-spec-binary 私有源获取组件的 podspec，那么下载地址为`http://iosframeworkserver-shopkeeperclient.app.2dfire.com/download/YYModel/1.0.4.2.zip`。

通过上个方案，我们可以知道 resolver_specs_by_target 方法创建了最终使用的 specifications ，接下来我们结合 cocoapods-bin 插件代码，看下如何切换组件的私有源：

```ruby
module Pod
  class Resolver
    # >= 1.4.0 才有 resolver_specs_by_target 以及 ResolverSpecification
    # >= 1.5.0 ResolverSpecification 才有 source，供 install 或者其他操作时，输入 source 变更
    # 
    if Pod.match_version?('~> 1.4') 
      old_resolver_specs_by_target = instance_method(:resolver_specs_by_target)
      define_method(:resolver_specs_by_target) do 
        specs_by_target = old_resolver_specs_by_target.bind(self).call()

        sources_manager = Config.instance.sources_manager
        use_source_pods = podfile.use_source_pods

        missing_binary_specs = []
        specs_by_target.each do |target, rspecs|
          # use_binaries 并且 use_source_pods 不包含
          use_binary_rspecs = if podfile.use_binaries? || podfile.use_binaries_selector
                                rspecs.select do |rspec| 
                                  ([rspec.name, rspec.root.name] & use_source_pods).empty? &&
                                  (podfile.use_binaries_selector.nil? || podfile.use_binaries_selector.call(rspec.spec))
                                end
                              else
                                []
                              end
          specs_by_target[target] = rspecs.map do |rspec|
            # developments 组件采用默认输入的 spec (development pods 的 source 为 nil)
            next rspec unless rspec.spec.respond_to?(:spec_source) && rspec.spec.spec_source

            # 采用二进制依赖并且不为开发组件 
            use_binary = use_binary_rspecs.include?(rspec)
            source = use_binary ? sources_manager.binary_source : sources_manager.code_source 

            spec_version = rspec.spec.version
            begin
              # 从新 source 中获取 spec
              specification = source.specification(rspec.root.name, spec_version)   

              # 组件是 subspec
              specification = specification.subspec_by_name(rspec.name, false, true) if rspec.spec.subspec?
              # 这里可能出现分析依赖的 source 和切换后的 source 对应 specification 的 subspec 对应不上
              # 造成 subspec_by_name 返回 nil，这个是正常现象
              next unless specification

              # 组装新的 rspec ，替换原 rspec
              rspec = if Pod.match_version?('~> 1.4.0')
                        ResolverSpecification.new(specification, rspec.used_by_tests_only)
                      else
                        ResolverSpecification.new(specification, rspec.used_by_tests_only, source)
                      end
              rspec
            rescue Pod::StandardError => error  
              # 没有从新的 source 找到对应版本组件，直接返回原 rspec
              missing_binary_specs << rspec.spec if use_binary
              rspec
            end

            rspec 
          end.compact
        end

        missing_binary_specs.uniq.each do |spec| 
          UI.message "【#{spec.name} | #{spec.version}】组件无对应二进制版本 , 将采用源码依赖." 
        end if missing_binary_specs.any?

        specs_by_target
      end
    end
  end

  if Pod.match_version?('~> 1.4.0')
    # 1.4.0 没有 spec_source
    class Specification
      class Set
        class LazySpecification < BasicObject
          attr_reader :spec_source

          old_initialize = instance_method(:initialize)
          define_method(:initialize) do |name, version, source|
            old_initialize.bind(self).call(name, version, source)

            @spec_source = source 
          end

          def respond_to?(method, include_all = false) 
            return super unless method == :spec_source
            true
          end
        end
      end
    end
  end
end
```

上面就是切换私有源的代码逻辑，可以看到还是比较简短的，这里只单独说三点：

- 我们默认 Development Pods 中的组件为未发布组件，没有二进制版本，所以始终采用原版本
- 因为无法直接从 source 中获取组件的 subspec ，所以这里统一获取 root spec ，如果目标 spec 是 subspec 再从 root spec 中获取 subspec
- 其他业务线的组件可能没有二进制化版本，这里我们如果没有找到组件目标版本的 spec ，会让组件采用原版本，这样就不会因为某个组件版本的缺失而导致 install 失败。

存在两个私有源意味着会有两个不同的 podspec ，分别为源码 podspec 和二进制 podspec ，手动同步这两个 podspec 将会是一个很耗费精力的事情，这时候就需要 cocoapods-bin 插件的辅助命令了。针对没有 subspec 的组件，cocoapods-bin 会根据源码 podspec 自动生成对应的二进制 podspec ；针对有 subspec 的组件，cocoapods-bin 会根据使用者提供的 template podspec 和源码 podspec 自动生成对应的二进制 podspec 。由于源码 podspec 和二进制 podspec 的 diff 是可预见的，我们就可以通过这种半自动的方式避免同时维护两套 podspec 。

更多使用信息可以查看 cocoapods-bin 的 [README](https://github.com/tripleCC/cocoapods-bin) ，这里就不赘述了。

## 整合 CI

从上文可以看出，二进制化还是增加了重复性工作，包括制作二进制包、发布二进制版本等，如果不辅以自动化工具，无疑会增加组件维护者的工作。

在[火掌柜 iOS 团队 GitLab CI 集成实践](https://triplecc.github.io/2018/06/23/2018-06-23-ji-gitlabcide-ci-shi-jian/)的基础上，我们对 CI 配置文件做了些调整：

```yaml
variables:
  # 二进制优先
  BINARY_FIRST: 1 
  # 不允许通知 
  DISABLE_NOTIFY: 0

before_script:
  # https://gitlab.com/gitlab-org/gitlab-ce/issues/14983
  # shared runner 会出现，special runner只会报warning
  - export LANG=en_US.UTF-8
  - export LANGUAGE=en_US:en
  - export LC_ALL=en_US.UTF-8
  
  - pwd
  - git clone git@git.2dfire.net:ios/ci-yaml-shell.git 
  - ci-yaml-shell/before_shell_executor.sh

after_script:
  - rm -fr ci-yaml-shell

stages:
  - check
  - lint
  - test
  - package
  - publish
  - report
  - cleanup

component_check:
  stage: check
  script: 
    - ci-yaml-shell/component_check_executor.rb
  only:
    - master
    - /^release.*$/
    - /^hotfix.*$/
    - tags
    - CI
  tags:
    - iOSCI
  environment:
    name: qa
...

package_framework:
  stage: package 
  only:
    - tags
  script:
    - ci-yaml-shell/framework_pack_executor.sh
  tags:
    - iOSCD
  environment:
    name: production

publish_code_pod:
  stage: publish
  only:
    - tags
  retry: 0
  script:
    - ci-yaml-shell/publish_code_pod.sh
  tags:
    - iOSCD
  environment:
    name: production

publish_binary_pod:
  stage: publish
  only:
    - tags
  retry: 0
  script:
    - ci-yaml-shell/publish_binary_pod.sh
  tags:
    - iOSCD
  environment:
    name: production

report_to_director:
  stage: report
  script:
    - ci-yaml-shell/report_executor.sh
  only:
    - master
    - tags
  when: on_failure
  tags:
    - iOSCD
```

推送 tag 后，如果一切顺利，可以看到 pipeline 执行结果如下：

![pipeline 结果](/images/Snip20190122_1.png)

其中的 package 、 publish 这两个 stage 囊括了二进制化资源制作的主要工作，组件维护者依然可以像二进制化前一样，关注源码版本的发布流程即可。

这里需要注意的是，由于 CocoaPods push 的 Validator 和 lint 基本一致，上文提到的[这个 bug](https://github.com/CocoaPods/CocoaPods/issues/7617) ，对 publish stage 也会有影响，需要暂时指定 CocoaPods 为 1.4.0 版本（`pod _1.4.0_ bin repo push`）。

## 总结

整个组件二进制化的尝试与实践，耗费了我大半年的主要精力，并且我们还需要多维护一个二进制文件服务器，以及对应的二进制版本，在组件 / 代码不多时，做这件事情费时费力，还收效甚微，因此我并不建议还未进行**业务组件化**并且没有上 CI 的团队去做这件事情。

结合我们团队目前的业务性质以及业务组件化进程，在团队实施了组件二进制化之后，团队内部工程编译速度的提升还是显而易见的，并且受益于编译时间的减少，组件自动发布平台的发布时间也大大减少，所以对于我们来说，花时间去做这件事情还是值得的。

## 参考

[iOS CocoaPods组件平滑二进制化解决方案](https://www.jianshu.com/p/5338bc626eaf)

[iOS CocoaPods组件平滑二进制化解决方案及详细教程二之subspecs篇](https://www.jianshu.com/p/85c97dc9ab83)

[组件化-二进制方案](https://www.valiantcat.cn/index.php/2017/05/16/48.html)

