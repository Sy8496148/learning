### Nginx请求流程

nginx使用一个多进程模型来对外提供服务，其中一个master进程，多个worker进程。master进程负责管理nginx本身和其他worker进程。

所有实际上的业务处理逻辑都在worker进程。worker进程中有一个函数，执行无限循环，不断处理收到的来自客户端的请求，并进行处理，直到整个nginx服务被停止。

worker进程中，ngx_worker_process_cycle()函数就是这个无限循环的处理函数。在这个函数中，一个请求的简单处理流程如下：

1. 操作系统提供的机制（例如epoll, kqueue等）产生相关的事件。
2. 接收和处理这些事件，如是接受到数据，则产生更高层的request对象。
3. 处理request的header和body。
4. 产生响应，并发送回客户端。
5. 完成request的处理。
6. 重新初始化定时器及其他事件。



### Nginx请求处理流程

从nginx的内部来看，一个HTTP Request的处理过程涉及到以下几个阶段。

1. 初始化HTTP Request（读取来自客户端的数据，生成HTTP Request对象，该对象含有该请求所有的信息）。
2. 处理请求头。
3. 处理请求体。
4. 如果有的话，调用与此请求（URL或者Location）关联的handler。
5. 依次调用各phase handler进行处理。

phase handler：包含若干个处理阶段的一些handler。在每一个阶段，包含有若干个handler，再处理到某个阶段的时候，依次调用该阶段的handler对HTTP Request进行处理，通常情况下，一个phase handler对这个request进行处理，并产生一些输出。通常phase handler是与定义在配置文件中的某个location相关联的。

一个phase handler通常执行以下几项任务：

1. 获取location配置。
2. 产生适当的响应。
3. 发送response header。
4. 发送response body。

当nginx读取到一个HTTP Request的header的时候，nginx首先查找与这个请求关联的虚拟主机的配置。如果找到了这个虚拟主机的配置，那么通常情况下，这个HTTP Request将会经过以下几个阶段的处理（phase handlers）：

| 阶段名称                                              | 是否可以挂载自定义模块 |
| :---------------------------------------------------- | ---------------------- |
| NGX_HTTP_POST_READ_PHASE：读取请求内容阶段            | 是                     |
| NGX_HTTP_SERVER_REWRITE_PHASE：Server请求地址重写阶段 | 是                     |
| NGX_HTTP_FIND_CONFIG_PHASE：配置查找阶段              | 否                     |
| NGX_HTTP_REWRITE_PHASE：Location请求地址重写阶段      | 是                     |
| NGX_HTTP_POST_REWRITE_PHASE：请求地址重写提交阶段     | 否                     |
| NGX_HTTP_PREACCESS_PHASE：访问权限检查准备阶段        | 是                     |
| NGX_HTTP_ACCESS_PHASE：访问权限检查阶段               | 否                     |
| NGX_HTTP_POST_ACCESS_PHASE：访问权限检查提交阶段      | 是                     |
| NGX_HTTP_TRY_FILES_PHASE：配置项try_files处理阶段     | 否                     |
| NGX_HTTP_CONTENT_PHASE：内容产生阶段                  | 是                     |
| NGX_HTTP_LOG_PHASE：日志模块处理阶段                  | 是                     |

内容产生阶段完成以后，生成的输出会被传递到filter模块去进行处理。filter模块也是与location相关的。所有的fiter模块都被组织成一条链。输出会依次穿越所有的filter，直到有一个filter模块的返回值表明已经处理完成。



### Nginx 模块开发

#### 配置文件概述

nginx.conf中的配置信息，根据其逻辑上的意义，对它们进行了分类，也就是分成了多个作用域，或者称之为配置指令上下文。不同的作用域含有一个或者多个配置项。

当前nginx支持的几个指令上下文：

| 指令上下文 | 说明                                                         |
| :--------- | ------------------------------------------------------------ |
| http:      | 与提供http服务相关的一些配置参数。例如：是否使用keepalive啊，是否使用gzip进行压缩等。 |
| server:    | http服务上支持若干虚拟主机。每个虚拟主机一个对应的server配置项，配置项里面包含该虚拟主机相关的配置。在提供mail服务的代理时，也可以建立若干server.每个server通过监听的地址来区分。 |
| location:  | http服务中，某些特定的URL对应的一系列配置项。                |
| main:      | nginx在运行时与具体业务功能（比如http服务或者email服务代理）无关的一些参数，比如工作进程数，运行的身份等。 |

