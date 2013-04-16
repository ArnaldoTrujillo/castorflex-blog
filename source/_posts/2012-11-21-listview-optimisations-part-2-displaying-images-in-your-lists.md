---
title: 'ListView optimisations : Part 2 (Displaying images in your lists)'
author: Antoine Merle
layout: post
comments: true
permalink: /listview-optimisations-part-2-displaying-images-in-your-lists/
description: "This second post about listview optimisations will talk about the images in your lists. This post is an extension of the first, which one was about the viewholder."
keywords: "android, listview, image, images, custom adapter, adapter, imageview, smooth, optimization"
categories:
  - Listview
---
# 

This second post about listview optimisations will talk about the images in your lists. This post is an extension of the first, which one was about the viewholder :

*   [ListView opimisation : Part 1 (the ViewHolder)][1] : The viewholder is essential when you make a listView. See this post first.

 [1]: http://antoine-merle.com/listview-optimisations-part-1-the-viewholder/ "ListView optimisations : Part 1 (the ViewHolder)"

<!-- more -->
## I.    The ImageLoader

We will create a class which will do all the job (here is the skeleton):

```java
public class ImageLoader {
	public ImageLoader(Context context) {
	}
	public void displayImage(String url, ImageView imageView) {     
	}
}
```
And how to use it in your adapter:

```java
public class MyAdapter extends BaseAdapter {
     
        private LayoutInflater mLayoutInflater;
        private List mData;
        private ImageLoader mImageLoader;
     
        public MyAdapter(Context context, List data){
            mData = data;
            mLayoutInflater = LayoutInflater.from(context);
            mImageLoader = new ImageLoader(context);
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
        public View getView(int position, View view, ViewGroup viewGroup) {
     
            View vi = view;             
            ViewHolder holder = null;
     
            if (vi == null) {
                vi = mLayoutInflater.inflate(R.layout.item, null);
                holder = new ViewHolder();
     
                holder.ivIcon = (ImageView) vi.findViewById(R.id.imageView_item_icon);
                vi.setTag(holder);
            } else {
                holder = (ViewHolder) vi.getTag();
            }
     
            MyPojo item = getItem(position);
     
            mImageLoader.displayImage(item.getIcon(), holder.icon);
     
            return vi;
        }
     
        static class ViewHolder{
            ImageView ivIcon;
        }
    }
```

A simple picture to understand the role of the ImageLoader:  
{% img center /images/imageloader.png %}  
And his code:


```java
public class ImageLoader {
     
        private LruCache memoryCache;
        private FileCache fileCache;
        private Map imageViews = Collections.synchronizedMap(new WeakHashMap());
     
        private Drawable mStubDrawable;
     
        public ImageLoader(Context context) {
            fileCache = new FileCache(context);
            init(context);
        }
     
        private void init(Context context) {
            // Get memory class of this device, exceeding this amount will throw an
            // OutOfMemory exception.
            final int memClass = ((ActivityManager) context.getSystemService(
                    Context.ACTIVITY_SERVICE)).getMemoryClass();
            // 1/8 of the available mem
            final int cacheSize = 1024 * 1024 * memClass / 8;
            memoryCache = new LruCache(cacheSize);
     
            mStubDrawable = context.getResources().getDrawable(R.drawable.default_icon);
        }
     
        public void displayImage(String url, ImageView imageView) {
            imageViews.put(imageView, url);
            Bitmap bitmap = null;
            if (url != null && url.length() > )
                bitmap = (Bitmap) memoryCache.get(url);
            if (bitmap != null) {
                //the image is in the LRU Cache, we can use it directly
                imageView.setImageBitmap(bitmap);
            } else {
                //the image is not in the LRU Cache
                //set a default drawable a search the image
                imageView.setImageDrawable(mStubDrawable);
                if (url != null && url.length() > )
                    queuePhoto(url, imageView);
            }
        }
     
        private void queuePhoto(String url, ImageView imageView) {
                new LoadBitmapTask().execute(url, imageView);
        }
     
        /**
         * Search for the image in the device, then in the web
         * @param url
         * @return
         */
        private Bitmap getBitmap(String url) {
            Bitmap ret = null;
            //from SD cache
            File f = fileCache.getFile(url);
            if (f.exists()) {
                ret = decodeFile(f);
                if (ret != null)
                    return ret;
            }
     
            //from web
            try {
                //your requester will fetch the bitmap from the web and store it in the phone using the fileCache
                ret = MyRequester.getBitmapFromWebAndStoreItInThePhone(url);    // your own requester here
                return ret;
            } catch ([Exception][8] ex) {
                ex.printStackTrace();
                return null;
            }
        }
     
        //decodes image and scales it to reduce memory consumption
        private Bitmap decodeFile(File f) {
            Bitmap ret = null;
            try {
                [FileInputStream][9] is = new [FileInputStream][9](f);
                ret = BitmapFactory.decodeStream(is, null, null);
            } catch ([FileNotFoundException][10] e) {
                e.printStackTrace();
            } catch ([Exception][8] e) {
                e.printStackTrace();
            }
            return ret;
        }
     
        private class PhotoToLoad {
            public String url;
            public ImageView imageView;
     
            public PhotoToLoad(String u, ImageView i) {
                url = u;
                imageView = i;
            }
        }
     
        private boolean imageViewReused(PhotoToLoad photoToLoad) {
            //tag used here
            String tag = imageViews.get(photoToLoad.imageView);
            if (tag == null || !tag.equals(photoToLoad.url))
                return true;
            return false;
        }
     
        class LoadBitmapTask extends AsyncTask {
            private PhotoToLoad mPhoto;
     
            @Override
            protected TransitionDrawable doInBackground([Object][11]... params) {
                mPhoto = new PhotoToLoad((String) params[], (ImageView) params[1]);
     
                if (imageViewReused(mPhoto))
                    return null;
                Bitmap bmp = getBitmap(mPhoto.url);
                if (bmp == null)
                    return null;
                memoryCache.put(mPhoto.url, bmp);
     
                // TransitionDrawable let you to make a crossfade animation between 2 drawables
                // It increase the sensation of smoothness
                TransitionDrawable td = null;
                if (bmp != null) {
                    Drawable[] drawables = new Drawable[2];
                    drawables[] = mStubDrawable;
                    drawables[1] = new BitmapDrawable(Application.getRessources(), bmp);
                    td = new TransitionDrawable(drawables);
                    td.setCrossFadeEnabled(true); //important if you have transparent bitmaps
                }
     
                return td;
            }
     
            @Override
            protected void onPostExecute(TransitionDrawable td) {
                if (imageViewReused(mPhoto)) {
                    //imageview reused, just return
                    return;
                }
                if (td != null) {
                    // bitmap found, display it !
                    mPhoto.imageView.setImageBitmap(drawable);
                    mPhoto.imageView.setVisibility(View.VISIBLE);
     
                    //a little crossfade
                    td.startTransition(200);
                } else {
                    //bitmap not found, display the default drawable
                    mPhoto.imageView.setImageDrawable(mStubDrawable);
                }
            }
        }
    }
```

