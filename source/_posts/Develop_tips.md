---
title: Develop Tip
date: 2015-03-07 15:16:24
tags: Android
---

### 开发记录

#### Debug

```
FragmentManager.enableDebugLoggin(true)

android.os.Debug.waitForDebugger();

//trigger your service
<service
	android:name="...."
	android:exported="true"
	android:process=":sync" />

```

#### Log
[hugo](https://github.com/JakeWharton/hugo)