我们开发的模块一般处于location指令上下文中，一个模块中可以有多个指令

例：

配置文件

location /doggy {

​			doggy "hello dog";

}

我们要开发一个模块为 doggy 其中包含 指令 doggy ，其接收的参数为一个 ngx_str_t

开发步骤

1，读取配置文件中的命令，以及参数（如果有参数）

2，进行http包装（添加请求头等操作）

3，返回给客户端



#### 定义配置模块结构体

用于从配置文件中读取参数（如果配置文件中没有参数可以忽略此步骤）

```c
typedef struct {
    ngx_str_t wd;
} ngx_http_doggy_loc_conf_t;
```

ngx_str_t：nginx内部定义数据

wd：用于接收用户存取命令字符串参数，（ "hello dog"）

ngx_http_doggy_loc_conf_t：命名规则为 ngx_http\_[模块名称\]_\[main\|srv\|loc\]\_conf_t ，此**模块的名称为 doggy**（注意区分模块名称跟指令名称），doggy模块运行在location下，所以选择loc



#### 定义指令

Nginx模块使用一个ngx_command_t数组表示模块所能接收的所有指令，其中每一个元素表示一个条指令.

ngx_command_t是ngx_command_s的一个别称ngx_command_s定义在core/ngx_config_file.h中：

