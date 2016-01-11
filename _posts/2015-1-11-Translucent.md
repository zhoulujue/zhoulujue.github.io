---
layout: post
title: Android : Translucent 和 Transition的冲突
---
  
    


设置 Theme 时，如果希望Activity背景是透明的，可以设置当前使用的Theme集成Theme.Translucent

设置以后，你会发现 Activity 默认的 Transition 动画都消失了

原因是 Theme.Translucent 里使用了这个属性


    <!-- Note that we use the base animation style here (that is no animations) 
         because we really have no idea how this kind ofactivity will be used. -->
    <item name="android:windowAnimationStyle">
         @android:style/Animation
    </item>
    


我的解决方案是，不集成 Theme.Translucent 而是将 Theme.Translucent 所有的属性拷贝到自己的Theme里，然后去掉这个android:windowAnimationStyle，重现build and run 就好了

