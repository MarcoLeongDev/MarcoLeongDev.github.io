---
published: true
layout: post
date: '2019-04-07 12:12:12 +1000'
categories: lifecycle flutter ios android widgetsBendingobserver
---

![lifecycle.png]({{site.baseurl}}/_posts/1*sInVXSx62Kc0ilTdPettNQ.png)



Flutter a handy framework to work with even for a long time iOS developer like myself. One of the first questions that came to my mind when I first started with Flutter is: where is the basic app lifecycle!?

In iOS there are protocols in place for the **UIAppDelegate** like _applicationDidBecomeActive_, _applicationDidEnterBackground_, _applicationWillResignActive_, etc, and **UIViewController** with _init()_, _viewDidLoad()_, _viewWillDisappear()_, etc. These protocol are the keys to create Apps that retain the state for the next time it is (re)launched.

The anwser, at least for **UIAppDelegate**, is to use **WidgetsBindingObserver**. Start the project off with the default app, you should get something like this.



![1*DWaTBzpCrsz5sBlgKnhY8g.png]({{site.baseurl}}/_posts/1*DWaTBzpCrsz5sBlgKnhY8g.png)


Modify the **\_MyHomePageState** with the following code (just copy and paste):


```
class _MyHomePageState extends State<MyHomePage> with WidgetsBindingObserver {
  ...
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }
  @override
    void dispose() {
    super.dispose();
  }
  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
    case AppLifecycleState.inactive:
    print("Inactive");
    break;
    case AppLifecycleState.paused:
    print("Paused");
    break;
    case AppLifecycleState.resumed:
    print("Resumed");
    break;
    case AppLifecycleState.suspending:
    print("Suspending");
    break;
  }
  ...
}
```

The class should look like this:

![1*Gjd8AFwSKIhhFOrVADPrbA.png]({{site.baseurl}}/_posts/1*Gjd8AFwSKIhhFOrVADPrbA.png)

Run and test it, results:

![1*i5rvx4XygAYYqTupjm5PZw.png]({{site.baseurl}}/_posts/1*i5rvx4XygAYYqTupjm5PZw.png)

It is just this simple, happy coding
