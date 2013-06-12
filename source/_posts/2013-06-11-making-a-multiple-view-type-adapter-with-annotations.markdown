---
layout: post
title: "Making a multiple view types Adapter with Annotations"
date: 2013-06-11 19:49
comments: true
description: "In this post, I will show you a different way to create a multiple view types adapter. I will use a custom adapter and java annotations."
keywords: "android, java, annotation, annotations, multiple view types, different row, adapter, custom adapter, listview, baseadapter"
categories: [Adapter, Listview]
---


This post is more an introduction to java annotations than a tutorial about how to create an adapter able to show different rows. I won't describe how to implement a custom `Adapter`. Indeed, `ListView`s and `Adapter`s are common things in Android and you can easily find a [tutorial][ViewHolder tutorial] about such components.

<!-- more -->

Delegates
=========
Today I faced a problem. I needed a `ListView` able to display many different types of row. This is the common implementation of the `getView` method:

```java
@Override
public int getViewTypeCount() {
	return fNUM_TYPES;
}
	

public View getView(etc...){
	if(convertView == null){
		switch(getItemViewType(position)){
			//ugly things happen here…
			case 0: //create view 0 + viewHolder0 and other stuff
				break;
			case 1: //create view 1 + viewHolder1 and other stuff
				break;
			case 2: //create view 2 + viewHolder2 and other stuff
				break;
			case 3: //ok you get it
```

This implementation works well if you have max 2 types of rows (ok, you will have 2 in most cases, with sections and items), but if you have more types, your `getView(…)` is going to be hard to read and maintain.

The first idea is to create custom classes in which we can delegate the `getView`.

Create an interface called `DelegateAdapter`:

```java
public interface DelegateAdapter {
	public View getView(int position, View convertView, ViewGroup parent, LayoutInflater inflater, Object item);
}
```

Then you have to create a custom class implementing DelegateAdapter for each type you can have in your list. For example:

```java
public class SimpleTextDelegateAdapter implements DelegateAdapter {

	@Override
	public View getView(int position, View convertView, ViewGroup parent, LayoutInflater inflater, Object item) {
		if(convertView == null){
			//same as always… create your view, set a viewholder, etc.
		}
	}
```

Then you can call the right delegate on your `getView()`:

```java
@Override
public int getViewTypeCount() {
	return mDelegateAdapterSparseArray.size();
}
	

public View getView(...){
	switch(getItemViewType(position)){
		case 0: return mDelegate0.getView(…)
			break;
		case 1: return mDelegate1.getView(…)
			break;
		case 2: return mDelegate2.getView(…)
			break;
		case 3: return mDelegate3.getView(…)
```

Ok that's better but we still have our switch case… Let's create a `LongSparseArray` ([what is this?][LongSparseArray]) to get rid of this switch.

```java
private   LongSparseArray<DelegateAdapter> mDelegateAdapterSparseArray;

/**
 * Constructor
 */
public MyCustomAdapter(...){
	//some initializations…
	initDelegates(); 
}

private void initDelegates(){
	mDelegateAdapterSparseArray = new LongSparseArray<DelegateAdapter>();
	
	//the first parameter represents the ItemViewType
	mDelegateAdapterSparseArray.put(0, new SimpleTextDelegateAdapter());
	mDelegateAdapterSparseArray.put(1, new HtmlTextDelegateAdapter());
	mDelegateAdapterSparseArray.put(2, new GalleryDelegateAdapter());
	//etc.
}

public View getView(...){
	DelegateAdapter adapter = mDelegateAdapterSparseArray.get(getItemViewType(position));
	if(adapter != null){
		convertView = adapter.getView(…);
	}else{
		//default case! this should not happen :p
	}
	return convertView;
}
```

Ok we could stop there, but I never created java annotations and I'm very curious… so I found a way to use it in that case.


Annotations
=========== 
Annotations are created with the `@interface` keyword. For each annotation you can define some parameters:

