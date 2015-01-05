---
layout: post
title: "[翻译]在iOS8下通过编码设置和管理VPN连接"
categories: iOS 翻译
---
我把示例写到了[iOS的通知中心扩展里](https://github.com/liulunet/TodayVPN)，能够很方便的全局控制VPN开关。但是没弄明白localIdentifier和remoteIdentifier到底是什么，翻墙用的VPN账号没有这个信息，用Mac Server和群晖的VPN Server搭建自己的VPN也没找到服务端的设置，问一个CCNP的朋友说就是一个标识，不设置也可以连接。但我总觉得那个水货欺骗了我，应该没那么简单…

[原文:http://ramezanpour.net/post/2014/08/03/configure-and-manage-vpn-connections-programmatically-in-ios-8/](http://ramezanpour.net/post/2014/08/03/configure-and-manage-vpn-connections-programmatically-in-ios-8/)

**更新3**：如果你要创建一个按需连接（on-demand）的VPN连接，在阅读本文以后看看我写的介绍[在iOS8下创建按需连接的VPN连接](http://ramezanpour.net/post/2014/10/15/create-an-on-demand-vpn-connection-programmatically-in-ios-8/)的文章。

**更新2**：beta5版本的问题已经在GM版本得到修复，现在完美了。

**更新**：这个方案在iOS8beta5版本中会遇到一个奇怪的“Missing name”问题，不过它在iOS8bete1到4版本中都正常。如果你有关于这方面的资讯，请在下方的评论中留言。

由于苹果的限制，通过编码在app中创建VPN连接一直以来都是不可能完成的任务。

在我之前的博文中，有一篇介绍[苹果发布全新的Network Extension framerowk，让开发者能通过编码配置VPN连接](http://ramezanpour.net/post/2014/07/14/introducing-network-extension-in-ios-8-and-os-x-10-10/)的文章，不过我还没有介绍如何使用。

官方目前尚未发布任何相关文档，本文可作为在iOS8和OS X(10.10)Yosemite下管理VPN配置的指南。感谢[quellish](http://pimpmyradar.com/)在这方面给我的很多帮助。

##要求

1. 最基本的一点，你得有一台运行iOS8 Beta1以上版本的设备来调试app！这个实验无法在模拟器上运行。如果你要开发的是Mac程序，电脑必须有OS X Yosemite Preview3以上版本的系统。

2. 由于无法在模拟器上进行调试，你必须加入iOS/Mac开发计划。另外你还得对项目的配置文件做一些修改，iOS7版本的配置文件不能用来进行iOS8VPN的程序开发。

3. Xcode6beta。在本文发布的时候（2014年8月2日），Xcode6还在进行beta4版本测试，之前也提到，你需要加入iOS/Mac开发计划来获取这个beta版本的工具。

4. 最后，你还需要一台Mac。Xcode不能在Windows和Linux设备上运行。

##开始

在着手编码之前，你需要先更新你的项目配置文件。如果你之前没有创建过配置文件，那现在就要新建一个！登陆你的[开发账号](http://developer.apple.com/)，然后点击“Certificate, Identifiers & Profiles”：

![update_provision_profile.png](http://ramezanpour.net/wp-content/uploads/2014/08/update_provision_profile.png)

选择Identifiers，然后选择你想要更新配置文件的app。如果你还没有配置文件，可以点击加号新建一个。选择一个app以后，你会看到该app将使用的功能列表。例如需要在app中调用iCloud相关的功能，就必须吧iCloud功能项打开，否则就无法测试和部署基于iCloud的app。下面是列表的截图：

![identifiers_first.png](http://ramezanpour.net/wp-content/uploads/2014/08/identifiers_first.png)

iOS8发布以后，这里新增了**“VPN Configuration & Control”**项目。这正是我们要找的！点击编辑按钮然后选中该项目的复选框来打开它。当你打开该项时，会弹出相关介绍：

![enable_vpn_funcationality.png](http://ramezanpour.net/wp-content/uploads/2014/08/enable_vpn_funcationality.png)

开启VPN Configuration & Control以后点击完成，然后重新下载这个配置文件覆盖安装到电脑上。好了，现在让我们回到Xcode。

**注：本文假定你熟悉iOS开发和Objective-C。如果你从未在iOS或Mac系统下开发过任何app，你得先学习一些基本的概念再来阅读本文。**

打开Xcode，然后新建一个单视图的项目，放一个button到屏幕中间并将它连接到你的ViewController。

我们要在viewDidLoad方法中设置VPN配置，然后在按钮的点击事件中连接指定的VPN服务器。

在开始之前，你得先了解这一切是如何实现的！如果你了解了Network Extension framework的原理，基于它的开发工作将会变得更容易。

##NetworkExtension.framework

苹果在开发这个库时已经有了一个精妙的设计。所有的app都可以在自己的沙盒之内访问系统设置，这意味着你无法访问其他app的沙盒。

首先，已保存的配置信息必须先从操作系统加载之后才能访问。配置信息被载入以后你就能修改它。修改之后必须保存才能生效。如果不再需要配置信息，也可以将其删除。因此，我们要按照以下步骤创建VPN配置信息：

* 载入app的配置信息
* 修改配置信息
* 保存配置信息

**要注意的是，哪怕不修改任何设置，也要先载入app的配置信息。**

VPN连接创建以后，我们可以通过它控制VPN的连接和断开。

Network extension包含了三个主要的类：

* NEVPNManager
* NEVPNProtocol
* NEVPNConnection

`NEVPNManager`是最重要的一个类。它负责载入，保存和删除配置信息。实际上，所有对VPN的行为都需要通过这个类来完成。

##创建一个新的VPN连接

首先得创建一个该类的实例对象：

```objective-c
NEVPNManager *manager = [NEVPNManager sharedManager];
```

初始化`NEVPNManager`后，系统配置就可以通过`loadFromPreferencesWithCompletionHandler:`方法来载入了：

```objective-c
[manager loadFromPreferencesWithCompletionHandler:^(NSError *error) {
    // Put your codes here...
}];
```

看上面的代码，load方法传入了一个block参数。每当载入完成时，这个block就会被执行。这个block还有一个`NSError`参数。如果加载成功，这个`NSError`将会是`nil`，否则就不为空。接下来：

```objective-c
[manager loadFromPreferencesWithCompletionHandler:^(NSError *error) {
    if(error) {
        NSLog(@"Load error: %@", error);
    } else {
        // No errors! The rest of your codes goes here...
    }
}];
```

加载完成以后，就可以设置我们的VPN连接了。

iOS8支持两种主要的协议，IPSec和IKEv2。苹果头一次将IKEv2协议加入到它的操作系统中，该协议支持包括Android，Windows Phone，Windows Desktop，Linux和现在的iOS，Mac在内的主流操作系统。本文中我们将讨论关于IPSec的内容，我之后会另发一篇博文讨论关于IKEv2的内容。除了这些协议之外，如果你需要的话，苹果现在还开放了创建自定义协议的能力！这个特性对于那些已经实现了自定义协议的人们来说非常重要，因为现在可以同时在iOS和Mac中添加自己的协议支持了。好了，现在来设置我们的IPSec协议：

```objective-c
NEVPNProtocolIPSec *p = [[NEVPNProtocolIPSec alloc] init];
p.username = @"[Your username]";
p.passwordReference = [VPN user password from keychain];
p.serverAddress = @"[Your server address]";
p.authenticationMethod = NEVPNIKEAuthenticationMethodSharedSecret;
p.sharedSecretReference = [VPN server shared secret from keychain];
p.localIdentifier = @"[VPN local identifier]";
p.remoteIdentifier = @"[VPN remote identifier]";
p.useExtendedAuthentication = YES;
p.disconnectOnSleep = NO;
```

第一行，我创建了一个`NEVPNProtocolIPSec`实例。这是`NEVPNProtocol`的子类，`NEVPNProtocol`是一个抽象类，你可以用它来创建你自己的协议。

之后，我在第二行和第三行设定了用户名和密码。要注意的是密码必须是一个钥匙串引用对象（a reference from Keychain），所以你需要先将密码保存到钥匙串，然后再从中取出引用对象。

第四行是服务器地址。服务器地址可以是IP、主机名或者URL。

接下来是认证方式。iOS8提供了三种认证方式

* `NEVPNIKEAuthenticationMethodNone`：不对IPSec服务器进行认证
* `NEVPNIKEAuthenticationMethodCertificate`：使用证书和私钥进行认证
* `NEVPNIKEAuthenticationMethodSharedSecret`：使用共享密钥进行认证

如你所见，我选择了使用共享密钥的认证方式，你可以使用你自己需要的认证方式。

下一行是共享密钥引用对象（Shared Secret reference）。它同样是一个钥匙串引用，所以你需要从钥匙串拿到它。如果你要用证书替换共享密钥，就没有必要给`sharedSecretReference`属性赋值了，不过你需要给`identityData`属性赋值。Identity data是VPN证书中的PKCS12数据。这个属性的值必须是PKCS12格式的NSData类型：

```objective-c
p.identityData = [NSData dataWithContentsOfFile:[[NSBundle mainBundle] pathForResource:@"clientCert" ofType:@"p12"]];
```

接下来的两行是本地和远程的标识符。这是进行验证的时候用来标识本地和远程IPSec端点(IPSec endpoints)的两个字符串。

我们需要设置的下一个属性是`useExtendedAuthetication`。这是一个设定是否使用行扩展认证（extended authentication）的标识，该协议通过扩展IKE认证来检查IKE会话中的端点。在IKE1版本中，如果这个标识设置为X-Auth，扩展验证就会作为IKE会话中的一部分，将用户名和密码属性作为凭证。在IKE2版本中，如果这个标识设置成EAP（Extensible Authentication Protocol）认证，扩展验证就会作为IKE会话的一部分，将用户名和密码或者证书作为凭证，这取决于服务器需要哪种EAP方法。

最后要设置的属性是`disconnectOnSleep`。这个布尔值标识在设备进入休眠的时候是否需要断开连接。

设置这些属性就够了。接下来我们要将刚才设定的协议设置到VPN manager中。可以使用`setProtocol:`方法。

```objective-c
[manager setProtocol:p];
```

IPSec和IKEv2协议有一个非常酷的特性叫做按需连接。可以在用户准备访问互联网的时候自动发起连接。在iOS8中，可以将连接开启按需连接功能。但是我准备在另一篇博文中讨论这个；现在先不管这个特性，将`onDemandEnabled`属性设置成NO。

```objective-c
[manager setOnDemandEnabled:p];
```

最后我们必须给要创建的VPN连接取个名字。直接通过setLocalizedDescription:方法设置它的localized description属性就行：

```objective-c
[manager setLocalizedDescription:@"[You VPN configuration name]"];
```

现在这个协议差不多设置好了，但我们还没有保存它。要保存这个配置很简单，调用saveToPreferencesWithCompletionHandler: 方法就行了：

```objective-c
[manager saveToPreferencesWithCompletionHandler:^(NSError *error) {
    if(error) {
        NSLog(@"Save error: %@", error);
    }
    else {
        NSLog(@"Saved!");
    }
}];
```

该方法只会把你已设定的配置项保存到系统中。

##连接刚才创建的VPN连接

配置信息已经保存到系统中，现在可以连接它了。NEVPNManager有一个属性叫`connection`。这个属性是一个`NEVPNConnection`类型的对象。它持有连接VPN所必须的信息。要将刚才创建的VPN连接连接到服务器，直接像下面的代码那样调用`NEVPNConnection`类中的`startVPNTunnelAndReturnError:`方法就可以了：

在你的设备上运行app，你会看到新创建的连接配置，你可以点击屏幕中间的按钮连接它。另外，你还可以调用`NEVPNConnection`的`stopVPNTunnel`方法，通过程序断开VPN连接。

```objective-c
- (IBAction)buttonPressed:(id)sender {
    NSError *startError;
    [[NEVPNManager sharedManager].connection startVPNTunnelAndReturnError:&startError];

    if(startError) {
        NSLog(@"Start error: %@", startError.localizedDescription);
    } else {
        NSLog(@"Connection established!");
    }

}
```

如果你对本文有任何想法和建议，请在下方留言。我近期将会写一篇关于IKEv2和按需连接特性的博文，敬请关注。