FileCache:

```java
    public class FileCache {
        private File cacheDir;
     
        public FileCache(Context context) {
            this(context, );
        }
     
        public FileCache(Context context, long evt) {
            //Find the dir to save cached images
            cacheDir = context.getCacheDir();
            if (!cacheDir.exists())
                cacheDir.mkdirs();
        }
     
        public File getFile(String url) {
            return new File(cacheDir, String.valueOf(url.hashCode()));
        }
     
        public void clear() {
            File[] files = cacheDir.listFiles();
            if (files == null)
                return;
            for (File f : files)
                f.delete();
        }
    }
```
***Note 1***: *The crossfade is done with a [TransitionDrawable][12], which give the feeling that your list scroll smoother!*

 [12]: http://developer.android.com/reference/android/graphics/drawable/TransitionDrawable.html

***Note 2***: *Try to avoid the setImageResource(ind resId) as much as possible and create a default drawable when you init your imageLoader. Indeed, according to the [DOC][13],*

 [13]: http://developer.android.com/reference/android/widget/ImageView.html#setImageResource(int)

>This does Bitmap reading and decoding on the UI thread, which can cause a latency hiccup. If that's a concern, consider using <code>setImageDrawable(Drawable)</code> or <code>setImageBitmap(Bitmap)</code> and <code>BitmapFactory</code> instead.


***Note 3*** : *We use the fileCache here to get bitmaps, but we store the bitmaps thanks to our requester. You can do this very easily using an InputStream and an OutputStream*

## II. The BlockingImageView

When you scroll now your list, you can notice some lags when images are displayed. This is due to the setImageDrawable(Drawable d) wich can call a requestLayout (which one can take some time).

There is a way to block the requestLayout, pretty simple. I found it thanks to Jorim Jaggy, [on this google post][14]. But look at the source code of the ImageView first:

 [14]: https://plus.google.com/113058165720861374515/posts/iTk4PjgeAWX

```java
    /**
         * Sets a drawable as the content of this ImageView.
         * 
         * @param drawable The drawable to set
         */
        public void setImageDrawable(Drawable drawable) {
            if (mDrawable != drawable) {
                mResource = ;
                mUri = null;
     
                int oldWidth = mDrawableWidth;
                int oldHeight = mDrawableHeight;
     
                updateDrawable(drawable);
     
                if (oldWidth != mDrawableWidth || oldHeight != mDrawableHeight) {
                    requestLayout();    //HERE
                }
                invalidate();
            }
        }
```
If you know that the dimension of all your images is the same, then you can block the <code>requestlayout</code>, by using a custom <code>ImageView</code> :

```java
    public class BlockingImageView extends ImageView {
        private boolean mBlockLayout;
     
        public BlockingImageView(Context context) {
            super(context);
        }
     
        public BlockingImageView(Context context, AttributeSet attrs) {
            super(context, attrs);
        }
     
        public BlockingImageView(Context context, AttributeSet attrs, int defStyle) {
            super(context, attrs, defStyle);
        }
     
        @Override
        public void requestLayout() {
            if (!mBlockLayout) {
                super.requestLayout();
            }
        }
     
        @Override
        public void setImageResource(int resId) {
            mBlockLayout = true;
            super.setImageResource(resId);
            mBlockLayout = false;
        }
     
        @Override
        public void setImageURI(Uri uri) {
            mBlockLayout = true;
            super.setImageURI(uri);
            mBlockLayout = false;
        }
     
        @Override
        public void setImageDrawable(Drawable drawable) {
            mBlockLayout = true;
            super.setImageDrawable(drawable);
            mBlockLayout = false;
        }
    }
```
Don’t forget to modify the xml of your elements to add your custom <code>ImageView</code>.

You should now have a smoothy listView with images.

Hope you enjoyed, do not hesitate to leave comments and share it!
