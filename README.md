# 如何创建更加灵活的App

移动应用的部署方式，即发布->下载->安装->运行，决定了他不具备Web的高灵活性。这是多数互联网公司在移动应用开发的过程中所遇到的一个限制。如何打破这一瓶颈，让程序更灵活，让开发更敏捷，让迭代更迅速，是提高开发流程整体效率的一条捷径。

HTML5的发展给开发人员提供了一种可能，PhoneGap就是目前比较成熟的结合原生与HTML的一种方案。其带来的优势不仅仅在于跨平台，同时可以将业务逻辑部署在服务器上，不需要进行客户端的升级即可灵活的进行调整。

但是就目前看，HTML5要全面达到原生应用的用户体验还有很长的路要走。并且与原生结合的开发方式虽能够取长补短，但同时也会对程序的复杂度、可维护性及开发人员的技能产生更高的要求。

本文分别介绍了Android和iOS上的两种基于原生应用开发的灵活性解决方案，可以在不改变开发人员技能要求和已有的开发方式的前提下，给于应用在发布后改变程序行为的能力。相关的代码及示例已共享为开源项目：<http://github.com/mmin18>。

# Android

所有的Android应用都被封装在APK包中，一个APK包一般由三部分组成：

* AndroidManifest.xml描述了APK包的固有属性，是不可变部分。
* classes.dex包含了Java类（.class）被编译打包成了Dalvik虚拟机的中间代码。
* res/文件夹下的包含了所有可访问的资源文件。

一个Android应用在启动时，首先Dalvik加载的是Android自身的框架。之后会加载APK包中的classes.dex文件到全局的ClassLoader。最后根据AndroidManifest.xml中指定的类名，创建对应的Activity实例来展示UI。

Android通过dalvik.system.DexClassLoader提供了动态加载Java代码的能力，如果我们能够在Activity启动之前，替换全局的ClassLoader（Application.mBase.mPackageInfo.mClassLoader），那么我们就可以改变载入的Activity类对应的实现。

### ActivityLoader

ActivityLoader和ActivityDeveloper项目（<https://github.com/mmin18/AndroidDynamicLoader>）展示了利用Java反射机制替换全局ClassLoader到我们自定义的dalvik.system.DexClassLoader，从而在程序运行过程中改变Activity的具体实现。

ActivityDeveloper为开发环境，开发完成后，将生成的APK文件（不需要签名）放入ActivityLoader的资源文件中，并重新启动ActivityLoader来装载新的代码，就可以使得ActivityDeveloper中的代码在ActivityLoader中运行。

