---
title: APK瘦身
date: 2016-04-04 15:16:24
tags: Android
---

### APK瘦身系列

#### 1.[解剖APK](https://medium.com/google-developers/smallerapk-part-1-anatomy-of-an-apk-da83c25e7003?source=user_profile---------8-)

如果我问开发者他们的APP有多大，我能很确定，大部分人会去看下``Android Studio``生成的APK有多大，然后告诉我。这是最直接的答案。考虑以下例子:

* 当你的APP安装在用户的设备上时会占用多大的空间？
* 用户需要花费多少的网络流量来下载和安装您的APP
* 更新APP时，需要下载多少内容？
* 你的APP运行时占用了多少内存？
<!--more-->
##### APK里面究竟有什么？

```
classes.dex
res 
resources.arsc
AndriodManifest.xml
libs
assets
META-INF
```

##### 使用Zopfli来重新压缩APK(5.0.1可能会有crash现象)

使用``zipalign -z 4 input.apk output.apk``或者在gradle里加入  

```
//add zopfli to variants with release build type
android.applicationVariants.all { variant ->
    if (variant.buildType.name == 'release') {
        variant.outputs.each { output ->
            output.assemble.doLast {
                println "Zopflifying... it might take a while"
                exec {
                    commandLine output.zipAlign.zipAlignExe,'-f','-z', '4', output.outputFile.absolutePath , output.outputFile.absolutePath.replaceAll('\\.apk$', '-zopfli.apk')
                }
            }
        }
    }
}
```

#### 2.[简化代码](https://medium.com/google-developers/smallerapk-part-2-minifying-code-554560d2ed40#.g0uo79o4u)
##### 使用proguard简化dex代码
##### 上传ProGuard mappings到play store上
##### 三方库的Proguard配置
三方库  

```
android {
	defaultConfig {
		consumerProguardFiles "proguard-rules.txt"
	}
}
```
 
##### 跟踪你需要的依赖
##### 过渡库的依赖

```
./gradlew app:dependencies
```

查看引入的依赖
##### 使用``ClassyShark``来检查Dex文件  

#### 3.[移除没用的资源文件](https://medium.com/google-developers/smallerapk-part-3-removing-unused-resources-1511f9e3f761#.itzxr72ra)
``shrinkResources true``
由于系统可能会出现某些错误，因此可以使用

```
res/raw/keep.xml(在资源文件目录下)
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
   tools:keep="@layout/l_used*_c,@layout/l_used_a,@layout/l_used_b*"
/>

或者
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:shrinkMode="safe"
    tools:discard="@layout/unused2"
/>
```

##### 使用``ResConfigs``移除无用的配置

```
android {
	defaultConfig {
		resConfigs "en", "fr"
	}
}
```

##### 稀疏resources.arsc中的配置

#### 4.[通过ABI和分辨率区分多种APK](https://medium.com/@wkalicinski/smallerapk-part-4-multi-apk-through-abi-and-density-splits-477083989006#.vhvtofczw)

```
android {
	splits {
		density {
			enable true
			exclude 'ldpi', 'tvdpi', 'xxxhdpi'
			compatibleScreens 'small', 'normal', 'large', 'xlarge'
		}
	}
}

ext.additionalDensities = ['xhdpi': ['280'], 'xxhdpi': ['420', '400', '360'], 'xxxhdpi': ['560']]
import com.android.build.OutputFile

android.applicationVariants.all { variant ->
    // assign different version code for each output
    variant.outputs.each { output ->
        if (output.getFilter(OutputFile.DENSITY) != null && project.ext.additionalDensities.containsKey(output.getFilter(OutputFile.DENSITY))) {
            output.processManifest.doFirst {
                def manifestFile = new File(project.buildDir, "intermediates" + File.separator + "manifests" + File.separator + "density" + File.separator + output.getFilter(OutputFile.DENSITY) + File.separator + variant.buildType.name + File.separator + "AndroidManifest.xml")
                def manifestText = manifestFile.text
                for (String density : project.ext.additionalDensities.get(output.getFilter(OutputFile.DENSITY))) {
                    manifestText = manifestText.replaceAll("</compatible-screens>", "<screen android:screenSize=\"small\" android:screenDensity=\"${density}\" />\n" +
                            "<screen android:screenSize=\"large\" android:screenDensity=\"${density}\" />\n" +
                            "<screen android:screenSize=\"xlarge\" android:screenDensity=\"${density}\" />\n" +
                            "<screen android:screenSize=\"normal\" android:screenDensity=\"${density}\" />\n </compatible-screens>")
                }
                manifestFile.text = manifestText
            }
        }
    }
}
```

