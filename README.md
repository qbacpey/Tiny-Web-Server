# Tiny Web Server

使用Epoll进行实现的简单Web服务器

## 代码设计

![image-20230319201257171](https://github.com/qbacpey/Tiny-Web-Server/blob/master/README.assets/image-20230319201257171.png)

### 并行性

本项目的并行性通过线程池（Posix线程）以及epoll进行实现

#### 执行流程

1. 主线程执行初始化工作（配置网络、监听端口等）；

2. 主线程创建epoll实例，将需要监听的文件描述符添加到epoll的兴趣列表中；

3. 主线程创建`THREAD_NUM`个线程（`epollfd`以及`listenfd`会被作为参数传递），随后主线程和子线程都将作为工作线程存在；

4. 所有工作线程在epoll实例上调用`epoll_wait`函数；

5. 当任何事件被捕捉到时，工作线程会执行下列操作：

   1. 检查`fd`是否为`listenfd`，如果是，那么对其调用`accept`并将`connfd`（`accept`的返回值）添加到兴趣列表中；

   2. 如果不是，那么`fd`就是`connfd`，此时可以从中读取数据。如果读取完成（检测到新的一行），那么响应此请求。否则将读取状态保存到`event.data.ptr`中

      读取状态的定义如下：

      ~~~c
      typedef struct HttpStatus {
          int connfd; // 连接文件描述符
          char *header; // 已读取的HTTP请求头
          size_t readn; // 已读取的字节数
          FILE *file; // 需要发送的文件
          size_t left; // 尚需发送的文件
          req_status_t req_status;
      } http_status_t;
      ~~~

      `req_status_t`被定义为：

      ~~~c
      typedef enum REQUEST_STATUS {
          Reading,
          Writing,
          Ended
      } req_status_t;
      ~~~

      下一次`epoll_wait`接收到此`fd`中的事件时，服务器会继续处理此请求

   3. 请求头读取完毕之后，`connfd`会进入`Writing`状态。如果`sendfile`函数发送数据时触发了`EAGAIN`事件，同时`left > 0`，那么意味着写端暂时不可用。保存当前状态到`HttpStatus`中之后，利用`EPOLL_CTL_MOD`将`connfd`以`EPOLLOUT | EPOLLET`重新装配到epoll实例中；
