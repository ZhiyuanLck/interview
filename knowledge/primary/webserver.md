# webserver思路

核心类为Response、Request、Connection、Session

## Response

Response与一个Session对象（包含连接和Request对象）绑定，用来发送响应包。

Response是ostream的派生类，与一个额外的streambuf相关联，使用write函数来将头部和数据写入streambuf

构造Response对象时基类使用空指针初始化，steambuf默认初始化为空，以及接收一个Session对象的共享指针

send函数

- 从session中获得连接的socket，使用async_write向socket写入buffer中的数据
- 发送完毕后调用一个回调函数，该回调函数参数为与此次发送相关的Response和Request对象的共享指针

## Request

Request继承自istream，负责从buffer中解析出请求方法、请求字段、HTTP版本

需要read_time保存读取buffer完毕时的时间

Request还需要保存一个connection对象，以及获取此连接的本地端点和远程端点的方法

## 连接相关的类

Connection类保存一个socket对象和定时器的指针，具备关闭连接，重置定时器，取消定时器的功能。定时器到时则关闭连接。每个对象新建一个socket对象，每次重置定时器时新建一个新的定时器对象。

Session类保存一个Connection对象和一个Request对象的共享指针，初始化时转移一个传入的connection对象的所有权，并新建一个Request对象

## Server类

Server类的成员

- Config类对象，保存配置信息，比如地址、端口号、buffer最大大小、超时的时间等
- resource对象，保存用来匹配路径的正则表达式和请求方法到回调函数的映射
- default_resource对象，保存请求方法到回调函数的映射，在resource中找不到对应的key时使用设置的默认值
- 用于启动和终止服务的锁
- 线程池，容器为vector
- 连接池，使用hash表管理，方便查询
- acceptor对象，用来接收连接请求
- io_context对象，I/O对象运行在io_service上
- on_error回调函数

start函数启动服务器，可以传入一个参数为port的回调函数，在服务启动后调用

- 根据设置中的地址和端口号设置本地端点，地址为空则使用默认的IPv4或IPv6地址
- 使用端点指定的协议调用acceptor的open函数，然后绑定到本地端点，开始监听然后可以接收连接请求
- 重启可能停止的io_service服务
- 在线程池的每个线程中调用run
- 启动主线程的io_service (run)
- 等待其他子线程结束

accept函数异步接收请求

- 创建一个连接
- 调用async_accept函数，回调函数中
  - 立即接收下一个新的连接，除非接收到operation_aborted错误
  - 为接收的连接创建一个Session对象（不管出错没）
  - 如果没有出错，关闭Naggle算法，调用read开始读取数据

stop函数停止接收新的请求并关闭所有连接，析构时调用

- 调用acceptor的close函数
- 关闭连接池中所有连接然后清空连接池
- 停止io_service

create_connection创建一个连接

- 新建一个Connection对象的共享指针并指定析构方式，即从连接池中删除然后delete
- 将连接放入连接池

read函数从socket中读取数据并写入Request对象中，通过传入Session对象实现

- 设置连接的定时器
- 使用async_read_until读取首部，以"\r\n\r\n"分割，读取完毕调用回调函数
  - 重置定时器
  - 记录写入的时间
  - 如果没有出错则解析Request中读取到的头部的内容，解析失败则调用on_error回调函数并返回
  - 查找Content_Length字段，没找到则调用对应的资源回调函数
  - 解析Content_Length字段，解析失败则调用回调函数并返回
  - 长度超过buffer最大值则发送包含client_error_payload_too_large状态码的相应包，并调用出错回调函数返回
  - 使用async_read继续读取剩下的内容，回调函数中调用对应的处理函数

find_resource函数接收一个Session对象并解析对应的Request对象中的内容然后调用对应的处理函数

- 找到resource中匹配的正则项对应的请求方法的处理函数
- 没有找到则调用默认的处理函数
- 调用write函数发送相应包

write函数用于发送请求包，参数为Session对象的共享指针以及一个回调函数，该回调函数参数为Response和Request对象的共享指针

- 新建response对象，并指定析构时调用response的send_on_delete函数，并且设置send的回调函数
  - 取消连接的定时器（不管出不出错）
  - 如果指定响应后关闭连接则立马返回
  - 查找Connection字段，如果设置为"close"则返回，如果是"keep-alive"，则用之前的连接新建Session对象继续接收请求并返回
  - 如果HTTP版本大于1.1则默认keep-alive，执行与上面相同的步骤
  - 出错则调用回调函数
- 调用resource_function回调函数

start -> read -> find_resource -> write -> read -> ... -> write -> stop

- Request对象是在构建Session对象的时候构建的
- Session对象是在新接收一个连接或者一次通信结束继续复用的时候构建
- Response对象是在write函数中构建的，即需要发送响应包时才构建
- Connection对象是在接收请求的时候构建的