##### ABI分割

```
splits {
    abi {
        enable true
        reset()
        include 'x86', 'armeabi-v7a', 'mips'
        universalApk false
    }
}
```

##### 设置版本号

```
// map for the version codes
ext.versionCodes = ['mdpi':1, 'hdpi':2, 'xhdpi':3].withDefault {0}
import com.android.build.OutputFile
android.applicationVariants.all { variant ->
// assign different version code for each output
    variant.outputs.each { output ->
        output.versionCodeOverride = project.ext.versionCodes.get(output.getFilter(OutputFile.DENSITY)) * 1000000 + android.defaultConfig.versionCode
    }
}
```

#### 5.[通过product flavors来区分多种APK](https://medium.com/@wkalicinski/smallerapk-part-5-multi-apk-through-product-flavors-e069759f19cd#.niun1j7q1)

```
android {
    ...
    productFlavors {
        xhdpi {
            resConfigs "xhdpi"
            versionCode 300001
        }
        hdpi {
            resConfigs "hdpi"
            versionCode 200001
        }
        mdpi {
            resConfigs "mdpi"
            versionCode 100001
        }
        anydpi {
            versionCode 1
        }
    }
}
```

##### 基于最小SDK版本的多种APK区分

```
android {
    productFlavors {
        prelollipop {
            versionCode 1
        }
        lollipop {
            minSdkVersion 21
            versionCode 2
        }
    }
}

android {
    ...
    flavorDimensions “density”, “version”
    productFlavors {
        xhdpi {
            dimension "density"
            resConfigs "xhdpi"
            versionCode 4
        }
        //other densities here...
        anydpi {
            dimension "density"
            versionCode 1
        }
        prelollipop {
            dimension "version"
            versionCode 1
        }
        lollipop {
            dimension "version"
            minSdkVersion 21
            versionCode 2
        }
    }
}
```

#### 6.[图片优化Zopfli&WebP](https://medium.com/@wkalicinski/smallerapk-part-6-image-optimization-zopfli-webp-4c462955647d#.mi1u9osne)

不能使用``WebP``作为启动画面的图片，因为它加载比较慢。

```
android {
    ...
    aaptOptions {
        cruncherEnabled = false
    }
}
```

#### 7.[图片优化Shape和VectorDrawables](https://medium.com/@wkalicinski/smallerapk-part-7-image-optimization-shape-and-vectordrawables-ed6be3dca3f#.wysrztkjj)

``Shape Drawable``,``VectorDrawables``
通过下列代码控制哪些哪些图片会由vectorDrawbles生成  

```
android {
    ...
    defaultConfig {
        //if you're using Android Gradle plugin < 2.0.0
        //omit the vectorDrawables block
        vectorDrawables {
            generatedDensities = ["mdpi", "hdpi", "xhdpi"]
        }
    }
}
```

#### 8.[从APK打开本地库](https://medium.com/@wkalicinski/smallerapk-part-8-native-libraries-open-from-apk-fc22713861ff#.h0f9i91tk)
从6.0开始

```
<application
	android:extractNativeLibs="false"
	>
```

### 总结
* 使用一套资源
* 使用``minifyEnabled``混淆代码
* 使用``shrinkResources``去除无用资源
* 删除无用语言资源
* 使用``tinypng``有损压缩
* 使用``jpg``格式
* 使用``webp``
* 优化``.so``文件，有些可以删除
* 缩小图片
* 使用微信资源压缩打包工具[AndResGuard](https://github.com/shwenzhang/AndResGuard)
* 使用``provided``编译
* 使用``shape``背景
* 使用``DrawableCompat``
* 考虑资源在线化
* 避免重复库
* 使用更小的库
* 使用插件化
* 精简功能业务