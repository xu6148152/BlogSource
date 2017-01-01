---
title: Glide源码分析
tags: 源码分析
---

### Glide源码分析

正常使用方式是``Glide.with(xxx).load(xxx).into(imageview)``
因此，我们首先来看看``Glide.java``

```java
public static RequestManager with(Activity activity) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(activity);
  }
```

``RequestManagerRetriever.java``内部采用``Handler``来处理外部的请求，内部还会创建``RequestManager``，其会管理请求的生命周期，与``Activity, Fragment``同步。

```java
public RequestBuilder<Drawable> load(@Nullable Object model) {
    return asDrawable().load(model);
  }
```

主要就是根据``model``创建``RequestBuilder``

```java
RequestBuilder.java

public Target<TranscodeType> into(ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      if (requestOptions.isLocked()) {
        requestOptions = requestOptions.clone();
      }
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions.optionalCenterCrop(context);
          break;
        case CENTER_INSIDE:
          requestOptions.optionalCenterInside(context);
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions.optionalFitCenter(context);
          break;
        //$CASES-OMITTED$
        default:
          // Do nothing.
      }
    }

    return into(context.buildImageViewTarget(view, transcodeClass));
  }
```

```java
RequestManager.java
```

在``Glide``的构造函数中会把所有默认的组件注册

```java
BitmapPool.java
```