# EPIPE (Broken pipe)


## What is EPIPE?

在網路連線的時候，如果socket connection處在關閉的情況，卻還是執行讀寫，就會發生EPIPE！
舉來來說，當我們上傳檔案的時候，socket connection是關閉或異常的情況下，我們仍然執行OutputStream.write()來做寫入檔案的動作，就會遇到此問題！

此問題不見得完全會是Code的問題，有可能Server或Client異常。
這次遇到的case主要就是因為手上的裝置送Tcp的順序跟人家不一樣產生的.....


## Solution

網路上的解法大多分為兩種：

1. 關掉keep-alive 
2. 設定Content-Length

第一種方法的來源是[這裡](http://stackoverflow.com/questions/15870631/epipe-broken-pipe-while-uploading) ！

```
conn.setRequestProperty("connection", "close");
```


在Java的HttpURLConnection預設是執行Keep-alive的，如果沒有關掉的話，HttpURLConnection 有可能會從pool中去找尋可以重複利用的connection。
其實Keep-alive本身是不會對request有影響，但因為他對於連上的保證上並不可靠，所以當server或者client端有狀況時，會導致意料之外的錯誤!

**如果真的在可重現的情況下發生了EPIPE，建議可以將所有request的keep-alive先關掉來驗證問題!**

第二種方法則是Android API < 19 的情況下，上傳超過2GB的檔案會有問題，要先告知Content-Length 才不會發生問題，不過因為沒實際測試過，故在此不多提。


## Case

一隻Android 的APP ，在上傳資料前會先透過post API來取得一些資料，接下來在透過multipart上傳一些檔案。
當第一次執行的時候，在特定的裝置上每次總會遇到EPIPE的錯誤。

post的request為預設開啟Keep-alive,　multipart 的 request為關閉Keep-alive。

在此case中，我的裝置中的HttpURLConnection發送Tcp的順序異常!
一般裝置的HttpURLConnection在執行有keep-alive的request後，connection可能被丟入pool中等待下次的調用
而當沒有keep-alive的request調用到已經存在的connection時，會先將資料都送出去，才送fin正式結束這個connection。

但你大爺的...
我手上的裝置硬是先送了fin出才，才開始丟資料，你說這資料能丟的出去嘛？
故最終解決的方法，便是將所有的keep-alive強制關掉，就不會確保每次的connection都是新的，來避免這種白痴狀況...

圖中淡藍色框框便是偷跑的fin:
![](/assets/epipe.jpeg)

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

我看到網路上比較好理解的說明是，可以將HttpURLConnection製造的request想成是一個想問的問題，而socket connection是電話！我們可以播通電話後(connection)，問對方一個問題(request)，接下來對方會回應(respones)，然後掛斷電話。當然我們也可以在一通電話裡面先問對方問題後，等待對方回答完，接下來我們繼續問下一個問題，並等待對方回答，所以這樣子就會變成一通電話(connection)有多個問題(request)。

上面的例子結合Keep-Alive後，就是我們問完問題(request)後，告訴對方我現在沒問題了(call disconnect())，此時這通電話有可能會被掛斷，也有可能就放著，當我下一次忽然想到問題後，如果發現電話還沒掛掉，我可以不用浪費時間在撥號碼等待對方接電話(e.g.  Three Way Handshake  in TCP )，我可以直接就問下一個問題(request)了。


反之，沒有Keep-Alive，我每一次想問問題(request)，我都要撥打一通新的電話(connection)


## 問題集

**1. Keep-alive會導致multipart/formdata失效嗎?**

 因網路上沒找到任何的討論，且經過測試後，Keep-alive不會直接導致multipart失效！

**2. socket connection 在Keep-Alive的狀況下何時才會真正的被中止 ?**
HttpURLConnection屬於比較上層的Object , 本身會自己去管理內部的pool,原則上我們沒辦法控制何時去關閉。

## 參考資料
1. [https://www.javaworld.com.tw/jute/post/view?bid=5&id=269709](https://www.javaworld.com.tw/jute/post/view?bid=5&id=269709) 
2. [http://docs.oracle.com/javase/7/docs/technotes/guides/net/http-keepalive.html](http://docs.oracle.com/javase/7/docs/technotes/guides/net/http-keepalive.html) 
3. [http://stackoverflow.com/questions/11056088/do-i-need-to-call-httpurlconnection-disconnect-after-finish-using-it](http://stackoverflow.com/questions/11056088/do-i-need-to-call-httpurlconnection-disconnect-after-finish-using-it) 
4. [https://developer.android.com/reference/java/net/HttpURLConnection.html](https://developer.android.com/reference/java/net/HttpURLConnection.html) 
5. [https://zh.wikipedia.org/wiki/HTTP%E6%8C%81%E4%B9%85%E8%BF%9E%E6%8E%A5](https://zh.wikipedia.org/wiki/HTTP%E6%8C%81%E4%B9%85%E8%BF%9E%E6%8E%A5) 
6. [http://docs.oracle.com/javase/6/docs/api/java/net/HttpURLConnection.html](http://docs.oracle.com/javase/6/docs/api/java/net/HttpURLConnection.html) 