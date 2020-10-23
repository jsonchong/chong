### H5 缓存机制介绍

H5 应用程序缓存为应用带来三个优势：

- 离线浏览 用户可在应用离线时使用它们
- 速度 已缓存资源加载得更快
- 减少服务器负载 浏览器将只从服务器下载更新过或更改过的资源。

1. 浏览器缓存机制
2. Dom Storgage（Web Storage）存储机制
3. Web SQL Database 存储机制
4. Application Cache（AppCache）机制
5. Indexed Database （IndexedDB）
6. File System API

### 浏览器缓存机制

浏览器缓存机制是指通过 HTTP 协议头里的 **Cache-Control**（或 Expires）和 **Last-Modified**（或 Etag）等字段来控制文件缓存的机制。这应该是 WEB 中最早的缓存机制了，是在 HTTP 协议中实现的。有点不同于 **Dom Storage**、**AppCache** 等缓存机制，但本质上是一样的。可以理解为，一个是协议层实现的，一个是应用层实现的。

**Cache-Control** 用于控制文件在本地缓存有效时长。最常见的，比如服务器回包：Cache-Control:max-age=600 表示文件在本地应该缓存，且有效时长是600秒（从发出请求算起）。在接下来600秒内，如果有请求这个资源，浏览器不会发出 HTTP 请求，而是直接使用本地缓存的文件。

**Last-Modified** 是标识文件在服务器上的最新更新时间。下次请求时，如果文件缓存过期，浏览器通过 If-Modified-Since 字段带上这个时间，发送给服务器，由服务器比较时间戳来判断文件是否有修改。如果没有修改，服务器返回304告诉浏览器继续使用缓存；如果有修改，则返回200，同时返回最新的文件。

Cache-Control 通常与 Last-Modified 一起使用。**一个用于控制缓存有效时间**，一个在**缓存失效后，向服务查询是否有更新。**(缓存失效后的策略)

Cache-Control 还有一个同功能的字段：Expires。Expires 的值一个绝对的时间点，如：Expires: Thu, 10 Nov 2015 08:45:11 GMT，表示在这个时间点之前，缓存都是有效的

另外有两种特殊的情况：

- 手动刷新页面（F5)，浏览器会直接认为缓存已经过期（可能缓存还没有过期），在请求中加上字段：`Cache-Control:max-age=0`，发包向服务器查询是否有文件是否有更新。
- 强制刷新页面（Ctrl+F5)，浏览器会直接忽略本地的缓存（有缓存也会认为本地没有缓存），在请求中加上字段：`Cache-Control:no-cache`（或 `Pragma:no-cache`），发包向服务重新拉取文件。

下面是通过 Google Chrome 浏览器（用其他浏览器+抓包工具也可以）自带的开发者工具，对一个资源文件不同情况请求与回包的截图

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200925174212483.png" alt="image-20200925174212483" style="zoom:50%;" />

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200925174221610.png" alt="image-20200925174221610" style="zoom:50%;" />

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200925174229439.png" alt="image-20200925174229439" style="zoom:50%;" />

一般浏览器会将缓存记录及缓存文件存在本地 Cache 文件夹中。Android 下 App 如果使用 Webview，缓存的文件记录及文件内容会存在当前 app 的 data 目录中。

分析：Cache-Control 和 Last-Modified 一般用在 Web 的静态资源文件上，如 JS、CSS 和一些图像文件。通过设置资源文件缓存属性，对提高资源文件加载速度，节省流量很有意义，特别是移动网络环境。但问题是：缓存有效时长该如何设置？如果设置太短，就起不到缓存的使用；如果设置的太长，在资源文件有更新时，浏览器如果有缓存，则不能及时取到最新的文件。

分析发现，浏览器的缓存机制还不是非常完美的缓存机制。完美的缓存机制应该是这样的：