![ActivityLoader](https://raw.github.com/mmin18/Create-a-More-Flexible-App/master/ActivityLoader.png)

但是这种方式也有局限性，首先在ActivityDeveloper中开发的Activity必须和ActivityLoader.AndroidManifest.xml中指定的类名相同，并且由于AndroidManifest.xml是APK包的固有属性，无法在运行时改变，所以无法动态的增加Activity。另外我们修改的变量并不在Android的文档中提及，所以无法保证在Google升级Android系统后该方法已然有效。

### FragmentLoader

每一个Activity的上下文都包含了自身的ClassLoader和资源管理，通过重载以下四个函数就可以替换Activity所载入的资源和代码的路径指向：

* ClassLoader getClassLoader()
* AssetManager getAssets()
* Resources getResources()
* Theme getTheme()

在Android 3.0引入Fragment后，我们可以在一个应用程序中只实现一个Activity作为入口和运行的容器，在启动的Intent参数中指定Activity需要动态加载的Fragment类名、classes.dex文件和资源文件路径，并在启动完成后把所有的界面响应、生命周期管理和状态可持久化等均交由Fragment处理。（Google在Android 3.0后引入了Fragment，Fragment既有Activity的生命周期管理和状态可持久化，又可以像View那样自由的组合。）

FragmentLoader和FragmentDeveloper项目（<https://github.com/mmin18/AndroidDynamicLoader>）实现了利用Fragment来负责UI和所有的代码逻辑，Activity作为唯一的启动入口，负责从APK加载Fragment及相关的资源。其项目开发在FragmentDeveloper中进行，FragmentLoader类似模拟器，从外部加载APK并运行。

![FragmentLoader](https://raw.github.com/mmin18/Create-a-More-Flexible-App/master/FragmentLoader.png)

利用这一开发方式，我们可以在应用部署后完全通过动态加载的方式来更新客户端运行的代码和资源文件。该方式的限制主要是无法在运行时变更AndroidManifest.xml的定义，如程序的权限，版本等。

# iOS

虽然Objective-C的运行时支持新增类型和方法，但是由于苹果的限制，开发者无法在iOS上动态加载Objective-C原生代码，所以只能寻求替代方案。

脚本语言就可以一定程度上解决这一问题，Wax项目（<http://github.com/probablycorey/wax>）实现了在运行时使用Lua创建Objective-C类与方法的能力，使得用Lua语言来开发iOS应用成为一种可能。

当然多数熟悉iOS的开发人员不一定会赞同采用这种开发方式，所以我们在引入Wax的同时希望能够和已有的iOS项目无缝的结合起来，不改变项目使用Objective-C开发的方式，在项目上线后通过Wax来改变程序运行时的行为。这样就可以避免漫长的AppStore上线审核，随时对线上程序进行调整，甚至是修复线上程序的问题或缺陷。

WaxPatch项目（<http://github.com/mmin18/WaxPatch>）展示了Lua补丁的功能，在程序启动时会从指定地址下载一个包含所有Lua补丁的zip包，通过Wax加载后改变了既有Objective-C实现方法的指向函数，从而改变了程序的行为。

下图是使用Objective-C开发的原版程序：

![原版](https://raw.github.com/mmin18/Create-a-More-Flexible-App/master/WaxOriginal.png)

其展示了一个UITableViewController，主要代码如下：

    @implementation MainViewController
    
    - (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
        return 10;
    }
    
    - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
        UITableViewCell *cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:@"Cell"];
        cell.textLabel.text = [NSString stringWithFormat:@"%d", indexPath.row + 1];
        return cell;
    }
    
    @end

在加载了Lua补丁后，程序的运行结果如下：

![补丁](https://raw.github.com/mmin18/Create-a-More-Flexible-App/master/WaxPatched.png)

Lua补丁覆盖了原程序的 tableView:cellForRowAtIndexPath: 方法，Lua脚本如下：

    waxClass{"MainViewController", UITableViewController}
    
    function tableView_cellForRowAtIndexPath(self, tableView, indexPath)
    	local cell = self:ORIGtableView_cellForRowAtIndexPath(tableView, indexPath) -- 被覆盖的方法会在名字前加上'ORIG'继续保留
    	cell:textLabel():setText("" .. (10 - indexPath:row()))
    	cell:detailTextLabel():setText("http://github.com/mmin18")
    	cell:textLabel():setTextColor(UIColor:redColor())
    	return cell
    end

可见Lua脚本的编写方式和Objective-C非常相似，因为所有的方法和类名都和iOS的原生框架保持一致。一个iOS开发人员只需要略微了解一下Lua的语法，就可以编写对应的Lua补丁。更多的细节可以参见Wax项目的介绍：<https://github.com/probablycorey/wax/wiki/Overview>。

需要注意的是Wax项目本身不支持方法覆盖和从Objective-C反向调用Lua修改后的实例，WaxPatch项目对Wax框架做了一定的修改。具体的实现请参见开源项目。

# 总结

Android框架本身就是一个插件系统，FragmentLoader项目利用了这一特性，采用极少的代码就实现了较完善的动态加载特性，几乎所有的业务逻辑代码和界面所需的资源文件都可以实现动态加载。

在iOS下使用WaxPatch的目的是引入灵活性，在已有程序的基础上修改部分代码逻辑，避免漫长的AppStore上线审核，这在程序需要紧急修复线上缺陷的情况下非常管用。

移动互联网的发展非常迅速，Web开发中的成功模式，如最小化原型，快速验证，灰度发布等也会越来越多的被移动应用开发所采用。这都需要程序灵活多变的特性来进行保证，希望以上介绍的项目能给应用开发者一点启发。