```c
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

我们只需要定义 指令数组的命名规则为ngx_http_\[模块名称]\_commands，数组最后一个元素要ngx_null_command结束

```c
static ngx_command_t  ngx_http_doggy_commands[] = {
    {
        ngx_string("doggy"),
        NGX_CONF_TAKE1|NGX_HTTP_LOC_CONF,
        ngx_http_doggy,
        NGX_HTTP_LOC_CONF_OFFSET,
        0,
        NULL 
    },
        ngx_null_command
};
```

参数说明

**name：ngx_string("doggy")**：指令的名称

**type：NGX_CONF_TAKE1\|NGX_HTTP_LOC_CONF**：配置指令的参数，NGX_CONF_TAKE 1表示接收一个参数；NGX_HTTP_LOC_CONF表示只会出现在location中，如果没有参数则设置为 NGX_CONF_NOARGS

**\*(\*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)：ngx_http_doggy**：参数转化函数，将配置文件中相关指令的参数转化成需要的格式并存入配置结构体，如果没有参数设置为NULL

```c
static char *
ngx_http_doggy(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_conf_set_str_slot(cf,cmd,conf);
    return NGX_CONF_OK;
}
```

这个函数除了调用内置的ngx_conf_set_str_slot（将裸字符串转化为ngx_str_t）转化doggy指令的参数

**conf：NGX_HTTP_LOC_CONF_OFFSET：**用于指定Nginx相应配置文件内存真实地址，一般可以通过内置常量NGX_HTTP_LOC_CONF_OFFSET指定，

**offset：0：**offset指定此条指令的参数的偏移量，一般设置为0

***post：NULL**：一般设置为NULL



#### 定义模块context

需要定义一个ngx_http_module_t类型的结构体变量，命名规则为ngx_http\_[模块名称]_module_ctx，这个结构主要用于定义各个Hook函数。下面是doggy模块的context结构

```c
static ngx_http_module_t  ngx_http_doggy_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_doggy_init,                   /* postconfiguration */
    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */
    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */
    ngx_http_doggy_create_loc_conf,        /* create location configration */
    NULL                                   /* merge location configration */
};
```

doggy 模块只作用于location，只需要定义最后两个Hook函数，其它设置为null。这两个函数会被nginx自动调用，命名规则为ngx_http\_[模块名称]\_[create|metge]_loc_conf。

下面是这两个函数的具体用法

**ngx_http_doggy_create_loc_conf:**

```c
static void *
ngx_http_doggy_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_doggy_loc_conf_t  *conf;
    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_doggy_loc_conf_t));
    if (conf == NULL) {
        return NGX_CONF_ERROR;
    }
    conf->wd.len = 0;
    conf->wd.data = NULL;
    return conf;
}
```

ngx_pcalloc用于在Nginx内存池中分配一块空间，是pcalloc的一个包装。使用ngx_pcalloc分配的内存空间不必手工free，Nginx会自行管理，在适当是否释放。

create_loc_conf新建一个ngx_http_doggy_loc_conf_t，分配内存，并初始化其中的数据，然后返回这个结构的指针。

**ngx_http_doggy_init**:

```c
static ngx_int_t
ngx_http_doggy_init(ngx_conf_t *cf)
{
        ngx_http_handler_pt        *h;
        ngx_http_core_main_conf_t  *cmcf;

        cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

        h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
        if (h == NULL) {
                return NGX_ERROR;
        }

        *h = ngx_http_doggy_handler;

        return NGX_OK;
}
```

挂载模块到对应的阶段，如果是要对http的请求头做处理则要挂载到NGX_HTTP_REWRITE_PHASE阶段



由于这个模块的指令只会出现在location层所以不会发生合并的情况，所以 /* merge location configration */ 可以设置为NULL



#### 编写handler

编写步骤

1，读入模块配置

2，处理功能业务

3，验证通过返回 HTTP对象，失败拒绝此次请求



#### 定义模块

```c
ngx_module_t  ngx_http_doggy_module = {
    NGX_MODULE_V1,
    &ngx_http_doggy_module_ctx,             /* module context */
    ngx_http_doggy_commands,                /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

开头结尾分别用NGX_MODULE_V1， NGX_MODULE_V1_PADDING填充。

从上到下以依次为context、指令数组、模块类型以及若干特定事件的回调处理函数（不需要可以置为NULL），注意我们的doggy是一个HTTP模块，所以这里类型是NGX_HTTP_MODULE。



#### 最终代码

ngx_http_doggy_module.c

```c
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

/* Module config */
typedef struct {
    ngx_str_t  wd;
} ngx_http_doggy_loc_conf_t;

static char *ngx_http_doggy(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
static void *ngx_http_doggy_create_loc_conf(ngx_conf_t *cf);
static char *ngx_http_doggy_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child);

/* Directives */
static ngx_command_t  ngx_http_doggy_commands[] = {
    { ngx_string("doggy"),
        NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
        ngx_http_doggy,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_doggy_loc_conf_t, wd),
        NULL },
        ngx_null_command
};

/* Http context of the module */
static ngx_http_module_t  ngx_http_doggy_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_doggy_init,                   /* postconfiguration */
    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */
    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */
    ngx_http_doggy_create_loc_conf,        /* create location configration */
    NULL,                                  /* merge location configration */
};

/* Module */
ngx_module_t  ngx_http_doggy_module = {
    NGX_MODULE_V1,
    &ngx_http_doggy_module_ctx,             /* module context */
    ngx_http_doggy_commands,                /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};

/* Handler function */
static ngx_int_t
ngx_http_doggy_handler(ngx_http_request_t *r)
{
    /* 判断加密狗是否在线 */
    /* 如果不在线返回异常 */
    /* 在线读取加密狗信息 */
    /* 读取失败返回异常 */
    /* 将读取信息添加到请求头 */
    /* 将请求返回发送给下一个handler处理 */
}

static char *
ngx_http_doggy(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_conf_set_str_slot(cf,cmd,conf);
    return NGX_CONF_OK;
}

static void *
ngx_http_doggy_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_doggy_loc_conf_t  *conf;
    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_doggy_loc_conf_t));
    if (conf == NULL) {
        return NGX_CONF_ERROR;
    }
    conf->ed.len = 0;
    conf->ed.data = NULL;
    return conf;
}

static char *
ngx_http_doggy_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_doggy_loc_conf_t *prev = parent;
    ngx_http_doggy_loc_conf_t *conf = child;
    ngx_conf_merge_str_value(conf->ed, prev->ed, "");
    return NGX_CONF_OK;
}

static ngx_int_t
ngx_http_doggy_init(ngx_conf_t *cf)
{
        ngx_http_handler_pt        *h;
        ngx_http_core_main_conf_t  *cmcf;

        cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

        h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
        if (h == NULL) {
                return NGX_ERROR;
        }

        *h = ngx_http_doggy_handler;

        return NGX_OK;
}
```



#### 模块安装

创建一个文件夹（例: /home/ngxdev/ngx_http_doggy），里面放入config文件，和doggy.c 

1，编写config文件

```c
例：
ngx_addon_name=模块完整名称
HTTP_MODULES="$HTTP_MODULES 模块完整名称"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/源代码文件名"
```

```c
ngx_addon_name=ngx_http_doggy_module
HTTP_MODULES="$HTTP_MODULES ngx_http_doggy_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_doggy_module.c"
```

2，编译安装

```
./configure --prefix=/usr/local/nginx --add-module=/home/ngxdev/ngx_http_doggy
make
sudo make install
```



https://tengine.taobao.org/book/chapter_03.html#id6