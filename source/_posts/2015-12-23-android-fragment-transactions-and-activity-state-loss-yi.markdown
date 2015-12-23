---
layout: post
title: "Android Fragment Transactions &amp; Activity State Loss(译)"
date: 2015-12-23 14:56:15 +0800
comments: true
categories: 
---
本文翻译自[Fragment Transactions & Activity State Loss](http://www.androiddesignpatterns.com/2013/08/fragment-transaction-commit-state-loss.html)，原作者为[Alex Lockwood](https://plus.google.com/+AlexLockwood), 译者为[Jared Luo](https://github.com/jaredlam)

自从Honeycomb版本发布以后，下面这一块异常消息就成了StackOverflow上问题的常客：

    java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
    at android.support.v4.app.FragmentManagerImpl.checkStateLoss(FragmentManager.java:1341)
    at android.support.v4.app.FragmentManagerImpl.enqueueAction(FragmentManager.java:1352)
    at android.support.v4.app.BackStackRecord.commitInternal(BackStackRecord.java:595)
    at android.support.v4.app.BackStackRecord.commit(BackStackRecord.java:574)

这篇文章就来给大家解释一下为什么这个异常会被抛出，并且会总结出一些建议，让您的APP再也不受其导致崩溃的困扰。

### 异常产生的原因？
这个异常会被抛出的原因是：**在Activity的状态被保存之后，你才尝试去commit一个FragmentTransaction，这样就会导致Activity的状态丢失**。在深入了解这句话真正的意思之前，让我们先来看看当onSaveInstanceState()被调用时，其内部发生了什么。正如我上一篇文章[Binders & Death Recipients](http://www.androiddesignpatterns.com/2013/08/binders-death-recipients.html)所说，Android应用程序在Android运行时当中对其自身的命运，几乎没有什么话语权。Android系统有权利随时杀掉任何一个进程来解放内存空间，这样就导致后台运行的Activity们随时都有可能，在没有一点警告的情况下被杀掉。为了保证这种不可预测的动作不被用户所感受到，系统框架给了Activity一个机会，在可能被杀掉之前，通过调用onSaveInstanceState()方法来保存状态。这样当保存的状态被恢复的时候，无论Activity是否被杀掉，用户都会感觉到好像前后台的Activity进行的是无缝的切换。

当框架调用onSaveInstanceState()的时候，它会向这个方法传递一个Bundle参数，Activity可以用这个参数来保存页面、对话框、Fragments和视图的状态。当onSaveInstanceState返回时，会将一个Bundle对象序列化之后通过Binder接口传递给系统服务进程,并安全的保存起来。当系统晚一点想要重启Activity的时候， 它会把之前的Bundle对象传递回应用，并用来恢复Activity之前的状态。

所以为什么会抛出之前的异常呢？问题的根源在于Bundle这个对象仅仅是Activity在onSaveInstanceState()方法被调用那一刻的快照。这就意味着当你如果在onSaveInstanceState()之后再调用FragmentTransaction.commit()的话，由于这次Transaction没有被作为Activity状态的一部分来保存，自然也就丢失掉了。从用户的角度来说，Transaction的遗失就导致意外的UI状态丢失。那么为了维护良好的用户体验，Android系统会不惜一切代价的避免页面状态的丢失，所以在这种情况发生的时候就直接抛出了IllegalStateException。

### 异常产生的时机？

如果你曾经遇见过这个异常，你可能会注意到在各个版本的Android平台上，这个异常抛出的时机略有不同。举个栗子，你可能会发现相对老的设备上这个异常可能会抛出的没有那么频繁，另外如果你的应用使用了support library的话，可能会比使用官方框架更容易崩溃一点。这个不同的现象可能会让我们假设support library是不是有bug，不能让我们信任呢？然而，并不是这样。

这种各个版本之间的现象不一致源于，在Honeycomb这个Android版本中，Activity的生命周期做了相当的调整。在Honeycomb之前的版本中，Activity在pause以前都是不能被杀掉的，所以onSaveInstanceState() 就会在onPause()之前被立刻调用。但是从Honeycomb开始，Activity变成了在stop之前不能被杀掉。所以自然onSaveInstanceState()就会变成在onStop()之前被立刻调用。下面的表格总结了这个区别：

上面的针对Activity生命周期的改变就导致了，有时候support library需要根据系统版本的不同, 来调整自己的行为。举个栗子，在Honeycomb和其以上版本的设备上，只要在onSaveInstanceState()以后调用commit(), 就会抛出异常来提示开发者页面状态丢失了。但是在Honeycomb之前的版本，因为onSaveInstanceState()被调用的时机提前了很多，导致状态的丢失更容易出现，所以每次都抛出异常就显得有点太严格了。Android开发团队不得不被迫做了一个妥协：为了更好的配合老版本的系统，Honeycomb之前的版本就必须忍受onPause()和onStop()之间可能会发生的状态丢失（而不会抛出异常，译者注）。
如下表格总结了Support library在两类版本的表现：

### 怎样避免异常出现？

一旦您了解了真正的原理，避免Activity状态丢失就会变得容易许多。如果您已经读到了这里，我希望您对support library对这类问题的处理方式，以及避免页面状态丢失的重要性，有了更深入的了解。如果您是来查找怎样能够快速修复这个问题，下面有一些关于使用FragmentTransactions的建议：

 - **谨慎的在Activity的生命周期方法中调用transaction的commit方法**。大多数应用只会在第一次调用onCreate()或者响应用户输入的时候去commit transaction， 这样不会有什么问题。但是，
当您把transaction放到其他Activity的生命周期方法中时，比如onActivityResult()， onStart() 和onResume()，事情就会变得有点复杂。举个栗子，您不应该在FragmentActivity#onResume()中去commit transaction，因为在某些情况下，这个方法可能会在Activity状态恢复之前被调用([看这里](http://developer.android.com/reference/android/support/v4/app/FragmentActivity.html#onResume()))。如果您的应用需要在onCreate()之外的其他生命周期方法中去commit transaction，那就请在 FragmentActivity#onResumeFragments()或者Activity#onPostResume()中去commit。这两个方法保证会在Activity恢复状态之后才会被调用，所以就能完全避免状态丢失的可能性。（作为例子，可以查看我在StackOverflow上，关于怎样在Activity#onActivityResult()中commit FragmentTransactions 的回答）。
 - 避免在异步回调中去处理transaction。包括使用AsyncTask#onPostExecute()和 LoaderManager.LoaderCallbacks#onLoadFinished()方法等。这样做的问题在于，您不会知道这些异步方法在调用的时候，当前的Activity究竟处于生命周期的哪一个状态。比如说，下面这一系列事件：

1.  Activity执行一个AsyncTask。
2.  当用户按下Home键，导致Activity的onSaveInstanceState()和onStop()执行。
3.  AsyncTask执行完成并且onPostExecute()被调用，其不知道Activity这个时候已经被stopped。
4.  FragmentTransaction在onPostExecute()中被调用，导致异常被抛出。

总体来说，在这种情况下，最好避免异常的方法是不要在异步回调中去commit transaction。Google的工程师们看起来也同意这个观点。根据Android开发者团队邮件组的这篇[文章](https://groups.google.com/forum/#!msg/android-developers/dXZZjhRjkMk/QybqCW5ukDwJ)，Android团队认为在异步方法中去调用commit transaction而产生的UI切换，会导致不好的用户体验。 如果您的应用确实需要在异步方法中处理transaction，并且没有简单的方法可以保证回调在onSaveInstanceState()之前被调用的话，你可能只有使用commitAllowingStateLoss()，并且自己处理可能发生的状态丢失了(请参考StackOverflow，[这里](http://stackoverflow.com/questions/8040280/how-to-handle-handler-messages-when-activity-fragment-is-paused)，[还有这里](http://stackoverflow.com/questions/7992496/how-to-handle-asynctask-onpostexecute-when-paused-to-avoid-illegalstateexception))。

- 只在不得已的情况下，才使用commitAllowingStateLoss()。commit()和commitAllowingStateLoss()唯一的不同是，后者当状态丢失时，不会抛出异常。通常由于存在状态丢失的可能性，您不会希望使用这个方法。更好的解决方法是，在您的应用中使用commit()，并保证在Activity的状态保存之前调用它，来获取更好的用户体验。除非状态丢失的可能性不能被避免，否则您不应该使用commitAllowingStateLoss()。

希望这些提示能够帮您解决这个异常带来的相关问题。如果您仍然有没有解决的疑问，请到[StackOverflow](http://stackoverflow.com/)上发帖子，并且把地址发布到下面的评论中，我可以帮您看看:)
