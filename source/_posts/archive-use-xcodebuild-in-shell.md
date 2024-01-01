---
title: 在shell上使用xcodebuild打包
date: 2017-04-23 01:23:41
tags:
- 效率
- Shell
categories:
- 学习笔记
---

最近学习了一下shell脚本，对unix很多命令更熟悉了，也学到了很多新命令
简单使用xcodebuild来写了一个打包的脚本，流程是打包，导出ipa，上传到蒲公英 / bugly之类的测试网站，通知 slack

<!-- more -->

### archive
打包，和导出ipa都是使用这个命令，可以使用`man xcodebuild`来查看详细用法
其中打包使用的是`archive`这个命令，一般`archive` 完导出一个`.xcarchive` 类型的文件，这需要我们指定路径
你打包的项目是project还是workspace也要指定，使用`-project <projectName>`或者`-workspace <workspaceName>`
还有一个就是要指定scheme，使用-scheme <schemeName>，这个scheme有个地方要注意的是，如果scheme name有空格则要用转义，即`-scheme test\ dev`这里要小吐槽一下，不建议scheme取有空格名字，可能会有各种转义问题
所以打包命令就是这样,其他选项默认即可

```
xcodebuild archive -archivePath ~/Desktop/tmp.xcarchive -workspace test.xcworkspace -scheme test\ dev
```

### export

打完包则要导出ipa文件，也是使用`xcodebuild`，使用的命令是`-exportArchive`
要指定打包的路径，即刚才打包出来的 `*.xcarchive` 文件，使用`-archivePath ~/Desktop/tmp.xcarchive`

然后就是导出ipa文件的路径

```
-exportPath ~/Desktop/
```

最后就是要指定一些选项，

```-exportOptionsPlist <.plist path>
```
需要指定一个plist文件的路径
例如使用xcode导出的时候会让你选择以什么方法打包

不指定的话一般默认`development`，还有`ad-hoc,app-store,package,enterprise,developer-id`等选项
还有 teamID是要指定的你的开发者账号的team id，
```
xcodebuild -showBuildSettings -workspace test.xcworkspace -scheme testScheme |grep DEVELOPMENT_TEAM
```
使用这个命令可以查看到，也可以上开发者官网上查看
指定的plist文件就是这样

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC \"-//Apple//DTD PLIST 1.0//EN\" \"http://www.apple.com/DTDs/PropertyList-1.0.dtd\">
<plist version="1.0">
<dict>
    <key>teamID</key>
    <string>${teamID}</string>
	<key>method</key>
	<string>${method}</string>
</dict>
</plist>
```

### 上传
这里我上传的是蒲公英，使用bugly也是差不多的
使用的是curl命令ipaPath即指定你的ipa的路径，然后添加你的user key和api key就行了

```
curl -F "file=@$ipaPath" -F "uKey=<your user key>" -F "_api_key=<your api key>" https://qiniu-storage.pgyer.com/apiv1/app/upload
```

### 通知slack
我使用的也是curl命令，最近经常用这个命令，感觉很好用

```
curl -X POST --data-urlencode 'payload={"channel": "@ascen", "username": "iOS", "text": "archive '${scheme}' <https://www.pgyer.com/****|你想写的链接文字>.", "icon_emoji": ":ghost:"}' https://hooks.slack.com/services/T2A8E9XSP/B50F24ZRS/**你的url**
```

所以选择shell script是这样的

```shell
#!/bin/sh
#!/bin/bash

currentPath=$(pwd)

#########configuration############
scheme="testScheme"
teamID="your team id"
method="development"
# project path
cd ~/Desktop/your project path

##################################

xcodebuild clean

xcws=$(echo *.xcworkspace)
xcodebuild archive -archivePath ~/Desktop/tmp.xcarchive -workspace $xcws -scheme $scheme

exportOptions="<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC \"-//Apple//DTD PLIST 1.0//EN\" \"http://www.apple.com/DTDs/PropertyList-1.0.dtd\">
<plist version="1.0">
<dict>
	<key>teamID</key>
	<string>${teamID}</string>
	<key>method</key>
	<string>${method}</string>
</dict>
</plist>"

echo $exportOptions > ~/Desktop/exportOptions.plist

xcodebuild -exportArchive -archivePath ~/Desktop/tmp.xcarchive -exportPath ~/Desktop/ -exportOptionsPlist ~/Desktop/exportOptions.plist

# delete tmp file
rm -rf ~/Desktop/tmp.xcarchive
rm ~/Desktop/exportOptions.plist

ipaPath="/Users/Ascen/Desktop/${scheme}.ipa"
echo $ipaPath

curl -F "file=@$ipaPath" -F "uKey=<your user key>" -F "_api_key=<your api key>" https://qiniu-storage.pgyer.com/apiv1/app/upload

curl -X POST --data-urlencode 'payload={"channel": "@ascen", "username": "iOS", "text": "archive '${scheme}' <https://www.pgyer.com/shundaochuxing|乘客端>.", "icon_emoji": ":ghost:"}' https://your slack url

rm $ipaPath

cd $currentPath
```


