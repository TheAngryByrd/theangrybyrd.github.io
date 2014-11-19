---
layout: post
title: "Visual Studio 2015 Preview Android error
published: true
---
## Visual Studio 2015 Preview Android error.

So I've been playing with Visual Studio 2015 because I was curious about the [new Android emulator running in Hyper-V](http://blogs.msdn.com/b/visualstudioalm/archive/2014/11/12/introducing-visual-studio-s-emulator-for-android.aspx).  I paired it up with Xamarin and tried to test out a few apps.  THe only problem is the apps wouldn't launch nor would they run in the debugger.  I was getting a Mono.Debugger.Soft.ConnectionException message.

![Scenario 1]({{ site.url }}/images/VS2015Android/VS2015Android.jpg)

Using my google-fu I found this [post](http://forums.xamarin.com/discussion/4083/debug-session-not-start-when-i-hit-f5-for-hello-world-project) on the Xamarin forums suggesting to turn off fast deployment.

This can be done by right clicking the Android startup project |> Properties |> Android Options |> Uncheck Use Fast Deploy

![Scenario 1]({{ site.url }}/images/VS2015Android/FastDeploy.jpg)
