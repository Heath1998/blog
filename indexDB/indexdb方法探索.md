##  indexDB 查询方法初探

> indexDB, 相信大家都听过这个名词。无非是存储在浏览器的数据库，但是今天的主角是里面的一个查询方法。但是还是先了解些前置知识吧。


###   概述
> 数据库的操作流程，基本大同小异。indexDB对一个已存在的数据库正常流程为，打开数据库 -->  创建一个事务指定操作的对象库  --> 执行对数据库中对象库的操作

####  打开数据库

```javascript
let request = indexedDB.open("example", 1); 

// 第一次打开或更新版本执行
request.onupgradeneeded = function() { 
	
}; 

// 成功打开后执行
openRequest.onsuccess = function() { 
	let db = openRequest.result;  
};
  
```


####  对象库(object store)
可以简单理解为数据库中的"表"。是存储数据的地方。

```javascript
let request = indexedDB.open("example", 1); 

// 第一次打开或更新版本执行
request.onupgradeneeded = function() { 
	db.createObjectStore('things', {keyPath: 'id'});
}; 
  
```

####  事务
对于事务，要么都做，要么都不做。我们一般将事务添加到对象库中进行操作
```javascript
let transaction = db.transaction("things", "readwrite");
let things = transaction.objectStore("things");

let thing = {
	x: 6,
	y: 6,
	z: 6
};

let request = books.add(thing);

request.onsuccess = function() { };

```



###  搜索

####  通过key搜索

```javascript

let transaction = db.transaction("things", "readwrite");
let things = transaction.objectStore("things");
things.get(234);

// 123 <= 345
things.getAll(IDBKeyRange.bound(123, 345));
```

#### 光标(Cursors)
光标，在查询对象库的时候，一次只返回一个值。
由对象库是按排序返回的。默认升序。



####  通过索引字段搜索
> 终于讲到使用bound的方法了，这里将会有比较困惑的结果

如果想搜索其它字段，我们需要创建一个名为"索引(index)"。
索引是存储的"附加项", 它存储有该值的对象的键列表。

先创建索引，索引可以为数组
``` javascript

open.onupgradeneeded = function() {
  var db = open.result;
  var store = db.createObjectStore('things', {autoIncrement: true});
  var index = store.createIndex('coords', ['x', 'y', 'z'], {unique: false});
};

```

``` javascript

index.openCursor(IDBKeyRange.bound([4,3,7], [6,5,9])).onsuccess = (e) => {
	const cursor = e.target.result;
	cursor.continue();
};

```

但是.bound的结果并不是获取到我们预期的
```
[4=<x<=6, 3<=y<=5,7<=z<=9]
```
这种结果，可以理解为会获取数字437<=?<=659之间的数。