1. 缓存文件没更新，尽可能使用缓存，不用和服务器交互；
2. 缓存文件有更新时，第一时间能使用到新的文件；
3. 缓存的文件要保持完整性，不使用被修改过的缓存文件；
4. 缓存的容量大小要能设置或控制，缓存文件不能因为存储空间限制或过期被清除。

在实际应用中，为了解决 Cache-Control 缓存时长不好设置的问题，以及为了”消灭304“，Web前端采用的方式是：

1. 在要缓存的资源文件名中加上版本号或文件 MD5值字串，如 common.d5d02a02.js，common.v1.js，同时设置 Cache-Control:max-age=31536000，也就是一年。在一年时间内，资源文件如果本地有缓存，就会使用缓存；也就不会有304的回包。
2. 如果资源文件有修改，则更新文件内容，同时修改资源文件名，如 common.v2.js，html页面也会引用新的资源文件名。

### Dom Storage 存储机制

DOM Storage 分为 sessionStorage 和 localStorage。localStorage 对象和 sessionStorage 对象使用方法基本相同，它们的区别在于作用的范围不同。**sessionStorage 用来存储与页面相关的数据，它在页面关闭后无法使用。而 localStorage 则持久存在，在页面关闭后也可以使用。**

sessionStorage 是个全局对象，它维护着在页面会话(page session)期间有效的存储空间。只要浏览器开着，页面会话周期就会一直持续。当页面重新载入(reload)或者被恢复(restores)时，页面会话也是一直存在的。每在新标签或者新窗口中打开一个新页面，都会初始化一个新的会话。

```js
<script type="text/javascript">
 // 当页面刷新时，从sessionStorage恢复之前输入的内容
 window.onload = function(){
    if (window.sessionStorage) {
        var name = window.sessionStorage.getItem("name");
        if (name != "" || name != null){
            document.getElementById("name").value = name;
         }
     }
 };

 // 将数据保存到sessionStorage对象中
 function saveToStorage() {
    if (window.sessionStorage) {
        var name = document.getElementById("name").value;
        window.sessionStorage.setItem("name", name);
        window.location.href="session_storage.html";
     }
 }
 </script>

<form action="./session_storage.html">
    <input type="text" name="name" id="name"/>
    <input type="button" value="Save" onclick="saveToStorage()"/>
</form>

```

当浏览器被意外刷新的时候，一些临时数据应当被保存和恢复。sessionStorage 对象在处理这种情况的时候是最有用的。比如恢复我们在表单中已经填写的数据。

分析：Dom Storage 给 Web 提供了一种更录活的数据存储方式，存储空间更大（相对 Cookies)，用法也比较简单，方便存储服务器或本地的一些临时数据。

在 Android 内嵌 Webview 中，需要通过 Webview 设置接口启用 Dom Storage

```java
WebView myWebView = (WebView) findViewById(R.id.webview);
WebSettings webSettings = myWebView.getSettings();
webSettings.setDomStorageEnabled(true);
```

### Application Cache 机制

Application Cache（简称 AppCache)似乎是为支持 Web App 离线使用而开发的缓存机制。它的缓存机制类似于浏览器的缓存（Cache-Control 和 Last-Modified）机制，都是以文件为单位进行缓存，且文件有一定更新机制。但 AppCache 是对浏览器缓存机制的补充，不是替代。

```html
<!DOCTYPE html>
<html manifest="demo_html.appcache">
<body>

<script src="demo_time.js"></script>

<p id="timePara"><button onclick="getDateTime()">Get Date and Time</button></p>
<p><img src="img_logo.gif" width="336" height="69"></p>
<p>Try opening <a href="tryhtml5_html_manifest.htm" target="_blank">this page</a>, then go offline, and reload the page. The script and the image should still work.</p>

</body>
</html>

```

通过 Google Chrome 浏览器自带的工具，我们可以查看已经缓存的 AppCache

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200925175403226.png" alt="image-20200925175403226" style="zoom:50%;" />

