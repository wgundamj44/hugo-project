---
layout: post
title: Workflow of AvatarDrive's native code (Android)
category: [tech]
tags: [cocos2d, android]
---
Some note of my invistigation into AvatarDrive's source code.
<!--more-->

### StartActivity
This is where all the story starts. It does nothing, but invoke webview Activity.

### MainWebViewActivity
As as hybrid game, most of the UI interaction happens in this webview activity.
When user is about to redirect to native battle logic, the this view intercepts cirtain URL with `shouldOverrideUrlLoading`, and native logic will take controll.
QuestBattleActivity will lanuch.

### QuestBattleActivity, Cocos2dActivity, Cocos2dRender
This is a subclass of Cocos2dActivity. Here we parse the intent passed in from webview, and put all the neccesary data for battle.
In onCreate, all the data neccesary for game start will be set, and loading effect starts to run. In onCreateView, Cocos2dRender is initialized and attached to activity. Now that render thread has started, in onSurfaceCreated method, native method nativeInit is called.

### jni/main.cpp
The implementation of nativeInit lies in main.cpp. Here I found my old friend AppController. AppController is newed here with:

```cpp
AppController *pAppController = new AppController();
```

And CCApplication 's run is called. run simplely calls applicationDidFinishLaunching, which in turn creates proper scene for battle, as:

```cpp
pScene = GvGBattle::scene();
PDirector->runWithScene(pScene)
```

### Scenes defiend in Classes
Here is nothing special but cocos2d code.

### How does system find native code?
In QuestbattleActivity, System.loadLibrary is called to load built C++ library, as:

```java
 static {
         System.loadLibrary("game");
    }
```
And the C++ library is descriped in jni/Android.mk where make options are defined. In this file `LOCAL_MODULE_FILENAME := libgame` defines the name of built library.

