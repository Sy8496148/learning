```
objectid组成
一共有四部分组成:时间戳、客户端ID、客户进程ID、三个字节的增量计数器

如何理解MongoDB中的GridFS机制，MongoDB为何使用GridFS来存储文件？
　GridFS是一种将大型文件存储在MongoDB中的文件规范。使用GridFS可以将大文件分隔成多个小文档存放，这样我们能够有效的保存大文档，而且解决了BSON对象有限制的问题
```