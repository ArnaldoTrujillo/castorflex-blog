---
title: 'ListView optimisations : Part 1 (the ViewHolder)'
author: Antoine Merle
layout: post
permalink: /listview-optimisations-part-1-the-viewholder/
categories:
  - Listview
---
# 

When you develop on Android, you will probably need to make lists pretty quickly. However, it can be complicated to understand all the mecanisms when you are a beginner. This serie of posts aims to give you tips if you want to increase the smoothness in your lists.

This first post is a basic one which talk about a well known tip, called ViewHolder. This is absolutely essential if you want to have a viable list. But we must first understand how it works.

A list is composed by elements displayed in views, which ones are declared in xml files (generally). Usually, a list is composed by elements whose views are the same, only the content change. That’s why we use the same view for all the elements. The role of the adapter is to provide views to the list. It has to create and modify their contents.  
Here is an example of a basic custom adapter, providing elements with a name and a description :

    public class MyAdapter extends BaseAdapter &#123;
    &nbsp;
        private LayoutInflater mLayoutInflater;
        private [List][1] mData;
    &nbsp;
        public MyAdapter&#40;[Context][2] context, [List][1] data&#41;&#123;
            mData = data;
            mLayoutInflater = LayoutInflater.from&#40;context&#41;;
        &#125;
    &nbsp;
        @Override
        public int getCount&#40;&#41; &#123;
            return mData == null ?  : mData.size&#40;&#41;;
        &#125;
    &nbsp;
        @Override
        public MyPojo getItem&#40;int position&#41; &#123;
            return mData.get&#40;position&#41;;
        &#125;
    &nbsp;
        @Override
        public long getItemId&#40;int position&#41; &#123;
            return position;
        &#125;
    &nbsp;
        @Override
        public [View][3] getView&#40;int position, [View][3] view, ViewGroup viewGroup&#41; &#123;
    &nbsp;
            //This is bad, do not do this at home
            [View][3] vi = mLayoutInflater.inflate&#40;R.layout.item, null&#41;;
    &nbsp;
            TextView tvName = &#40;TextView&#41; vi.findViewById&#40;R.id.textView_item_name&#41;;
            TextView tvDescription = &#40;TextView&#41; vi.findViewById&#40;R.id.textView_item_description&#41;;
    &nbsp;
            MyPojo item = getItem&#40;position&#41;;
    &nbsp;
            tvName.setText&#40;item.getName&#40;&#41;&#41;;
            tvDescription.setText&#40;item.getDescription&#40;&#41;&#41;;
    &nbsp;
            return vi;
        &#125;
    &#125;

That is the code we could make when we don’t know that listviews recycle their views. Indeed, when you scroll down, the views disappearing on the top are reused to display items on the bottom of your list. I hope that this is clear because it’s very important :p. As a consequence, we will tag our views in order to avoid to inflate views that already exists (this operation take long time).

 [1]: http://www.google.com/search?hl=en&q=allinurl:list java.sun.com&btnI=I'm Feeling Lucky
 [2]: http://www.google.com/search?hl=en&q=allinurl:context java.sun.com&btnI=I'm Feeling Lucky
 [3]: http://www.google.com/search?hl=en&q=allinurl:view java.sun.com&btnI=I'm Feeling Lucky

We will write a class (called ViewHolder) in which references between the widgets will be saved :

    static class ViewHolder&#123;
            TextView tvName;
            TextView tvDescription;
        &#125;

And the adapter :

    @Override
        public [View][3] getView&#40;int position, [View][3] view, ViewGroup viewGroup&#41; &#123;
    &nbsp;
            [View][3] vi = view;             //trying to reuse a recycled view
            ViewHolder holder = null;
    &nbsp;
            if &#40;vi == null&#41; &#123;
                //The view is not a recycled one: we have to inflate
                vi = mLayoutInflater.inflate&#40;R.layout.item, null&#41;;
                holder = new ViewHolder&#40;&#41;;
    &nbsp;
                holder.tvName = &#40;TextView&#41; vi.findViewById&#40;R.id.textView_item_name&#41;;
                holder.tvDescription = &#40;TextView&#41; vi.findViewById&#40;R.id.textView_item_description&#41;;
                vi.setTag&#40;holder&#41;;
            &#125; else &#123;
                // View recycled !
                // no need to inflate
                // no need to findViews by id
                holder = &#40;ViewHolder&#41; vi.getTag&#40;&#41;;
            &#125;
    &nbsp;
            MyPojo item = getItem&#40;position&#41;;
    &nbsp;
            holder.tvName.setText&#40;item.getName&#40;&#41;&#41;;
            holder.tvDescription.setText&#40;item.getDescription&#40;&#41;&#41;;
    &nbsp;
            return vi;
        &#125;

If you want to add other widgets in your elements (an image for example), you will have to add them in the ViewHolder too.

That’s all I’ll talk next time about the way you can optimize your images in your lists (LRUCache, etc.).