# EPIPE (Broken pipe)


## What is EPIPE?

在網路連線的時候，如果socket connection處在關閉的情況，卻還是執行讀寫，就會發生EPIPE！
舉來來說，當我們上傳檔案的時候，socket connection是關閉或異常的情況下，我們仍然執行OutputStream.write()來做寫入檔案的動作，就會遇到此問題！

此問題不見得完全會是Code的問題，有可能Server異常，或是Keep Alive造成。


## Solution

網路上的解法大多分為兩種：

1. 關掉keep-alive 
2. 設定Content-Length

第一種方法主要是因為，在Java的HttpURLConnection預設是執行Keep-alive的，如果沒有關掉的話，HttpURLConnection 有可能會從pool中去找尋可以重複利用的connection，最後導致上傳出錯。

可以參考[這裡](http://stackoverflow.com/questions/15870631/epipe-broken-pipe-while-uploading) 

```
conn.setRequestProperty("connection", "close");
```

第二種方法則是Android API < 19 的情況下，上傳超過2GB的檔案會有問題，要先告知Content-Length 才不會發生問題，不過因為沒實際測試過，故在此不多提。


## Keep-alive

Wiki:

```
HTTP持久連線（HTTP persistent connection，也稱作HTTP keep-alive或HTTP connection reuse）是使用同一個TCP連線來傳送和接收多個HTTP請  / 應答，而不是為每一個新的請求 / 應答開啟新的連線的方法。
```

![](/assets/HTTP_persistent_connection.svg)

## HttpURLConnection

在Java(Android)中的HttpURLConnection，有幾點注意幾點：

1. HttpURLConnection 與其使用的 socket connection 不是一對一的關係
2.  取得respone時用的getInputStream() 會在全部讀完，或是調用close後中止，故最好在捕捉到IOException後將其getErrorStream讀完
3.  在預設的Keep-alive下，Disconnect在意義上只代表把connection佔住資源釋放，connection接下來可能會被關閉，也有可能會將connection送回connected sockets的pool中，等待下次重新被呼叫使用
4. Keep-alive下，HttpURLConnection在可以的情況下，會儘量的使用pool中的connection


## HttpURLConnection vs socket connection

上面提到的HttpURLConnection與 socket connection不是一對一的關係，我們可以看看HttpURLConnection的說明:

```
Each HttpURLConnection instance is used to make a single request but the underlying network connection to the HTTP server may be transparently shared by other instances
```

我看到網路上比較好理解的說明是，可以將HttpURLConnection製造的request想成是一個想問的問題，而socket connection是電話！我們可以播通電話後，問對方一個問題，接下來對方會回應，然後掛斷電話。當然我們也可以在一通電話裡面先問對方問題後，等待對方回答完，接下來我們繼續問下一個問題，並等待對方回答，所以這樣子就會變成一通電話(socket connection)有多個問題(request)。

上面的例子再延伸成Keep-Alive後，就是我們問完問題後，告訴對方我現在沒問題了(call disconnect())，此時這通電話有可能會被掛斷，也有可能就放著，當我下一次忽然想到問題後，如果發現電話還沒掛掉，我可以不用浪費時間在撥號碼等待對方接電話(E.g  Three Way Handshake  in TCP )，我可以直接就問下一個問題(request)了。


反之，沒有Keep-Alive，我每一次想問問題(request)，我都要撥打一通新的電話(socket connection)


## 待釐清問題

**1. Keep-alive為何會導致multipart/formdata失效**

**2. socket connection 在Keep-Alive何時才會真正的被中止**


## 結論

總而言之，如果發生  EPIPE (Broken pipe) 時，除了注意upload 檔案的connection本身有沒有關掉keep-alive 外，也可以先用一些追蹤connection的軟體確定是否有其他的connection在connected sockets的pool中，否則也有影響到上傳的connection.

## 參考資料
1. [https://www.javaworld.com.tw/jute/post/view?bid=5&id=269709](https://www.javaworld.com.tw/jute/post/view?bid=5&id=269709) 
2. [http://docs.oracle.com/javase/7/docs/technotes/guides/net/http-keepalive.html](http://docs.oracle.com/javase/7/docs/technotes/guides/net/http-keepalive.html) 
3. [http://stackoverflow.com/questions/11056088/do-i-need-to-call-httpurlconnection-disconnect-after-finish-using-it](http://stackoverflow.com/questions/11056088/do-i-need-to-call-httpurlconnection-disconnect-after-finish-using-it) 
4. [https://developer.android.com/reference/java/net/HttpURLConnection.html](https://developer.android.com/reference/java/net/HttpURLConnection.html) 
5. [https://zh.wikipedia.org/wiki/HTTP%E6%8C%81%E4%B9%85%E8%BF%9E%E6%8E%A5](https://zh.wikipedia.org/wiki/HTTP%E6%8C%81%E4%B9%85%E8%BF%9E%E6%8E%A5) 
6. [http://docs.oracle.com/javase/6/docs/api/java/net/HttpURLConnection.html](http://docs.oracle.com/javase/6/docs/api/java/net/HttpURLConnection.html) 