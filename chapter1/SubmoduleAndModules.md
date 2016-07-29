# Git Submodule 和Android Studio Modules

## 簡介

在專案中我們有時會遇到一些需要掛入Third party librays 的時候，雖然Android Studio中的gradle有提供許多種的導入方式，更甚至有方便的Remote binary dependency 可以用：

[Android Studio - Declare Dependencies ](https://developer.android.com/studio/build/build-variants.html#dependencies)

```
android {...}
...
dependencies {
// The 'compile' configuration tells Gradle to add the dependency to the 
// compilation classpath and include it in the final package.

// Dependency on the "mylibrary" module from this project compile project(":mylibrary")

// Remote binary dependency 
compile 'com.android.support:appcompat-v7:23.4.0' 

// Local binary dependency 
compile fileTree(dir: 'libs', include: ['*.jar'])}
}
```

然而許多時候我們還是需要將一份source導入自己的專案中，平時可能還好，最怕的就是開發到一半遇到了bug或者需要的功能是在新的版本上，如果要舊的資料砍掉再放入新的，一來一往之間不但git的資料會膨脹，管理上其實也不是很方便。

這個時候git的submodule搭配上Android studio的modules來管理就相當的方便了！
先來看看git submodule的 [指令](https://git-scm.com/book/zh-tw/v1/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E7%B5%84-Submodules)：

```
$git submodule [--quiet] add [-b <branch>] [-f|--force] [--name <name>] [--reference <repository>] [--] <repository> [<path>]
//For exampl
$git submodule add https://android.googlesource.com/platform/frameworks/volley app/libs/volley
```

上面的意思就是將rack這個專案當作子專案掛入 rack目錄下，接下來把新加進來的source從Android Studio中設定好Modules就完成了，這樣一來你的主程式的git就跟新加進來soruce就會分開來處理了，如此一來不用擔心source的管理會影響到你的主程式，也因為Android Studio modules的特性，讓你再開發上也更好區分哪些是你的主程式，哪些則是其他的source!

## 教學

如何把volley加到新的專案並設定好：

1. Add volley to submodule


```
~/desgin/Sample$ git submodule add https://android.googlesource.com/platform/frameworks/volley app/libs/volleyCloning into 'app/libs/volley'...remote: Counting objects: 179, doneremote: Finding sources: 100% (179/179)remote: Total 3237 (delta 302), reused 3237 (delta 302)Receiving objects: 100% (3237/3237), 1.23 MiB | 0 bytes/s, done.Resolving deltas: 100% (302/302), done.Checking connectivity... done.
```

2.Open Android Studio & Add VCS 

![](/assets/submodule1.jpeg)

3.Open Project Structure \(Default hotkey : F4\) , than add new module

![](/assets/submodule2.jpeg)

4.Import Gradle project![](/assets/submodule3.jpeg)

5.Select submodule path ![](/assets/submodule4.jpeg)

6.Click Ok and waiting gardle running!

![](/assets/submodule5.jpeg)

完成後，在git中就會看類似這樣的結構

![](/assets/submodule6.jpeg)

這樣我們就可以分開管理不同的Code也不會衝突了！

最後，如果要clone一個含有submodule的專案，記得git clone後要再使用來完成下載submodule

```
$git submodule init 
$git submodule update
```

