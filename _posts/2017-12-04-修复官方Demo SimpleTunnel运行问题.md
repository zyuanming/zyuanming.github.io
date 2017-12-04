---
layout: post
title: 修复官方Demo SimpleTunnel运行问题
date: 2017-12-04
categories: blog
tags: [iOS]
description: NetworkExtension

---

## 解决方法

On iOS problems like this almost always boil down to one of two things.
Entitlements
You wrote:
I checked the entitlements and NE seems to be properly setup in both the containing App and the app-extension.
Did you check the entitlements of the built binaries or just the .entitlements file?  You must check do the former, because that’s what the system looks at.  The .entitlements file is just one input to the code signing machinery, as I explain in my Debugging Entitlement Issues post.
Info.plist
You should double check that your provider’s Info.plist is set up correctly.  Specifically, check that the NSExtension property is set up like this:

```
<key>NSExtension</key>
<dict>  
    <key>NSExtensionPointIdentifier</key>  
    <string>com.apple.networkextension.packet-tunnel</string>  
    <key>NSExtensionPrincipalClass</key>  
    <string>Module.Provider</string>  
</dict>

```

where:
Module is your Swift module name (if you’re using Objective-C, remove both the Module and the .)
Provider is the name of your NEPacketTunnelProvider subclass
Share and Enjoy

注意，上面的NSExtensionPrincipalClass key对应的内容，应该是：

    $(PRODUCT_NAME:c99extidentifier).PacketTunnelProvider

>>> 这个在有些app中，却不需要这么写，直接 `PacketTunnelProvider ` 指明类名就可以了。还在探讨中...
    
## 参考链接

\[1]: <https://forums.developer.apple.com/thread/77704>