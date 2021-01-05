## 第十三章 协程

### 事件循环

#### 获取协程的返回值

```python
import asyncio
import time
from functools import partial
 
async def get_html(url):
     print("start get url")
     await asyncio.sleep(2)
     return "bobby"

def callback(url, future):
    print(url)
    print("send email to bobby")

if __name__ == "__main__":
	loop = asyncio.get_event_loop()
    # 第一种方法
	get_future = asyncio.ensure_future(get_html("http://www.imooc.com"))
    loop.run_until_complete(get_future)
    print(get_future.result())
    
    # 第二种方法
	task = loop.create_task(get_html("http://www.imooc.com"))
    task.add_done_callback(partial(callback, "http://www.imooc.com"))
    loop.run_until_complete(task)
    print(task.result())
        
```

#### wait和gather区别

```python
import asyncio
import time
async def get_html(url):
    print("start get url")
    await asyncio.sleep(2)
    print("end get url")

if __name__ == "__main__":
    start_time = time.time()
    loop = asyncio.get_event_loop()
    # wait
    tasks = [get_html("http://www.imooc.com") for i in range(10)]
    loop.run_until_complete(asyncio.wait(tasks))
	
    # gather
    tasks = [get_html("http://www.imooc.com") for i in range(10)]
    loop.run_until_complete(asyncio.gather(*tasks))
    
    #gather和wait的区别
    #gather更加high-level,可以实现分组
    group1 = [get_html("http://projectsedu.com") for i in range(2)]
    group2 = [get_html("http://www.imooc.com") for i in range(2)]
    loop.run_until_complete(asyncio.gather(*group1, *group2))

```

#### 取消协程

```python
import asyncio
import time

async def get_html(sleep_times):
    print("waiting")
    await asyncio.sleep(sleep_times)
    print("done after {}s".format(sleep_times))


if __name__ == "__main__":
    task1 = get_html(2)
    task2 = get_html(3)
    task3 = get_html(3)

    tasks = [task1, task2, task3]

    loop = asyncio.get_event_loop()

    try:
        loop.run_until_complete(asyncio.wait(tasks))
    except KeyboardInterrupt as e:
        all_tasks = asyncio.Task.all_tasks()
        for task in all_tasks:
            print("cancel task")
            print(task.cancel())
        loop.stop()
        loop.run_forever()
    finally:
        loop.close()
```

#### 协程调用子协程

```python
import asyncio

async def compute(x, y):
    print("Compute %s + %s ..." % (x, y))
    await asyncio.sleep(1.0)
    return x + y

async def print_sum(x, y):
    result = await compute(x, y)
    print("%s + %s = %s" % (x, y, result))

loop = asyncio.get_event_loop()
loop.run_until_complete(print_sum(1, 2))
loop.close()
```

![image text](https://upload-images.jianshu.io/upload_images/12363668-b2e0b27756ee614f.png)

#### 在协程中集成阻塞io：运用多线程

```python
loop = asyncio.get_event_loop()

executor = ThreadPoolExecutor(3)
task = loop.run_in_executor(executor, get_url, url) # get_url 为同步代码

loop.run_until_complete(asyncio.wait(tasks))
```

