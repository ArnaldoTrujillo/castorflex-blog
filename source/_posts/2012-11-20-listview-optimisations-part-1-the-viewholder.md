---
title: 'ListView optimisations : Part 1 (the ViewHolder)'
author: Antoine Merle
layout: post
comments: true
permalink: /listview-optimisations-part-1-the-viewholder/
description: "Make your listview smoother with these tutorials. Part 1: ViewHolder"
keywords: "android, listview, smooth, optimization, tutorial, viewholder, getview"
categories:
  - Listview
---
# 

When you develop on Android, you will probably need to make lists pretty quickly. However, it can be complicated to understand all the mecanisms when you are a beginner. This serie of posts aims to give you tips if you want to increase the smoothness in your lists.

This first post is a basic one which talk about a well known tip, called ViewHolder. <!-- more -->This is absolutely essential if you want to have a viable list. But we must first understand how it works.

A list is composed by elements displayed in views, which ones are declared in xml files (generally). Usually, a list is composed by elements whose views are the same, only the content change. That’s why we use the same view for all the elements. The role of the adapter is to provide views to the list. It has to create and modify their contents.  
Here is an example of a basic custom adapter, providing elements with a name and a description :

```java
    public class MyAdapter extends BaseAdapter {
     
        private LayoutInflater mLayoutInflater;
        private List mData;
     
        public MyAdapter(Context context, List data){
            mData = data;
            mLayoutInflater = LayoutInflater.from(context);
        }
     
        @Override
        public int getCount() {
            return mData == null ?  : mData.size();
        }
     
        @Override
        public MyPojo getItem(int position) {
            return mData.get(position);
        }
     
        @Override
        public long getItemId(int position) {
            return position;
        }
     
        @Override
        public View getView(int position, View view, ViewGroup parent) {
     
            //This is bad, do not do this at home
            View vi = mLayoutInflater.inflate(R.layout.item, parent, false);
     
            TextView tvName = (TextView) vi.findViewById(R.id.textView_item_name);
            TextView tvDescription = (TextView) vi.findViewById(R.id.textView_item_description);
     
            MyPojo item = getItem(position);
     
            tvName.setText(item.getName());
            tvDescription.setText(item.getDescription());
     
            return vi;
        }
    }
```

That is the code we could make when we don’t know that listviews recycle their views. Indeed, when you scroll down, the views disappearing on the top are reused to display items on the bottom of your list. I hope that this is clear because it’s very important :p. As a consequence, we will tag our views in order to avoid to inflate views that already exists (this operation take long time).



We will write a class (called ViewHolder) in which references between the widgets will be saved :

```java
static class ViewHolder{
    TextView tvName;
    TextView tvDescription;
}
```
And the adapter :

```java
@Override
public View getView(int position, View view, ViewGroup parent) {

	View vi = view;             //trying to reuse a recycled view
	ViewHolder holder = null;

	if (vi == null) {
		//The view is not a recycled one: we have to inflate
		vi = mLayoutInflater.inflate(R.layout.item, parent, false);
		holder = new ViewHolder();

		holder.tvName = (TextView) vi.findViewById(R.id.textView_item_name);
		holder.tvDescription = (TextView) vi.findViewById(R.id.textView_item_description);
		vi.setTag(holder);
	} else {
		// View recycled !
		// no need to inflate
		// no need to findViews by id
		holder = (ViewHolder) vi.getTag();
	}

	MyPojo item = getItem(position);

	holder.tvName.setText(item.getName());
	holder.tvDescription.setText(item.getDescription());

	return vi;
}
```
If you want to add other widgets in your elements (an image for example), you will have to add them in the ViewHolder too.

That’s all I’ll talk next time about the way you can optimize your images in your lists (LRUCache, etc.).
