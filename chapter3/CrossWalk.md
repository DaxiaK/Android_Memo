#crosswalk


##What is crosswalk?
Crosswalk 是一款基於chromium 核心的web runtime ，同時也是由Intel支持的open source project。
雖然Android本身已經存在了Webview ，但因為直到Android 每個版本的Webview都不相同，往往在維護上面會有許多問題
所以使用第三方的Webview算是一個不錯的選擇

此篇文章主要針對**Embedding  Crosswalk**來做記錄，同時也寫下一些 在Android Studio上值得注意的地方


##Embedding the Crosswalk Project
Crosswalk裡面的Webview叫做XWalkView，原則上跟原生的Webview很相識！
其實官網已經有相當完整的[教學](https://crosswalk-project.org/documentation/android/embedding_crosswalk.html)了，但如同我上面所提，有些小地方還可以再補充的更完整，所以可以邊看官網邊參考我的補充


**1. Set up by gradle** 

首先打開Android Studio，並建立一個專案，建立好後，在預設的app/build.gardle裡設定crosswalk下載的路徑：

```
repositories {
    maven {
        url 'https://download.01.org/crosswalk/releases/crosswalk/android/maven2'
    }
}
```

還有compile 的版本(可以到maven的網址查看最新的版本)：


```
 dependencies {
  ...
 compile "org.xwalk:xwalk_core_library:21.51.546.6"
 }
```

接下來sync gardle並等待下載和build，完成的build.gardle大概會長這個樣子：
![](/assets/crosswalk1.jpg) 
 
 
 **2.Add permission** 
 
 這邊原則上跟官網一樣，基本的三個權限要先加入AndroidManifest，有其他需要在視狀況自行新增：
 
```
 <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
 <uses-permission android:name="android.permission.INTERNET" />
 <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
```

 **3.開始使用** 
 
  再想要使用的Layout.xml上加入：
  
```
 <org.xwalk.core.XWalkView
        android:id="@+id/WebView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
```

接著到對應的Activity上新增Code：

```
webView = (XWalkView) findViewById(webview);
webView.load("http://www.google.com", null);
```

最後，最重要的一點是要對應Activity的LifeCycle做出相關的處理，不然會有些問題，完整的Code如下：
```
public class MainActivity extends Activity {

    XWalkView webView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        webView = (XWalkView) findViewById(webview);
        webView.load("http://www.google.com", null);
    }

    @Override
    protected void onPause() {
        super.onPause();
        if (webView != null) {
            webView.pauseTimers();
            webView.onHide();
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        if (webView != null) {
            webView.resumeTimers();
            webView.onShow();
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (webView != null) {
            webView.onDestroy();
        }
    }
}
```
##心得

對於crosswalk的一些狀況，我寫在這邊：

**多重Webview** 

我目前測試遇XWalkView到跟原生Webview最大不一樣的地方就是一個頁面裡面有多個Webview了！
其實我個人不太喜歡使用這種使用方式，因為Webview很佔資源，且會使得不同版本相容性的問題變得更加複雜，在開發上或是維護上都是不小的負擔。但有些專案總有奇特的需求...

對於XWalkView來說，比較好的使用方式是一個Activity綁定一個XWalkView，經我測試後，發現在Listview中嵌入多個XWalkView的確是會發生顯示不出來或者layout異常的狀況!網路上也有相關的討論，參考[這邊](https://crosswalk-project.org/jira/browse/XWALK-3545) 

**wrap_content** 

我在測試的時候發現XWalkView使用wrap_content時會被強迫給予一個固定的高度，我目前是沒有找到解決的方法。
其實原生的Webview就最好不要使用這種方法，不只是因為效能差，容易出狀況，這樣的設計滿糟糕的外，最重要的是原生的Webview有使用wrap_content後Resize異常的[Bug](https://code.google.com/p/android/issues/detail?id=18726&can=1&q=wrap_content%20webview&colspec=ID%20Status%20Priority%20Owner%20Summary%20Stars%20Reporter%20Opened)
與此相比，我想這可能也是為什麼XWalkView的wrap_content會無法作用的原因所在吧！

**64K Methods**

在我開發Android 中的經驗，目前用過最肥的Libary，除了包入全部的Google app外，就屬這crosswalk最肥！
只要開發的專案稍微複雜或者又與其他的Libary一起用，很容易會遇到Android的64K methods上限。
當你照上面的教學build完之後，卻發現有類似下方的Error Message，那恭喜你，你就是遇到了這個問題：
```
Error:Execution failed for task ':app:dexDebug'.
> com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command '/home/xxx/android-studio/jre/bin/java'' finished with non-zero exit value 2
```

在早期這個問題的確是個麻煩的問題，但現在Android官方其實已經有方便的解決方式[解決方式](https://developer.android.com/studio/build/multidex.html)了。
經過測試確實也可以順利的執行，不必太擔心！


##參考資料

[https://diego.org/2015/01/07/embedding-crosswalk-in-android-studio/](https://diego.org/2015/01/07/embedding-crosswalk-in-android-studio/)

[https://crosswalk-project.org/documentation/getting_started.html](https://crosswalk-project.org/documentation/getting_started.html)

