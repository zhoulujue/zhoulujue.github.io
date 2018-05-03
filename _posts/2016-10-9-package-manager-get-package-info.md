---
layout: post
title: PackageManager.getPackageInfo 指定合适的FLAG避免Binder错误
---

如果只是获取版本号和版本名称，那么使用 PackageManager 的时候不要使用 类似于 `GET_ACTIVITIES` 等 FLAG
可以尝试用 `GET_META_DATA`