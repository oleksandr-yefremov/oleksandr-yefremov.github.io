---
layout: post
title: "Fragment FTW!"
date: 2014-09-14 23:04:06 +0300
comments: true
categories: 

--- 

There is a lot of stuff written already about Activity and Fragment lifecycle and how to manage state and object retaining properly. So this is just a recap for future myself, rather than a tutorial.

Let's take a look at **Activities** and their destiny. There are 3 cases when and how activity can be destroyed.

<!-- more -->
**First case**: User enters activity *A*, from there goes to *B*, and then navigates back to *A*. Android won't save any instance state of activity *B* because user won't need this particular instance anymore. When he comes again to *B* he will get a fresh instance.

**Second case**: User enters activity *A* and rotates screen (or other configuration change occurs). Android has to recreate activity, so first it calls `onSaveInstanceState(Bundle outState)`, destroys *A* and creates new instance, then calls `onRestoreInstanceState(Bundle savedInstanceState)` to restore state of views (the same can be done in `onCreate(Bundle savedInstanceState)`).

**Third case**: User enters activity *A*, from there goes to *B*, *C*, *D*… up the activity stack. At some point Android kills *A* to reclaim memory (or because you turned on useful feature *Developer options -> Don't keep activities*). Because user may eventually come back to *A*, system saves *A*'s state before killing and later recreates it with `savedInstanceState`.  
----------------------------------------------------------------------------------------------------- <br>**Fragment** is tightly coupled to activity's lifecycle. It also has methods `onSaveInstanceState()`, `onRestoreInstanceState()` and knows how to save its internal state.

This is how to create a fragment. It is common to put this code into static factory method (see why for example [here](http://www.androiddesignpatterns.com/2012/05/using-newinstance-to-instantiate.html)):

```java
public static MyFragment newInstance(int index) {
	MyFragment fragment = new MyFragment();
	Bundle args = new Bundle();
	args.putInt("index", index);
	fragment.setArguments(args);
	return fragment;
}
```

Whatever we pass in arguments bundle, gets saved if fragment has to be recreated later. Now let's say we have a clear dependency in our fragment:

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
Personally, I think this get.get.get and a cast is rather ugly. Ideally, we should have no knowledge about parent activity, so we gonna use some DI magic (read more on DI forms [in Martin Fowler's article](http://www.martinfowler.com/articles/injection.html#FormsOfDependencyInjection):  

* Constructor injection
* Setter injection
* Interface injection

```java
public static MyFragment newInstance(int index, Wallet wallet) {
	MyFragment fragment = new MyFragment();
	fragment.wallet = wallet;
	/// index goes into args 
	return fragment;
}
```
Now any activity knows, that in order to construct this fragment it'd have to provide a Wallet dependency to it.

Run the code, rotate the device and get *NPE*. This is **second case** from above: Activity is recreated, so is fragment by calling no-arg constructor together with the same bundle that was passed to `setArguments` last time. Instance fields of course lost, `wallet` is null. [Android doc](http://developer.android.com/guide/topics/resources/runtime-changes.html#RetainingAnObject) helps us solve this by adding `setRetainInstance(true)`. Rotating works fine now, fragment instance is saved in memory and not recreated.

Speaking about **third case**, there is no retaining like with configuration change. So we end up with restrictions, that default constructor has to have no arguments - `new Fragment()`, and static factory method `newInstance(Dependency)`, where we could pass a parameter, won't be called at the recreation time.

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
<div style="float: right;"><img class="caption" src="/images/fragment-lifecycle-setRetainInstance.png" width="317" height="847" title="test caption"></img></div>

Why do I put this in `onActivityCreated()`, rather than `onCreate()`? We know, that fragment lifecycle looks a little bit different with `setRetainInstance(true)`:  




If we use `setRetainInstance(true)`, we could put this code into `onCreate()` so 
Be sure to **not** move this code code into `onCreate()`, because fragment's `onCreate` is not guaranteed to be called **after** activity's and this can cause you some *"Debugging Time, come on grab your friends"*.


Now every time activity is recreated