上面截图中的缓存，就是我们刚才打开 HTML 的页面 AppCache。从截图中看，HTML 页面及 HTML 引用的 JS、GIF 图像文件都被缓存了；另外 HTML 头中 manifest 属性引用的 appcache 文件也缓存了

**AppCache 的原理有两个关键点：manifest 属性和 manifest 文件。**

HTML 在头中通过 manifest 属性引用 manifest 文件。manifest 文件，就是上面以 appcache 结尾的文件，是一个普通文件文件，列出了需要缓存的文件。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200925175658098.png" alt="image-20200925175658098" style="zoom:50%;" />

完整的 manifest 文件，如：

```
CACHE MANIFEST
# 2012-02-21 v1.0.0
/theme.css
/logo.gif
/main.js

NETWORK:
login.asp

FALLBACK:
/html/ /offline.html 

```

浏览器在首次加载 HTML 文件时，会解析 manifest 属性，并读取 manifest 文件，获取 Section：CACHE MANIFEST 下要缓存的文件列表，再对文件缓存。

AppCache 的缓存文件，与浏览器的缓存文件分开存储的，还是一份？应该是分开的。因为 AppCache 在本地也有 5MB（分 HOST）的空间限制。

AppCache 看起来是一种比较好的缓存方法，除了缓存静态资源文件外，也适合构建 Web 离线 App。在实际使用中有些需要注意的地方，有一些可以说是”坑“。

1. 要更新缓存的文件，需要更新包含它的 manifest 文件，那怕只加一个空格。常用的方法，是修改 manifest 文件注释中的版本号。如：# 2012-02-21 v1.0.0
2. 被缓存的文件，浏览器是先使用，再通过检查 manifest 文件是否有更新来更新缓存文件。这样缓存文件可能用的不是最新的版本。
3. 在更新缓存过程中，如果有一个文件更新失败，则整个更新会失败。
4. manifest 和引用它的HTML要在相同 HOST。
5. manifest 文件中的文件列表，如果是相对路径，则是相对 manifest 文件的相对路径。
6. manifest 也有可能更新出错，导致缓存文件更新失败。
7. 没有缓存的资源在已经缓存的 HTML 中不能加载，即使有网络。例如：http://appcache-demo.s3-website-us-east-1.amazonaws.com/without-network/
8. manifest 文件本身不能被缓存，且 manifest 文件的更新使用的是浏览器缓存机制。所以 manifest 文件的 Cache-Control 缓存时间不能设置太长。

根据官方文档，AppCache 已经不推荐使用了，标准也不会再支持。现在主流的浏览器都是还支持 AppCache的，以后就不太确定了。

### Indexed Database

IndexedDB 也是一种数据库的存储机制，但不同于已经不再支持的 Web SQL Database。IndexedDB 不是传统的关系数据库，可归为 NoSQL 数据库。IndexedDB 又类似于 Dom Storage 的 key-value 的存储方式，但功能更强大，且存储空间更大。

IndexedDB 提供了一组 API，可以进行数据存、取以及遍历。这些 API 都是异步的，操作的结果都是在回调中返回。

