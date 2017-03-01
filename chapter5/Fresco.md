# Fresco

##What is Fresco
一個由Facebook開發，用來簡化處理圖片的開源專案。
該專案運行於Android 2.3+ ，其開源位至於：

[https://github.com/facebook/fresco](https://github.com/facebook/fresco) 
[http://frescolib.org](http://frescolib.org) 


##Concepts
Fresco 採用類似MVC的三層模式來處理圖片，分別是DraweeView (View) 、DraweeHierarchy(Model)和DraweeController(Controller)。
大部分的情況下建議儘量使用SimpleDraweeView來完成處理圖片需求，因DraweeController默認使用了 image pipeline來管理記憶體，而SimpleDraweeView會自動完成相關的控制過程。

如果對於圖片的控制有額外的需求，DraweeController可以來做一些自定義的設定。
DraweeControllerBuilder 可以用來產生不能修改的DraweeController。


Fresco不支援相對路徑，餵給它的都必須是帶有scheme的絕對路徑：

![](/assets/fresco_uri.jpeg) 


##  ImageRequest


要更進階的使用DraweeController，必須使用ImageRequest來處理圖片，下面是使用Postprocessor(修改圖片的method)的例子
```
Uri uri;
Postprocessor myPostprocessor = new Postprocessor() { ... }
ImageRequest request = ImageRequestBuilder.newBuilderWithSource(uri)
    .setPostprocessor(myPostprocessor)
    .build();

DraweeController controller = Fresco.newDraweeControllerBuilder()
    .setImageRequest(request)
    .setOldController(mSimpleDraweeView.getController())
    // other setters as you need
    .build();
``` 


## Resizing and Rotating
Fersco提供三種resize方式來減少記憶體的使用：

1. **Scaling** is a canvas operation and is usually hardware accelerated. The bitmap itself is always the same size. It just gets drawn downscaled or upscaled.

2. **Resizing** is a pipeline operation executed in software. This changes the encoded image in memory before it is being decoded. The decoded bitmap will be smaller than the original image.

3. **Downsampling** is also a pipeline operation implemented in software. Rather than creating a new encoded image, it simply decodes only a subset of the pixels, resulting in a smaller output bitmap.

簡單來說，如果圖片大小變化不大的話，可以使用Scaling，如此一來不必浪費記憶體來多產生一張bitmap，而其他情況則建議使用Resizing。(例如使用相機拍照，返回的大小一定超出當前的view許多) 


大很多的定義是：
圖片的總像素 > view的大小 x 2 (in total number of pixels, i.e. width*height)


Resize同時還有一些限制：
1. 只能是JPEG
2. 大小調整只能是 N / 8，1 <= N <= 8
3. 圖片只能變小，無法變大

Downsampling則還是實驗性的功能，在此暫時不提。


**Auto-rotation**
Fersco提供圖片自動旋轉的機制來簡化圖片擺放的問題：

```
ImageRequest request = ImageRequestBuilder.newBuilderWithSource(uri)
    .setRotationOptions(RotationOptions.autoRotate())
    .build();
```


## Sample

Application:
```
public class MyApplication extends Application {
	@Override
	public void onCreate() {
		super.onCreate();
		Fresco.initialize(this);
	}
}
```


Activity:
```
    private SimpleDraweeView SDV_new_photo;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
    ...
    ..
    //Get view
    SDV_new_photo = (SimpleDraweeView) v.findViewById(R.id.SDV_new_photo);
    }
    
     public void setPhotoUri( Uri photoUri) {
      //產生ImageRequest
        ImageRequest request = ImageRequestBuilder.newBuilderWithSource(photoUri)
                .setResizeOptions(new ResizeOptions(DiaryItemHelper.getVisibleWidth(getContext()),
                        DiaryItemHelper.getVisibleHeight(getContext())))
                .setRotationOptions(RotationOptions.autoRotate())
                .build();
         //產生DraweeController
        DraweeController controller = Fresco.newDraweeControllerBuilder()
                .setImageRequest(request)
                .build();
        SDV_new_photo.setController(controller);
    }

```

## Notice

我在實作的時候遇到了幾個比較容易遇到的問題：


1. SimpleDraweeView不支援WRAP_CONTENT。如果真的非要要使用WRAP_CONTENT，最多只能有有一個屬性是WRAP_CONTENT，且你必須使用setAspectRatio(float aspectRatio) 來指定該圖片的比例。 因此採用固定大小的圖片設計是最適合使用Fresco的情境。
2. 指定內容大小後，如果SimpleDraweeView的長寬被調整，有可能會產生切到圖片的狀況。
3. 使用ScrollViews時，因其特性在activity或fragment的生命周期結束前 Fresco views不會知道什麼時候要回收view，需要注意放太多會導致OOM的風險，建議採用 RecyclerView, ListView 或 GridView等。
4. 不要向下轉型。
5. 雖然DraweeView繼承自ImageView，但不要使用setImageBitmap, setImageDrawable的方法來設置圖片，否則會造成DraweeHierarchy的遺失，而產生問題。 同理，也不要使用屬於ImageView但不屬於不View 的method。

更多細節和內容可以察看：[http://frescolib.org/docs/gotchas.html](http://frescolib.org/docs/gotchas.html) 
 