- `@Target` can specify the place your annotation can be used (as a class header, method header, etc.).

- `@Retention` specify how long your annotation will live. It can be `SOURCE`, `CLASS` or `RUNTIME`. 
 1. `SOURCE`: The annotations are not saved in `*.class` files. 
 2. `CLASS`:  The annotations are saved in `*.class` files, but can't be used by the VM. *(default parameter)*
 3. `RUNTIME`: Annotations save in `*.class` and can be used in runtime. Since we will use reflection, we need this parameter.

Please note that I'm showing you one way to do this but there are plenty.


We will need 2 different annotations:

- `@DelegateAdapters`: used on our `BaseAdapter` to indicate which adapters to delegate the `getView`. It will take the delegate classes as parameter.

- `@DelegateAdapterType`: used on our `DelegateAdapter`s implementations. This annotation take as parameter his `itemViewType` associated.


`DelegateAdapters` Annotation:

```java
@Target(ElementType.TYPE)	
@Retention(RetentionPolicy.RUNTIME)
public @interface DelegateAdapters {
	Class<? extends DelegateAdapter>[] delegateAdapters();
}
```

`DelegateAdapterType` Annotation:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DelegateAdapterType {
	long itemType();
}
```

***Note:*** *You can set the itemType as optional if you set a default value:*

```java 
long itemType() default 0;
```

We will use our annotations like that:

In our base adapter:

```java
@DelegateAdapters(delegateAdapters = {
		SimpleTextDelegateAdapter.class,
		HtmlTextDelegateAdapter.class,
		GalleryDelegateAdapter.class
})
public class MyCustomAdapter extends BaseAdapter{
	…
}
```

And in our delegates:

```java
@DelegateAdapterType(itemType = 0) //note that you can use constants
public class SimpleTextDelegateAdapter implements DelegateAdapter {
	…
}

@DelegateAdapterType(itemType = 1) //note that you can use constants
public class HtmlTextDelegateAdapter implements DelegateAdapter {
	…
}

@DelegateAdapterType(itemType = 2) //note that you can use constants
public class GalleryDelegateAdapter implements DelegateAdapter {
	…
}
```

Now our adapters are ready, we just need to initialize the SparseArray using reflection in the Base Adapter:

```java
private void initDelegates(){
	mDelegateAdapterSparseArray = new LongSparseArray<DelegateAdapter>();
	
	//get the annotation containing all the delegate classes
	DelegateAdapters annotation = getClass().getAnnotation(DelegateAdapters.class);
	if (annotation != null) {
		Class[] clazzs = annotation.delegateAdapters();
		for (Class<?> clazz : clazzs) {
			DelegateAdapterType delegateAdapterAnnotation = clazz.getAnnotation(DelegateAdapterType.class);
			//check if each delegate has an itemType
			if(delegateAdapterAnnotation == null){
				throw new RuntimeException("The class "+clazz.getName()+" should have the annotation DelegateAdapterField");
			}
			
			long itemtype = delegateAdapterAnnotation.itemType();
			if(mDelegateAdapterSparseArray.get(itemtype) != null){
				throw new RuntimeException("The item type "+itemtype+" is already defined!");
			}
			
			//instantiate with the default constructor
			DelegateAdapter adapter = null;
			try {
				adapter = (DelegateAdapter) clazz.newInstance();
			} catch (Exception e) {
				throw new RuntimeException("Error while instanciating "+clazz+" with default constructor: "+e.getMessage(), e);
			}

			//final step!
			mDelegateAdapterSparseArray.put(itemtype, adapter);
		}
	}
	
}
```

Conclusion
==========

I hope you found it interesting. Please note this is my first custom annotation, so don't hesitate to tell me (via comments / g+ / twitter, etc.) if you have any remark.








[LongSparseArray]:http://developer.android.com/reference/android/support/v4/util/LongSparseArray.html
[ViewHolder tutorial]:http://castorflex.github.io/listview-optimisations-part-1-the-viewholder/