```js
<script type="text/javascript">

var db;

window.indexedDB = window.indexedDB || window.mozIndexedDB || window.webkitIndexedDB || window.msIndexedDB;

//浏览器是否支持IndexedDB
if (window.indexedDB) {
   //打开数据库，如果没有，则创建
   var openRequest = window.indexedDB.open("people_db", 1);

   //DB版本设置或升级时回调
   openRequest.onupgradeneeded = function(e) {
       console.log("Upgrading...");

       var thisDB = e.target.result;
       if(!thisDB.objectStoreNames.contains("people")) {
           console.log("Create Object Store: people.");

           //创建存储对象，类似于关系数据库的表
           thisDB.createObjectStore("people", { autoIncrement:true });

          //创建存储对象， 还创建索引
          //var objectStore = thisDB.createObjectStore("people",{ autoIncrement:true });
         // //first arg is name of index, second is the path (col);
        //objectStore.createIndex("name","name", {unique:false});
       //objectStore.createIndex("email","email", {unique:true});
     }
}

//DB成功打开回调
openRequest.onsuccess = function(e) {
    console.log("Success!");

    //保存全局的数据库对象，后面会用到
    db = e.target.result;

   //绑定按钮点击事件
     document.querySelector("#addButton").addEventListener("click", addPerson, false);

    document.querySelector("#getButton").addEventListener("click", getPerson, false);

    document.querySelector("#getAllButton").addEventListener("click", getPeople, false);

    document.querySelector("#getByName").addEventListener("click", getPeopleByNameIndex1, false);
}

  //DB打开失败回调
  openRequest.onerror = function(e) {
      console.log("Error");
      console.dir(e);
   }

}else{
    alert('Sorry! Your browser doesn\'t support the IndexedDB.');
}

//添加一条记录
function addPerson(e) {
    var name = document.querySelector("#name").value;
    var email = document.querySelector("#email").value;

    console.log("About to add "+name+"/"+email);

    var transaction = db.transaction(["people"],"readwrite");
var store = transaction.objectStore("people");

   //Define a person
   var person = {
       name:name,
       email:email,
       created:new Date()
   }

   //Perform the add
   var request = store.add(person);
   //var request = store.put(person, 2);

   request.onerror = function(e) {
       console.log("Error",e.target.error.name);
       //some type of error handler
   }

   request.onsuccess = function(e) {
      console.log("Woot! Did it.");
   }
}

//通过KEY查询记录
function getPerson(e) {
    var key = document.querySelector("#key").value;
    if(key === "" || isNaN(key)) return;

    var transaction = db.transaction(["people"],"readonly");
    var store = transaction.objectStore("people");

    var request = store.get(Number(key));

    request.onsuccess = function(e) {
        var result = e.target.result;
        console.dir(result);
        if(result) {
           var s = "<p><h2>Key "+key+"</h2></p>";
           for(var field in result) {
               s+= field+"="+result[field]+"<br/>";
           }
           document.querySelector("#status").innerHTML = s;
         } else {
            document.querySelector("#status").innerHTML = "<h2>No match!</h2>";
         }
     }
}

//获取所有记录
function getPeople(e) {

    var s = "";

     db.transaction(["people"], "readonly").objectStore("people").openCursor().onsuccess = function(e) {
        var cursor = e.target.result;
        if(cursor) {
            s += "<p><h2>Key "+cursor.key+"</h2></p>";
            for(var field in cursor.value) {
                s+= field+"="+cursor.value[field]+"<br/>";
            }
            s+="</p>";
            cursor.continue();
         }
         document.querySelector("#status2").innerHTML = s;
     }
}

//通过索引查询记录
function getPeopleByNameIndex(e)
{
    var name = document.querySelector("#name1").value;

    var transaction = db.transaction(["people"],"readonly");
    var store = transaction.objectStore("people");
    var index = store.index("name");

    //name is some value
    var request = index.get(name);

    request.onsuccess = function(e) {
       var result = e.target.result;
       if(result) {
           var s = "<p><h2>Name "+name+"</h2><p>";
           for(var field in result) {
               s+= field+"="+result[field]+"<br/>";
           }
           s+="</p>";
    } else {
        document.querySelector("#status3").innerHTML = "<h2>No match!</h2>";
     }
   }
}

//通过索引查询记录
function getPeopleByNameIndex1(e)
{
    var s = "";

    var name = document.querySelector("#name1").value;

    var transaction = db.transaction(["people"],"readonly");
    var store = transaction.objectStore("people");
    var index = store.index("name");

    //name is some value
    index.openCursor().onsuccess = function(e) {
        var cursor = e.target.result;
        if(cursor) {
            s += "<p><h2>Key "+cursor.key+"</h2></p>";
            for(var field in cursor.value) {
                s+= field+"="+cursor.value[field]+"<br/>";
            }
            s+="</p>";
            cursor.continue();
         }
         document.querySelector("#status3").innerHTML = s;
     }
}

</script>

<p>添加数据<br/>
<input type="text" id="name" placeholder="Name"><br/>
<input type="email" id="email" placeholder="Email"><br/>
<button id="addButton">Add Data</button>
</p>

<p>根据Key查询数据<br/>
<input type="text" id="key" placeholder="Key"><br/>
<button id="getButton">Get Data</button>
</p>
<div id="status" name="status"></div>

<p>获取所有数据<br/>
<button id="getAllButton">Get EveryOne</button>
</p>
<div id="status2" name="status2"></div>

<p>根据索引:Name查询数据<br/>
    <input type="text" id="name1" placeholder="Name"><br/>
    <button id="getByName">Get ByName</button>
</p>
<div id="status3" name="status3"></div>

```

