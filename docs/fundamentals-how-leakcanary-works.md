## Detecting retained objects

The foundation of LeakCanary is the `leakcanary-object-watcher-android` library. It hooks into the Android lifecycle to automatically detect when activities and fragments are destroyed and should be garbage collected. These destroyed objects are passed to an `ObjectWatcher`, which holds **weak references** to them. You can watch any objects that is no longer needed, for example a detached view, a destroyed presenter, etc.

```kotlin
AppWatcher.objectWatcher.watch(myDetachedView, "View was detached")
```

If the weak references aren't cleared after **waiting 5 seconds** and running garbage collection, the watched objects are considered **retained**, and potentially leaking. LeakCanary logs this to Logcat:

```
D LeakCanary: Watching instance of com.example.leakcanary.MainActivity
  (Activity received Activity#onDestroy() callback) 

... 5 seconds later ...

D LeakCanary: Scheduling check for retained objects because found new object
  retained
```

LeakCanary waits for the count of retained objects to reach a threshold before dumping the heap, and displays a notification with the latest count.

![notification](images/retained-notification.png)

```
D LeakCanary: Rescheduling check for retained objects in 2000ms because found
  only 4 retained objects (< 5 while app visible)
```

!!! info
    The default threshold is **5 retained objects** when the app is **visible**, and **1 retained object** when the app is **not visible**. If you see the retained objects notification and then put the app in background (for example by pressing the Home button), then the threshold changes from 5 to 1 and LeakCanary dumps the heap within 5 seconds. Tapping the notification forces LeakCanary to dump the heap immediately.

## Dumping the heap

When the count of retained objects reaches a threshold, LeakCanary dumps the Java heap into a `.hprof` file stored onto the Android file system (see [Where does LeakCanary store heap dumps?](faq.md#where-does-leakcanary-store-heap-dumps)). This freezes the app for a short amount of time, during which LeakCanary displays the following toast:

![toast](images/dumping-toast.png)

## Analyzing the heap

LeakCanary parses the `.hprof` file using [Shark](shark.md) and locates the retained objects in that heap dump.

![done](images/finding-retained-notification.png)

For each retained object, LeakCanary finds the path of references which prevents that retained object from being garbage collected: its **leak trace**. Leak trace is another name for the *best strong reference path from garbage collection roots to a retained object*. 

![done](images/building-leak-traces-notification.png)

When the analysis is done, LeakCanary displays a **notification** with a summary, and also prints the result in **Logcat**. Notice below how the **4 retained objects** are grouped as **2 distinct leaks**. LeakCanary creates a **signature for each leak trace**, and groups together leaks that have the same signature, ie leaks that are caused by the same bug.

![done](images/analysis-done.png)

```
====================================
HEAP ANALYSIS RESULT
====================================
2 APPLICATION LEAKS

References underlined with "~~~" are likely causes.
Learn more at https://squ.re/leaks.

58782 bytes retained by leaking objects
Displaying only 1 leak trace out of 2 with the same signature
Signature: ce9dee3a1feb859fd3b3a9ff51e3ddfd8efbc6
┬───
│ GC Root: Local variable in native code
│
...
```

Tapping the notification starts an activity that provides more details. Each row corresponds to a group of leaks with the same signature. LeakCanary will mark a leak as **New** if it's the first time you've seen a leak with that signature.

![toast](images/heap-dump.png)

Tapping into a leak opens up a screen where you can see each retained object and its leak trace. You can toggle between retained objects via a drop down.

![toast](images/leak-screen.png)

The leak signature is the hash of the concatenation of each **<span style="color: #9976a8;">reference</span>** suspected to cause the leak, ie each reference **<span style="text-decoration: underline; text-decoration-color: red; text-decoration-style: wavy; color: #9976a8;">displayed with a red underline</span>**:

![toast](images/signature.png)

These same suspicious references are underlined with `~~~` when the leak trace is shared as text:

```
...
│  
├─ com.example.leakcanary.LeakingSingleton class
│    Leaking: NO (a class is never leaking)
│    ↓ static LeakingSingleton.leakedViews
│                              ~~~~~~~~~~~
├─ java.util.ArrayList instance
│    Leaking: UNKNOWN
│    ↓ ArrayList.elementData
│                ~~~~~~~~~~~
├─ java.lang.Object[] array
│    Leaking: UNKNOWN
│    ↓ Object[].[0]
│               ~~~
├─ android.widget.TextView instance
│    Leaking: YES (View.mContext references a destroyed activity)
...
```

In the example above, the signature of the leak would be computed as:

```kotlin
val leakSignature = sha1Hash(
    "com.example.leakcanary.LeakingSingleton.leakedView" +
    "java.util.ArrayList.elementData" +
    "java.lang.Object[].[x]"
)
println(leakSignature)
// dbfa277d7e5624792e8b60bc950cd164190a11aa
```

## Library leaks

LeakCanary separates the leaks it finds in your app into two categories: **Application Leaks** and **Library Leaks**.

A **Library Leak** is a leak caused by a known bug in 3rd party code that you do not have control over. This leak is impacting your application, but unfortunately fixing it may not be in your control so we separate them out.

You can see the two categories of leak in the result printed in **Logcat**:

```
====================================
HEAP ANALYSIS RESULT
====================================
0 APPLICATION LEAKS

====================================
1 LIBRARY LEAK

...
┬───
│ GC Root: Local variable in native code
│
...
```

The LeakCanary UI adds a <span style="border-radius: 20px; background: #4e462f; padding-left: 8px; padding-right: 8px; padding-top: 2px; padding-bottom: 2px; color: #ffcc32;">Library Leak</span> tag:

![Library Leak](images/library-leak.png)

LeakCanary ships with a database of known leaks, which it recognizes by pattern matching on reference names, for example:

```
Leak pattern: instance field android.app.Activity$1#this$0
Description: Android Q added a new IRequestFinishCallback$Stub class [...]
┬───
│ GC Root: Global variable in native code
│
├─ android.app.Activity$1 instance
│    Leaking: UNKNOWN
│    Anonymous subclass of android.app.IRequestFinishCallback$Stub
│    ↓ Activity$1.this$0
│                 ~~~~~~
╰→ leakcanary.internal.activity.LeakActivity instance
```

!!! quote "What did I do to cause this leak?"
    Nothing wrong! Most likely you used an API the way it was intended but the implementation has a bug that is causing this leak.

!!! quote "Is there anything I can do to prevent it?"
    Maybe! Some Library Leaks can be fixed using reflection, others by exercising a code path that makes the leak go away. This type of fix tends to be hacky, so beware! Your best option might be to find the bug report or file one, and insist that it must get fixed.

!!! quote "Since I can't do much about this leak, is there a way I can ask LeakCanary to ignore it?"
	There's no way for LeakCanary to know whether a leak is a Library Leak or not prior to dumping the heap and analyzing it. If we didn't show the result notification when a Library Leak is found then you'd start wondering what happened to the LeakCanary analysis after the dumping toast.

You can see the full list of known leaks in the [AndroidReferenceMatchers](https://github.com/square/leakcanary/blob/master/shark-android/src/main/java/shark/AndroidReferenceMatchers.kt#L49) class. If you find an Android SDK leak that isn't recognized, please [report it](faq.md#can-a-leak-be-caused-by-the-android-sdk). You can also [customize the list of known Library Leaks](recipes.md#matching-known-library-leaks).

What's next? Learn how to [fix a memory leak](fundamentals-fixing-a-memory-leak.md)!
