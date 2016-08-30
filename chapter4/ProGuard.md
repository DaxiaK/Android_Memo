# ProGuard

##What is ProGuard
[proguard](http://proguard.sourceforge.net/) 

ProGuard is a Java class file shrinker, optimizer, obfuscator, and preverifier. The shrinking step detects and removes unused classes, fields, methods, and attributes. The optimization step analyzes and optimizes the bytecode of the methods. The obfuscation step renames the remaining classes, fields, and methods using short meaningless names. These first steps make the code base smaller, more efficient, and harder to reverse-engineer. The final preverification step adds preverification information to the classes, which is required for Java Micro Edition or which improves the start-up time for Java 6.

Each of these steps is optional. For instance, ProGuard can also be used to just list dead code in an application, or to preverify class files for efficient use in Java 6.

**簡單來說 - ProGuard是一個用來優化與混淆程式碼的工具**

需要ProGuard是因為Android是基於Java來作為開發，Java本身很容易的被反編譯而導致程式碼外流，增加資安的疑慮，
同時ProGuard也會移除不用的程式碼，讓程式更輕巧，且提升效能。

可以參考最近很火紅的寶可夢Go輕易的被逆向了：
 [Unbundling Pokémon Go](https://applidium.com/en/news/unbundling_pokemon_go/)

##How to use
[Android Official](https://developer.android.com/studio/build/shrink-code.html)

1. 確認不需要餛淆的內容
2. 設定gradle , rule
3. Build Source 
4. 保存輸出文件

##Customize which code to keep
ProGuard會將程式碼轉換成簡單且不易閱讀的名稱，如果我們不先確定好哪些內容不能被餛淆到，會發生ClassNotFoundException的問題
不能被餛淆的程式碼有：

1. When your app references a class only from the AndroidManifest.xml file
2. When your app calls a method from the Java Native Interface (JNI)
3. When your app manipulates code at runtime (such as with reflection or introspection)

第一類是常見的Activity,Service等等定義在AndroidManifest的東西，第二類JNI比較沒什麼好說。
實務上最長遇到的多是處於第三類，其中包含：

* Signature / generics（泛型）
* Parcelable & Serializable 
* Java Invoke
* R file
* Custom view
* enum

## proguard-android.txt & proguard-rules.pro
看到上述那麼多種類，先別擔心，其實大部分的東西在Android中有一份`proguard-android.txt`都處理好了，具體位置在 `<sdk-root>/tools/proguard/ `，我們只需針對額外的需求，在`gradle - module`底下設定好`proguard-rules.pro`即可。
當然，你也可以再程式碼加入 `@Keep` 來手動指定要保留的Code。

下面是使用Gson後，因涉及到Invoke的內容，所以將相關的rule定義好的範本 ：

```
##---------------Begin: proguard configuration for Gson  ----------
# Gson uses generic type information stored in a class file when working with fields. Proguard
# removes such information by default, so configure it to keep all of it.
-keepattributes Signature

# For using GSON @Expose annotation
-keepattributes *Annotation*

# Gson specific classes
-keep class sun.misc.Unsafe { *; }
#-keep class com.google.gson.stream.** { *; }

# Application classes that will be serialized/deserialized over Gson
-keep class tw.com.hellowrold.bean.** { *; }

##---------------End: proguard configuration for Gson  ----------
```

設定好`proguard-rules.pro`後，記得要再 `gradle - module - build.gradle` 加上

```
android {
    ...
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                   'proguard-rules.pro'
        }
    }
}
```

## Output
上述的東西都設定好了後，接下來每次build出Release的APK時 ，在`<module-name>/build/outputs/mapping/release/`底下會自動產生幾個重要的文件，這些文件主要使用來記錄與ProGuard有關的資訊：

* dump.txt
Describes the internal structure of all the class files in the APK.

* mapping.txt
Provides a translation between the original and obfuscated class, method, and field names.

* seeds.txt
Lists the classes and members that were not obfuscated.

* usage.txt
Lists the code that was removed from the APK.

要注意的是，每次產生的文件都不一定會一樣，請記得保存好這些文件。

##Trace
如果你有使用或自己加入一些log的Tracing 機制 (Ex.[ACRA](https://github.com/ACRA/acra) )，那餛淆過後的APK產生的log將會難以幫助你來閱讀並追蹤問題所在。
這時可以透過retrace + mapping.txt 來還原原來的樣子：

`retrace.bat|retrace.sh [-verbose] mapping.txt [<stacktrace_file>]`

For example:

`retrace.bat -verbose mapping.txt obfuscated_trace.txt`

該工具位於 `<sdk-root>/tools/proguard/bin`。