将上面的代码复制到 indexed_db.html 中，用 Google Chrome 浏览器打开，就可以添加、查询数据。在 Chrome 的开发者工具中，能查看创建的 DB 、存储对象(可理解成表）以及表中添加的数据。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200925180719957.png" alt="image-20200925180719957" style="zoom:50%;" />

IndexedDB 有个非常强大的功能，就是 index（索引）。它可对 Value 对象中任何属性生成索引，然后可以基于索引进行 Value 对象的快速查询

分析：IndexedDB 是一种灵活且功能强大的数据存储机制，它集合了 Dom Storage 和 Web SQL Database 的优点，用于存储大块或复杂结构的数据，提供更大的存储空间，使用起来也比较简单。可以作为 Web SQL Database 的替代。不太适合静态文件的缓存。

1. 以key-value 的方式存取对象，可以是任何类型值或对象，包括二进制。
2. 可以对对象任何属性生成索引，方便查询。
3. 较大的存储空间，默认推荐250MB(分 HOST)，比 Dom Storage 的5MB 要大的多。
4. 通过数据库的事务（tranction）机制进行数据操作，保证数据一致性。
5. 异步的 API 调用，避免造成等待而影响体验。

Android 在4.4开始加入对 IndexedDB 的支持，只需打开允许 JS 执行的开关就好了

```java
WebView myWebView = (WebView) findViewById(R.id.webview);
WebSettings webSettings = myWebView.getSettings();
webSettings.setJavaScriptEnabled(true);
```

### 移动端 Web 加载性能（缓存）优化

通过对一些 H5页面进行调试及抓包发现，每次加载一个 H5页面，都会有较多的请求。除了 HTML 主 URL 自身的请求外，HTML外部引用的 JS、CSS、字体文件、图片都是一个独立的 HTTP 请求，每一个请求都串行的（可能有连接复用）。这么多请求串起来，再加上浏览器解析、渲染的时间，Web 整体的加载时间变得较长；请求文件越多，消耗的流量也会越多。我们可综合使用上面说到几种缓存机制，来帮助我们优化 Web 的加载性能。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200925181143183.png" alt="image-20200925181143183" style="zoom:50%;" />

结论：综合各种缓存机制比较，对于静态文件，如 JS、CSS、字体、图片等，适合通过浏览器缓存机制来进行缓存，通过缓存文件可大幅提升 Web 的加载速度，且节省流量。但也有一些不足：缓存文件需要首次加载后才会产生；浏览器缓存的存储空间有限，缓存有被清除的可能；缓存的文件没有校验。要解决这些不足，可以参考手 Q 的离线包，它有效的解决了这些不足。

对于 Web 在本地或服务器获取的数据，可以通过 Dom Storage 和 IndexedDB 进行缓存。也在一定程度上减少和 Server 的交互，提高加载速度，同时节省流量。



































