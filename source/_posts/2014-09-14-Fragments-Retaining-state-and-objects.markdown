---
layout: post
title: "Fragments. Retaining state and objects."
date: 2014-09-14 23:04:06 +0300
comments: true
categories: 

--- 

There is a lot of good stuff written already about Activity and Fragment lifecycle and how to manage state and object retaining properly. So following is just a recap for future myself, rather than a tutorial.

Let's take a look at **Activities** and their destiny. There are 3 cases when and how activity can be destroyed.

<!-- more -->
**First case**: User enters activity *A*, from there goes to *B*, and then navigates back to *A*. Android won't save any instance state of activity *B* because user won't need this particular instance anymore. When he comes again to *B* he will get a fresh instance.

**Second case**: User enters activity *A* and rotates screen (or other configuration change occurs). Android has to recreate activity, so first it calls `onSaveInstanceState(Bundle outState)`, destroys *A* and creates new instance, then calls `onRestoreInstanceState(Bundle savedInstanceState)` to restore state of views (the same can be done in `onCreate(Bundle savedInstanceState)`).

**Third case**: User enters activity *A*, from there goes to *B*, *C*, *D*… up the activity stack. At some point Android kills *A* to reclaim memory (or because you turned on useful feature *Developer options -> Don't keep activities*). Because user may eventually come back to *A*, system saves *A*'s state before killing and later recreates it with `savedInstanceState`.  
----------------------------------------------------------------------------------------------------- <br>**Fragment** is tightly coupled to activity's lifecycle. It also has methods `onSaveInstanceState()`, ~~onRestoreInstanceState()~~ you can do restore in `onActivityCreated()` or `onViewStateRestored()`.

This is how I usually create fragment. Why prefer static factory method, see for example [here](http://www.androiddesignpatterns.com/2012/05/using-newinstance-to-instantiate.html):

```java
public static MyFragment newInstance(int index) {
	MyFragment fragment = new MyFragment();
	Bundle args = new Bundle();
	args.putInt("index", index);
	fragment.setArguments(args);
	return fragment;
}
```

Whatever we pass in arguments bundle also gets saved in case fragment has to be recreated later. Great! Now, let's say we have to pass an object to a fragment which is non-serializable. We can't put it in bundle, so my first thought would be:

```java
public class MyFragment extends Fragment {
	private Wallet wallet;
	
	@Override
	public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		// non-serializable object
		wallet = ((WalletApplication) getActivity().getApplication()).getWallet();
	}
}
```
I think this get.get.get and a cast could [look better](http://haacked.com/archive/2009/07/14/law-of-demeter-dot-counting.aspx/). Ideally, we should not "talk to strangers", so we could use some DI here (read more on DI forms in [Martin Fowler's article](http://www.martinfowler.com/articles/injection.html#FormsOfDependencyInjection)):  

* Constructor injection
* Setter injection
* Interface injection

```java
public static MyFragment newInstance(int index, Wallet wallet) {
	MyFragment fragment = new MyFragment();
	fragment.setWallet(wallet); 
	/// index goes into args 
	return fragment;
}
```
Now any activity knows, that in order to construct this fragment it'd have to provide a Wallet dependency.

Run the code, rotate the device and get *NPE*. This is our **second case**: Activity is recreated, so is fragment by calling no-arg constructor together with the same bundle that was passed to `setArguments` last time. Instance fields of course lost, `wallet` is null. [Android doc](http://developer.android.com/guide/topics/resources/runtime-changes.html#RetainingAnObject) helps us solve this by adding `setRetainInstance(true)`. Rotating works fine now, fragment instance is saved in memory and not recreated.

When we get to **third case**, there is no retaining like with configuration change. So we end up with restrictions, that default constructor has to have no arguments - `new Fragment()`, and static factory method `newInstance(Dependency)`, where we could pass a parameter, won't be called at the recreation time.

We can code following:

```java
public class MainActivity extends FragmentActivity
		implements WalletProvider {
	…
	@Override
	public Wallet provideWallet() {
		return wallet;
	}
	…
}

public class TransactionsFragment extends ListFragment {
	private Wallet wallet;
	
	@Override
	public void onActivityCreated(Bundle savedInstanceState) {
		super.onActivityCreated(savedInstanceState);
	
		// get Wallet dependency
		try {
			wallet = ((WalletProvider) getActivity()).provideWallet();
		} catch (ClassCastException e) {
			throw new ClassCastException(getActivity().getClass().getSimpleName() +
			                             " should implement WalletProvider interface");
		}
	
	}
}
```
Why do I put this in `onActivityCreated()`, rather than `onCreate()`? Because fragment's `onCreate()` is not guaranteed to be called **after** activity's `onCreate()` and our dependency might not yet be initialized in activity.

---------------------------------------------------------------------------------------------------------------------- 
 <br>
`setRetainInstance(true)` has also couple of features you should keep in mind.

* `Bundle savedInstanceState` is always null. This kinda makes sense, why would you want to restore state if fragment is not destroyed?

* Fragment's lifecycle becomes slightly different, i.e. `onCreate()` and `onDestroy()` won't be called:

<a href="/images/fragment-lifecycle-setRetainInstance_horiz.png"><img class="caption" src="/images/fragment-lifecycle-setRetainInstance_horiz.png" width="1213" height="162" /></a>

In my case I had `wallet.addEventListener(txEventListener)` in `onCreate()` and `wallet.removeEventListener(txEventListener)` in `onDestroy()`. It was perfect, since these are not called on config change with retained instance and I wasn't removing-adding the same listener. However, if I add listener in `onActivityCreated()` to be sure that activity is fully created, I have to remove it in `onDestroyView()`.
  




