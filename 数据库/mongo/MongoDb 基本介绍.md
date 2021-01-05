## Mongodb基本介绍

- 简介:

  MongoDB 是一个基于分布式文件存储的数据库,是一个介于关系数据库和非关系数据库之间的产品。

- 跟关系型数据库的一些概念对比

  | SQL术语/概念 | MongoDB术语/概念 |  说明   |
  | :----------: | :--------------: | :-----: |
  |   database   |     database     | 数据库  |
  |    table     |    collection    | 表/集合 |
  |     row      |     document     | 行/文档 |
  |    column    |      field       | 字段/域 |
  |    index     |      index       |  索引   |

- 内存管理

  MongoDB使用的是内存映射存储引擎，把磁盘文件的一部分或全部内容直接映射到内存，这样文件中的信息位置就会在内存中有对应的地址空间，内存中主要存储**索引+热数据**，Mongodb没有单独的内存管理机制，跟随当前的操作系统的内存管理。
  
  

## Python操作Mongodb

- 主流的操作mongodb模块 pymongo

  安装 **pip install pymongo**

- 创建连接对象

  ```python
  from pymongo import MongoClient
  
  class MongoDBClient(object):
      def __new__(cls):
          if not hasattr(cls, 'instance'):
              cls.instance = super(MongoDBClient, cls).__new__(cls)
          return cls.instance
  
      def __init__(self):
          self.params = {
              'host': 'localhost',
              'port': 27017,
              'connect': False,
              'maxPoolSize': 2000,
              "authSource": 'workbench',  # 验证数据库
              "username": 'cnns',       # 用户名
              "password": 'cnns.net'    # 密码
          }
          self.mgdb = MongoClient(**params)
  
      def getMongoClient(self):
          return self.mgdb
  ```
  
- 选择数据库集合

  ```python
  mongoclinet = MongoDBClient().getMongoClient()
  db = mongoclinet['test'] # 选择数据库 如果数据库不存在会自动创建
  collection = db['warn'] # 选择集合 如果集合不存在会自动创建
  ```

- 增

  ```python
  article = {
      'id': 123456,
      'author': '我是作者',
      'title': '我是文章标题',
      'tag': {
          '1111': 1111,
          '2222': 2222,
      },
      'timestamp': int(round(time.time() * 1000)),
  }
  
  collection.insert_one(article)
  ```

- 删

  ```python
  # 删除满足条件的一条数据
  collection.delete_one({'title': '我是文章标题'})  
  
  # 删除满足条件的所有数据
  collection.delete_many({'title': '我是文章标题'})  
  ```

- 改

  ```python
  # 第一个参数是条件，第二个参数是要修改的参数以及对应修改成的值 
  # 修改满足条件的第一参数 
  collection.update_one({'title': '我是文章标题'}, {'$set': {'author': '我是作者2'}})
  
  # 修改满足条件的所有参数 update collection set author='我是作者2' where title='我是文章标题'
  collection.update_many({'title': '我是文章标题'}, {'$set': {'author': '我是作者2'}})
  ```

- 查 

  ```python
  # 查询单条数据
  # 只返回满足查询结果的第一条数据 返回python字典类型
  # {'author': '我是作者'} 查询的条件，{'_id': 0} 表示返回的结果中不包含'_id' 字段
  collection.find_one({'author': '我是作者'}, {'_id': 0})
  
  # 查询多条数据 
  # 返回所有满足条件的结果 返回Cursor
  results = collection.find({'author': '我是作者'})
  for result in results:
      print(result) # 字典
  
  # 多条件查询 
  # and 类似mysql select * from collection where author = '我是作者' AND tag = '1111'
  results = collection.find({'author': '我是作者', 'tag.1111': 1111})
  for result in results:
  	print(result)
  # or 类似mysql select * from collection where author = '我是作者' OR tag = '1111'
  results = collection.find({'$or': [{"author":"我是作者"},{"tag.1111": "1111"}]})
  for result in results:
  	print(result)   
  
  # 范围查询 
  # 查询author为‘我是作者’且timestamp大于等于65535的文章 
  # 类似mysql Select * from collection where author='我是作者' AND  likes>=200;
  # $lt 小于 $gt 大于 $lte 小于等于 $gte 大于等于 $ne 不等于 
  # $in 在范围内 $nin 不在范围内 ex：'timestamp': {'$in': [0, 65535]}
  results = collection.find({'author': '我是作者', 'timestamp': {'$gte': 65535}})
  for result in results:
    	print(result)
  
  # 排序 
  # 1 升序 -1 倒序
  # 类似mysql select * from collection order by timestamp desc
  results = collection.find().sort('timestamp', -1)
  for result in results:
      print(result)
  
  # 切片
  # skip(2) 表示查询结果跳过前面两条数据
  results = collection.find({'author': '我是作者'}).skip(2)
  for result in results:
    	print(result)
  # limit(2) 表示查询结果只取前面两条数据
  # 类似mysql select * from collection where author='我是作者' LIMIT 2
  results = collection.find({'author': '我是作者'}).limit(2)
  for result in results:
    	print(result)
      
  # 计数 
  # 查找author为'我是作者'数据的总数
  # 类似mysql select count(*) from telephone_numbers WHERE author='我是作者'
  collection.count_documents({'author': '我是作者'})
  ```

- 更多操作查看官方文档 https://docs.mongodb.com/manual/reference/operator/


