---
title: Shell 脚本加密的一些思考
slug: some-thoughts-on-shell-script-encryption
tags: [Linux, Shell]
date: 2022-11-02T22:16:54+08:00
---

今天同事请教我一个问题，说不想让别人看到脚本执行的内容，我一开始脸懵，后来才了解到，她是不想让用户看到脚本的内容（卸载软件的路径），以防软件的文件被人恶意破坏，导致不能正常运行等问题，于是便有了本文。<!--more-->

假设需要加密的脚本长这样：

```shell
#!/bin/bash
echo "开始卸载su"
rm -f /system/bin/su
rm -f /system/xbin/su
rm -f /system/bin/.ext/.su
rm -f /system/etc/.installed_su_daemon
rm -f /system/app/Superuser.apk
rm -f /system/app/Superuser.odex
rm -f /system/app/SuperUser.apk
rm -f /system/app/SuperUser.odex
rm -f /system/app/superuser.apk
rm -f /system/app/superuser.odex
rm -f /system/app/Supersu.apk
rm -f /system/app/Supersu.odex
rm -f /system/app/SuperSU.apk
rm -f /system/app/SuperSU.odex
rm -f /system/app/supersu.apk
rm -f /system/app/supersu.odex
echo "卸载su完成"
```

加密，联想到之前群里见过的挖矿脚本，黑客在登录别人机器后，会进行一些常规的操作，但是不希望机主登录时发现，就会把脚本进行加密，或者从互联网的某一处匿名的地址去下载一段加密的脚本。

那么到底是怎么加密的？

```shell
IyEvYmluL2Jhc2gKZWNobyAi5byA5aeL5Y246L29c3UiCnJtIC1mIC9zeXN0ZW0vYmluL3N1CnJtIC1mIC9zeXN0ZW0veGJpbi9zdQpybSAtZiAvc3lzdGVtL2Jpbi8uZXh0Ly5zdQpybSAtZiAvc3lzdGVtL2V0Yy8uaW5zdGFsbGVkX3N1X2RhZW1vbgpybSAtZiAvc3lzdGVtL2FwcC9TdXBlcnVzZXIuYXBrCnJtIC1mIC9zeXN0ZW0vYXBwL1N1cGVydXNlci5vZGV4CnJtIC1mIC9zeXN0ZW0vYXBwL1N1cGVyVXNlci5hcGsKcm0gLWYgL3N5c3RlbS9hcHAvU3VwZXJVc2VyLm9kZXgKcm0gLWYgL3N5c3RlbS9hcHAvc3VwZXJ1c2VyLmFwawpybSAtZiAvc3lzdGVtL2FwcC9zdXBlcnVzZXIub2RleApybSAtZiAvc3lzdGVtL2FwcC9TdXBlcnN1LmFwawpybSAtZiAvc3lzdGVtL2FwcC9TdXBlcnN1Lm9kZXgKcm0gLWYgL3N5c3RlbS9hcHAvU3VwZXJTVS5hcGsKcm0gLWYgL3N5c3RlbS9hcHAvU3VwZXJTVS5vZGV4CnJtIC1mIC9zeXN0ZW0vYXBwL3N1cGVyc3UuYXBrCnJtIC1mIC9zeXN0ZW0vYXBwL3N1cGVyc3Uub2RleAplY2hvICLljbjovb1zdeWujOaIkCI=
```

上面的脚本变成了这样，解密之后就是上面的样子了，用到的就是 [Base64](https://zh.wikipedia.org/zh-cn/Base64) 转换。

把脚本转成 Base64，然后放到某台机器上，在需要执行的机器上下载下来，然后执行。就算有好奇宝宝，看看脚本里面是啥，打开啥也看不懂。当然标题的加密确实该打引号了，毕竟但凡他听说过 Base64，就知道怎么解了。

```shell
curl -Os http://10.31.140.88/file/delete_after_run && cat delete_after_run | base64 -d | bash
```

命令的意思是，从远程地址下载，然后通过 `base64 -d` 对其进行解密，然后用 bash 执行。

说完了讲清楚了，`5YaN6KeB5LqG77yM5oSf6LCi6KeC55yL`