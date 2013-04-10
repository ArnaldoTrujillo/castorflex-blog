---
title: Using Monkey tool
author: Antoine Merle
layout: post
permalink: /using-monkey-tool/
categories:
  - Tests
---
# 

This first post is dedicated to a wonderfull tool => [Monkey][1]!

 [1]: http://developer.android.com/tools/help/monkey.html "Monkey"

If you had never heard of this tool, your life is turning better now. Monkey is a simple tool which allow to **generate pseudo-random gestures**, on a **device** or on an **emulator**. Shortly, you can in a few seconds simulate a user using your application, but a pretty stupid one (just as all users ?) because all his actions will be generated randomly. These actions won’t make any sense, but the aim here is to **detect bugs** in your app, and this can be very useful.

***Note 1***: *This is absolutely not a substitute to other tests you should do (like unit tests…). Do what I say, not what I do :p*

***Note 2***: *For this tut, I suppose you have one emulator or a device connected in debug mode (with ADB drivers installed). I won’t detail this part.*

Here we go, launching cmd, and browsing to *[android-sdk-path]**/platform-tools.* The basic command to use monkey is the following:

    adb shell monkey <options>

For example, if your application package is com.myapp and you want to generate 1000 gestures, with a delay of 500ms between each event, you have to use this command:

    adb shell monkey -p com.myapp --throttle 500 -v 1000

The list of all the options is available [here][1].

### Going further…

If you want to go one step further, you can automate a lot of things ,like the installation on the device, the launch, screenshots, etc. with the [monkeyrunner tool][2]

 [2]: http://developer.android.com/tools/help/monkeyrunner_concepts.html "monkeyrunner"

I hope you enjoyed my first tutorial, do not hesitate to comment if you have any question or share it if you liked it!