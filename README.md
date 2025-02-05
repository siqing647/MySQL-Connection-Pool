
- [1 环境配置](#1-环境配置)
- [2 使用方式](#2-使用方式)
- [3 关键技术点](#3-关键技术点)
- [4 连接池的使用场景](#4-连接池的使用场景)
- [5 连接池参数介绍](#5-连接池参数介绍)
- [6 功能实现设计](#6-功能实现设计)
- [7 压力测试](#7-压力测试)
- [8 MySQL 数据库编程](#8-mysql-数据库编程)

参考[开发日志](https://github.com/Corner430/MySQL-Connection-Pool/blob/main/logs.md)进行学习

# 1 环境配置

[Docker 开发环境](https://github.com/Corner430/Docker/tree/main/MySQL-Connection-Pool)

# 2 使用方式

```shell
cd build && cmake .. && make
```

CMakeList.txt 文件需要加入如下内容：

```cmake
# 定义 MySQL 配置文件路径
add_definitions(-DMYSQL_CONFIG_FILE_PATH=\"${PROJECT_SOURCE_DIR}/config/mysql.conf\")
```

在自己创建的项目中，引入 `libmysql_connection_pool.so` 动态链接库文件，然后在项目中引入 `/include` 目录下的头文件和 mysql.conf 配置文件即可使用连接池功能，具体使用方式可以参考 `examples` 目录下的示例程序。

# 3 关键技术点

MySQL 数据库编程、单例模式、`queue` 队列容器、C++11 多线程编程、线程互斥、线程同步通信和 `unique_lock`、基于 CAS 的原子整型、智能指针 `shared_ptr`、`lambda`表达式、生产者-消费者线程模型

# 4 连接池的使用场景

**业务层与 MySQL 数据层**的交互往往会成为服务的瓶颈，解决方案如下：

- **在数据层增加缓存数据库（redis）缓存常用的数据**
- **连接池**
  - **在高并发情况下，大量的 TCP 三次握手、MySQL Server 连接认证、MySQL Server 关闭连接回收资源和 TCP 四次挥手**所耗费的资源不容忽视，增加连接池就是为了减少这一部分的性能损耗。

# 5 连接池参数介绍

连接池一般包含了**数据库连接所用的 ip, port, username 和 password 以及其它的性能参数**，例如**初始连接量，最大连接量，最大空闲时间、连接超时时间等**

**该项目主要实现如下连接池四大参数对应的功能**

- **初始连接量(`initSize`)**：表示连接池事先会和 MySQL Server 创建 `initSize` 个数的 `connection` 连接，当应用发起 MySQL 访问时，不用再创建和 MySQL Server 新的连接，直接从连接池中获取一个可用的连接就可以，使用完成后，并不去释放 `connection`，而是把当前 `connection` 再归还到连接池当中

- **最大连接量(`maxSize`)**：当并发访问 MySQL Server 的请求增多时，初始连接量已经不够使用了，此时会根据新的请求数量去创建更多的连接给应用去使用，但是新创建的连接数量上限是 `maxSize`，不能无限制的创建连接，因为每一个连接都会占用一个 `socket` 资源，**一般连接池和服务器程序是部署在一台主机上的**，如果连接池占用过多的 `socket` 资源，那么服务器就不能接收太多的客户端请求了。当这些连接使用完成后，再次归还到连接池当中来维护

- **最大空闲时间(`maxIdleTime`)**：当访问 MySQL 的并发请求多了以后，连接池里面的连接数量会**动态增加**，上限是 `maxSize` 个，当这些连接用完再次归还到连接池当中。如果在指定的 `maxIdleTime` 里面，这些新增加的连接都没有被再次使用过，那么新增加的这些连接资源就要被回收掉，只需要保持初始连接量 `initSize` 个连接就可以了。

- **连接超时时间(`connectionTimeout`)**：当 MySQL 的并发请求量过大，连接池中的连接数量已经到达 `maxSize` 了，而此时没有空闲的连接可供使用，那么此时应用从连接池获取连接无法成功，**它通过阻塞的方式获取连接**的时间如果超过 `connectionTimeout` 时间，那么获取连接失败，无法访问数据库。

> `show variables like 'max_connections';` 该命令可以查看 **MySQL Server 所支持的最大连接个数**，超过 `max_connections` 数量的连接，MySQL Server 会直接拒绝，所以在使用连接池增加连接数量的时候，MySQL Server 的 `max_connections` 参数也要适当的进行调整，以适配连接池的连接上限。

# 6 功能实现设计

**项目目录结构如下**

```shell
.
|-- CMakeLists.txt  # CMake 构建文件
|-- README.md       # 项目说明文档
|-- bin             # 可执行文件
|   `-- example
|-- build           # CMake 构建目录
|-- config          # 配置文件
|   `-- mysql.conf
|-- examples        # 示例程序源码
|   `-- main.cpp
|-- include         # 头文件
|   |-- CommonConnectionPool.h  # 连接池头文件
|   |-- Connection.h          # 数据库操作头文件
|   `-- public.h           # 公共头文件
|-- lib            # 库文件
|   `-- libmysql_connection_pool.so # 连接池动态链接库输出文件
`-- src           # 源码
    |-- CommonConnectionPool.cpp  # 连接池代码实现
    `-- Connection.cpp        # 数据库操作代码实现
```

**连接池主要包含了以下功能点**：

1. 连接池只需要一个实例，所以 `ConnectionPool` 以**单例模式**进行设计
2. 从 `ConnectionPool` 中可以获取和 MySQL 的连接 `Connection`
3. 空闲连接 `Connection` 全部维护在一个线程安全的 `Connection` 队列中，使用线程互斥锁保证队列的线程安全
4. 如果 `Connection` 队列为空，还需要再获取连接，此时需要动态创建连接，上限数量是 `maxSize`
5. 队列中空闲连接时间超过 `maxIdleTime` 的就要被释放掉，只保留初始的 `initSize` 个连接就可以了，这个功能点肯定需要放在独立的线程中做
6. 如果 `Connection` 队列为空，而此时连接的数量已达上限 `maxSize`，那么等待 `connectionTimeout` 时间，如果还获取不到空闲的连接，那么获取连接失败，此处从 `Connection` 队列获取空闲连接，可以使用带超时时间的 `mutex` 互斥锁来实现连接超时时间
7. 用户获取的连接用 `shared_ptr` 智能指针来管理，**用 `lambda` 表达式定制连接释放的功能(不真正释放连接，而是把连接归还到连接池中)**
8. 连接的生产和连接的消费采用**生产者-消费者**线程模型来设计，使用了线程间的**同步通信机制条件变量和互斥锁**

# 7 压力测试

验证数据的插入操作所花费的时间，第一次测试使用普通的数据库访问操作，第二次测试使用带连接池
的数据库访问操作，对比两次操作同样数据量所花费的时间，性能压力测试结果如下:

| 数据量 |     未使用连接池花费时间     |      使用连接池花费时间      |
| :----: | :--------------------------: | :--------------------------: |
|  1000  |  单线程:1891ms 四线程:497ms  |  单线程:1079ms 四线程:408ms  |
|  5000  | 单线程:10033ms 四线程:2361ms | 单线程: 5380ms 四线程:2041ms |
| 10000  | 单线程:19403ms 四线程:4589ms | 单线程:10522ms 四线程:4034ms |

# 8 MySQL 数据库编程

如果是 windows，在 VS 上需要进行相应的头文件和库文件的配置，如下:

1. 右键项目 - C/C++ - 常规 - 附加包含目录，填写 `mysql.h` 头文件的路径
2. 右键项目 - 链接器 - 常规 - 附加库目录，填写 `libmysql.lib` 的路径
3. 右键项目 - 链接器 - 输入 - 附加依赖项，填写 `libmysql.lib` 库的名字
4. 把 `libmysql.dll` 动态链接库(Linux 下后缀名是 `.so` 库)放在工程目录下

- `int mysql_errno(MYSQL*)`：返回上次调用的 MySQL 函数的错误编号，即 **mysql 错误码**
- `const char* mysql_error(MYSQL*)`：返回上次调用的 MySQL 函数的错误消息

MySQL 数据库 C++代码封装如下:

**_main.cpp_**

```cpp
#include "Connection.h"
#include "public.h"
#include <iostream>
using namespace std;

// 初始化数据库连接
Connection::Connection() { _conn = mysql_init(nullptr); }

// 释放数据库连接资源
Connection::~Connection() {
  if (_conn)
    mysql_close(_conn);
}

// 连接数据库
bool Connection::connect(string ip, unsigned short port, string user,
                         string password, string dbname) {
  MYSQL *p =
      mysql_real_connect(_conn, ip.c_str(), user.c_str(), password.c_str(),
                         dbname.c_str(), port, nullptr, 0);
  // cout << mysql_error(_conn) << endl;
  return p != nullptr;
}

// 更新操作 insert delete update
bool Connection::update(string sql) {
  if (mysql_query(_conn, sql.c_str())) {
    LOG("更新失败：" + sql);
    return false;
  }
  return true;
}

// 查询操作 select
MYSQL_RES *Connection::query(string sql) {
  if (mysql_query(_conn, sql.c_str())) {
    LOG("查询失败：" + sql);
    return nullptr;
  }
  return mysql_use_result(_conn);
}
```

**_public.h_**

```cpp
#ifndef PUBLIC_H
#define PUBLIC_H

#define LOG(str)                                                               \
  cout << __FILE__ << ":" << __LINE__ << " " << __TIMESTAMP__ << " : " << str  \
       << endl

#endif
```
