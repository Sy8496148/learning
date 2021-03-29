#### Flask 线程隔离

current_app、request、session、g等对应都是指向相应栈的栈顶，当我们接受到一个请求里，flask才会把相应的app对象和新创建的Request对象压入相应的栈。由此，我们就知道了flask是怎么实现的多个请求，request的线程隔离。最后总结一下，字典、Local、LocalStack三者的关系，Local使用字典的方式实现的线程隔离，LocalStack封装了Local对象，并把它作为了自己的一个属性，从而实现了线程隔离的栈结构