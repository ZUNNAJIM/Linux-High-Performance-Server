# Linux高性能服务器编程

[toc]



## TCP/IP协议族

### TCP的有限状态机

<img src="https://i.loli.net/2021/07/07/lE71sS3BpUKLuDX.png"  height="500" width="600">

​    

### TCP/IP 协议族

<img src="https://i.loli.net/2021/07/07/H2fkmFTBC5pKvEo.png" height="400" width="600">

## Linux网络编程基础API

### socket地址API

#### 主机字节序和网络字节序

- 大端字节序：一个整数的高位字节存储在内存的低地址处；
- 小端字节序：一个整数的低位字节存储在内存的高地址处；

***检查机器的字节序代码：***

```c++
#include <stdio.h>
int main()
{
    union
    {
        short value;
        char union_bytes[sizeof(short)];
    }test;
    test.value = 0x0102;
    if((test.union_bytes[0] == 1) && (test.union_bytes[1] == 2))
    {
        printf("big edian!\n");
    }
    else if((test.union_bytes[0] == 2) && (test.union_bytes[1] == 1))
    {
        printf("small endian!\n");
    }
    else
    {
        printf("unknown\n");
    }
    return 0;
}
```

**解决不同字节序之间的格式问题：发送端总是将要发送的数据转换为大端字节序，并且通知接收方按照大端字节序来接收，接收端根据自己的字节序再决定要不要进行转换。**

**==Linux提供的4个转换网络字节序和主机字节序的函数：==**

```c++
#inlcude <netinet/in.h>

unsigned long int htonl(unsigned long int hostlong);		// host to network
unsigned short int htons(unsigned short int hostshort);		
unsigned long int ntohl(unsigned long int netlong);			// network to host
unsigned short int ntohs(unsigned short int netshort);
```



####  通用socket地址

socket网络编程接口中表示socket地址的是`sockaddr`结构体：

```c++
#include <bits/socket.h>
struct sockaddr
{
    sa_family_t sa_family;			// sa_family_t是地址族类型
    char sa_data[14];				// 用来存放地址值
}
```

<table border="1" cellspacing="1" cellpadding="1" align="center"> 
    <caption>
        <strong>
            <big>地址族与协议族的对应关系</big>
        </strong>
    </caption>
    <tr>
    </tr>
    <tr>
        <td align="center"><b>协议族</b></td>
        <td align="center"><b>地址族</b></td>
        <td align="center"><b>描述</b></td>
    </tr>
    <tr>
	<td align="center">PF_UNIX</td>
    <td align="center">AF_UNIX</td>
    <td align="center">UNIX本地域协议族</td>
</tr>
<tr>
	<td align="center">PF_INET</td>
    <td align="center">AF_INET</td>
    <td align="center">TCP/IPv4协议族</td>
</tr>
<tr>
	<td align="center">PF_INET6</td>
    <td align="center">AF_INET6</td>
    <td align="center">TCP/IPv6协议族</td>
</tr>





<table border="1" cellspacing="0" cellpadding="0" align="center"> 
    <caption>
        <strong>
            <big>协议族及其地址值</big>
        </strong>
    </caption>
    <tr>
    </tr>
    <tr>
        <td align="center"><b>协议族</b></td>
        <td align="center"><b>地址值含义及其长度</b></td>
    </tr>
    <tr>
	<td align="center">PF_UNIX</td>
    <td align="center">文件的路径名，长度可达108字节</td>
</tr>
<tr>
	<td align="center">PF_INET</td>
    <td align="center">16bit端口号和32bitIP地址</td>
</tr>
<tr>
	<td align="center">PF_INET6</td>
    <td align="center">16bit端口号，32bit流标识，128bit IPv6地址，32bit范围ID</td>
</tr>

为了适应不同字节长度的地址值，Linux更新了通用结构体sockaddr_storage

```c++
#inlclude <bits/socket.h>
struct sockaddr_storage
{
    sa_family_t sa_family;
    unsigned long int __ss_align;
    char __ss_padding[128-sizeof(__ss_align)];
}
```

#### 专用socket地址

UNIX本地域协议族专用：

```c++
#inlcude <sys/un.h>
struct sockaddr_un
{
    sa_family_t sin_family;		/* 地址族AF_UNIX */
    char sun_path[108];			/* 文件路径名 */
}
```



IPv4专用socket结构体：

```c++
struct sockaddr_in
{
    sa_family_t sin_family;			/* AF_INET地址族 */
    u_int_16_t sin_port;			/* 端口号，使用网络字节序表示 */
    struct in_addr sin_addr;		/* IPv4地址结构体结构体 */
};

struct in_addr
{	
	u_int32_t a_addr;				/* IPv4地址，使用网络字节序表示 */
}
```

IPv6专用的socket结构体：

```c++
struct sockaddr_in6
{
    sa_family_t sin6_family;			/* AF_INET6地址族 */
    u_int16_t sin6_port;				/* 端口号，使用网络字节序表示 */
    u_int32_t sin6_flowinfo;			/* 流标识，应设置为0 */
    struct in6_addr sin6_addr;			/* IPv6地址结构体 */
    u_int32_t sin6_scope_id;			/* scope ID,尚处于试验阶段 */
}

struct in6_addr
{
    unsigned char sa_addr[16];			/* IPv6地址，使用网络字节序表示 */
}
```

**==所有的专用的socket地址（包括socket_storage）类型的变量在使用的时候，都需要强转为socket类型sockaddr的变量==**

#### IP地址转换函数

下面3个函数用于点分十进制字符串表示的IPv4地址和用网络字节序整数表示的IPv4地址之间的转换

```cpp
#include <arp/inet.h>
/* 将点分十进制的IPv4地址转换为网络字节序表示的IPv4地址，失败时返回INADDR_NONE*/
in_addr_t inet_addr(const char* strptr);		
/* 将点分十进制的IPv4转换为网络字节序并保存在inp中,成功返回1，失败返回0 */
int inet_aton(const char* cp,struct in_addr* inp);
/* 将网络字节序表示的IPv4地址转换为点分十进制，但是因为函数内部是用static变量存储结果，因此是不可重入的 */
char* inet_ntoa(struct in_addr in);
```

下面更新的函数同样可以完成上述函数的功能，不过同时适用于IPv6和IPv4：

```cpp
#include <arp/inet.h>
/* 将字符串形式的IP地址（点分十进制的IPv4或者16进制形式的IPv6）src转换为网络字节序形式的IP地址，
 * 存储在dst中，af用来表明是IPv4还是IPv6，可以是AF_INET或者AF_INET6
 * 成功返回1，失败返回0并且设置errno */
int inet_pton(int af, const char* src, void* dst);

/* 前三个参数与上面的函数相同，第四个参数是存储单元的大小,返回字符串形式的IP地址，
 * 成功返回存储单元的地址，失败返回nullptr并且设置errno */
const char* inet_ntop(int af, const void* src, char* dst, socklen_t cnt);
```



### 创建socket：socket()

```cpp
#include <sys/types.h>
int socket(int domain, int type, int protocal);

```

- domain:

    使用哪个底层协议族，可选的参数：PF_INET，PF_INET6，PF_UNIX

 * type:

    是定服务类型，可选参数：SOCKE_STRAM（流服务TCP协议），SOCK_UGRAM（数据报UDP），或者与

    SOCK_NONBLOCK(设置socket为非阻塞的)和SOCK_CLOEXEC(调用fork创建子进程使关闭该socket)相与得的值；

 * protocal:设置为0，表示使用默认，其实已被前面两个参数确定

 * 调用成功返回1，失败返回0

### 命名socket： bind()

上述创建socket指定了地址族，但是并没有指定使用该地址族中的具体哪一个socket地址。

将一个socket与socket地址绑定称为给socket命名。

在服务器程序中，我们通常要命名socket，因为只有命名后客户端程序才能知道如何连接服务器程序。

在客户端程序中，通常不需要命名socket，即采用匿名方式，使用操作系统自动分配的socket地址。

socket绑定的系统调用如下：

```cpp
#include <sys/types.h>
#include <sys/socket.h>
int bind(int socketfd, const struct sockaddr* my_addr, socklen_t addrlen);
```

`bind`将`my_addr`所指的socket地址分配给未命名的`sockfd`文件描述符,`addrlen`参数指定该`socket`的地址的长度。

bind成功时返回0，失败则返回1，并设置errno，常见的errno是：

- `EACCES`:被绑定的地址是受保护的，仅超级用户能访问。普通用户将端口绑定为熟知端口时（0~1023），会报此错误；
- `EADDRINUSE`:被绑定的地址正在使用中；

### 监听socket：listen()

被命名的socket不能马上连接客户端，需要如下的系统调用来创建一个监听队列以存放待处理的客户端请求：

```cpp
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

- `sockfd`:指定被监听的socket；

- `backlog`:提示内核监听队列的最大长度。内核监听队列的最大长度如果超过backlog，那么服务器将不在受理新的客户连接，客户端也会受到`ECONNREFUSED`错误信息；

    ==只表示处于完全连接(ESTABLISHED)状态的socket的上限，处于半连接(SYN_RCVD)状态的socket的上限值则由`/proc/sys/net/ipv4/tcp_max_syn_backlog`内核参数定义。==




***创建监听队列：***

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>
#include <assert.h>
#include <stdio.h>
#include <string.h>

static bool stop = false;
static void handle_term( int sig )
{
    stop = true;
}

int main( int argc, char* argv[] )
{
    signal( SIGTERM, handle_term );

    if( argc <= 3 )
    {
        printf( "usage: %s ip_address port_number backlog\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];			/* 字符串形式的IP地址 */
    int port = atoi( argv[2] );			/* 将字符串形式的端口号转为整数类型 */
    int backlog = atoi( argv[3] );

    int sock = socket( PF_INET, SOCK_STREAM, 0 );	/* 创建socket */
    assert( sock >= 0 );							/* 判断是否创建成功 */

    struct sockaddr_in address;						/* 声明一个IPv4类型的地址 */
    bzero( &address, sizeof( address ) );			/* 置0 */
    address.sin_family = AF_INET;					/* 初始化地址族协议 */
    inet_pton( AF_INET, ip, &address.sin_addr );	/* 将字符串形式的IP地址转换为二进制形式，并赋值给address的成员 */
    address.sin_port = htons( port );				/* 将使用主机字节序表示的port转换为网络字节序 */

    /* 命名socket */
    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    /* 监听socket 创建监听队列 */
    ret = listen( sock, backlog );
    assert( ret != -1 );

    while ( ! stop )
    {
        sleep( 1 );
    }

    close( sock );
    return 0;
}

```

### 接受连接：accept()

下面的系统调用从listen监听队列中接收一个连接：

```cpp
#include <sys/types.h>
#include <sys/socket.h>
int accept(int sockfd, struct sockkaddr* adr, socklen_t* addrlen);
```

**`sockfd`:**执行过listen系统调用的监听socket[^1]，**`addr`:**用来获取被接受连接的远端socket地址(**==客户端socket地址==**)，该socket地址的长度由**`addrlen`**参数给出。

**`accept`**函数成功时返回一个新的连接socket，该socket唯一的标识了被接受的这个连接，服务器可以通过读写该socket来与被接受连接对应的客户端通信。

**`accept`**失败时返回-1并设置errno。

**==需要注意：accept只是从监听队列中取出一个连接请求，而不管连接处于何种状态==**



***接受连接示例代码：***

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <assert.h>
#include <stdio.h>
#include <string.h>


int main( int argc, char* argv[] )
{

    if( argc <= 3 )
    {
        printf( "usage: %s ip_address port_number backlog\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

	struct sockaddr_in address;				/* create an IPv4 address */
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0 );

	int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

    /* 暂停20秒等待客户端完成相关操作 */
	sleep(20);
	struct sockaddr_in client;					/* 客户端socket地址 */
	socklen_t client_addrlength = sizeof(client);
	int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);
	if(connfd <= 0)
	{
		printf("errno is %d\n", errno);
	}
	else
	{
		char remote[INET_ADDRSTRLEN];
        /* inet_ntop函数将二进制形式的IP地址转换为字符串形式，并处存在remote中，返回内存地址 */
		printf("connected with ip:%s and port:%d\n",inet_ntop(AF_INET, \
					&client.sin_addr, remote, INET_ADDRSTRLEN), ntohs(client.sin_port));
		close(connfd);
	}
	close(sock);
	return 0;

}

```



[^1]:把执行过listen调用、处于LISTEN状态的socket称为监听socket.



### 发起连接：connect()

客户端需要通过以下系统调用主动与服务器进行连接：

```cpp
#include <sys/types.h>
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr* serv_addr, socklen_t addrlen);
```

**`sockfd`**:有socket系统调用返回一个socket。

**`serv_addr`**:服务器监听socket的socket地址。

**`addrlen`**:指定这个地址的长度。

**`connect`**成功时返回0；==一旦成功的建立了连接，**`sockfd`**就唯一的标识了该地址==，客户端就可以读写该`sockefd`来与服务器进行通信。

**`connect`**失败返回-1，并设置errno。其中两种常见的errno：

- `ECONNREFUSED`:目标端口不存在，连接被拒绝；
- `ETIMEDOUT`:连接超时。



### 关闭连接：close()

关闭一个连接其实就是关闭该连接对应的socket。可以通过如下文件描述符关闭一个socket：

```cpp
#include <unistd.h>
int close(int fd);
```

**`fd`**就是指socket标识符，==不过close并非立即关闭这个socket，而是将fd的引用计数-1，只有当fd的引用计数变为0时，才会关闭连接==。

一次`fork`系统调用默认将父进程中打开的socket的引用计数+1，因此必须要在子进程和父进程都调用close函数，才可以真正关闭一个连接。



如果需要立即关闭一个连接（不是将引用计数-1），可以使用如下的系统调用：

```cpp
#include <sys/socket.h>
int shutdown(int sockfd, int howto);
```

**`sockfd`**就是待关闭的socket连接。

**`howto`**参数决定了shutdown的行为，可取以下值中的一个：

****

<table border="1" cellspacing="1" cellpadding="1" align="center"> 
    <caption>
        <strong>
            <big>howto参数的可选值</big>
        </strong>
    </caption>
    <tr>
        <td align="center"><b>可选值</b></td>
        <td align="center"><b>含义</b></td>
    </tr>
    <tr>
        <td align="center">SHUT_RD</td>
        <td align="center">关闭sockfd的读操作。应用程序不能再针对sockfd文件描述符执行读的操作，并且该socket接收缓冲区的数据会被丢弃</td>
    </tr>
    <tr>
    	<td align="center">SHUT_WR</td>
        <td align="center">关闭sockfd的写操作。sockfd的发送缓冲区的的数据会在真正关闭连接之前会全部发送出去，应用程序不可在针对该sockfd文件描述符进行写操作。这种情况下，连接处于<b>半关闭状态</b></td>
    </tr>
    <tr>
    	<td align="center">SHUT_RDWR</td>
        <td align="center">同时关闭sockfd的读和写操作。</td>
    </tr>



**`shutdown`**成功返回0，失败返回-1并设置errno。



### 数据读写

#### TCP数据读写

用于TCP流数据读写的系统调用是：

```cpp
#include <sys/types.h>
#include <sys/socket.h>

/* recv读取sockfd上的数据，buf和len分别指定读缓冲区的位置和大小。
   flags参数：通常设置为0.
   recv成功时返回实际读取到的数据长度，因此可能小于我们期望的长度len，因此可能需要多    次调用recv.recv可能返回0，意味着这个对方已经关闭这个通信连接。recv失败返回-1并设    置errno。
*/
ssize_t recv(int sockfd, void* buf, size_t len, int flags);


/*
  send函数往sockfd上写数据，buf和len分别指定缓冲区的位置和大小。
  send成功时返回实际写入的数据的长度，
  失败时返回-1并设这errno
*/
ssize_t send(int sockfd,const void* buf,size_t len, int flags);
```

> flags参数为数据收发提供了额外的控制，可以去以下参数中的一个或几个的逻辑或.

<style>
    td,th{
        display:table-cell;
        vertical-align:middle;
        text-align:center;
    }
</style>
<div align="center">
    <table border="1">
        <caption style="font-size:large;font-weight:700">flags参数的可选值</caption>
        <tr style="font-size:medium;font-weight:400" align="center">
        	<th>选项名</th>
        	<th>含义</th>
        	<th>send</th>
        	<th>recv</th>
        </tr>
        <tr style="line-height: 100%;">
        	<td>MSG_CONFIRM</td>
            <td>指示数据链路层持续监听对方的回应，直到得到恢复。<br />它仅能用于SOCK_DGRAM和SOCK_RAW类型的socket。</td>
            <td>Y</td>
            <td>N</td>
        </tr>
        <tr>
        	<td>MSG_DONTROUTE</td>
            <td>不查看路由表，直接将数据报发送给本局域网中的主机。<br/>这表示发送者确切的直到目标主机就在本网络上。</td>
            <td>Y</td>
            <td>N</td>
        </tr>
        <tr>
        	<td>MSG_DONTWAIT</td>
            <td>对socket的此次操作是非阻塞的。</td>
            <td>Y</td>
            <td>N</td>
        </tr>
        <tr>
        	<td>MSG_MORE</td>
            <td>告诉内核驱动程序还有更多数据要发送，内核将超时等待新数据写入TCP缓冲区后一并发送。<br/>这样可避免TCP发送过多小的报文段，从而提高传输效率。</td>
            <td>Y</td>
            <td>N</td>
        </tr>
        <tr>
        	<td>MSG_WAITALL</td>
            <td>读操作仅在读取到足够的字节之后才返回。</td>
            <td>N</td>
            <td>Y</td>
        </tr>
        <tr>
        	<td>MSG_PEEK</td>
            <td>窥探读缓存中的数据，此次读操作不会导致读缓存中的数据被清除。</td>
            <td>N</td>
            <td>Y</td>
        </tr>
        <tr>
        	<td>MSG_OOB</td>
            <td>发送或接收紧急数据</td>
            <td>Y</td>
            <td>Y</td>
        </tr>
        <tr>
        	<td>MSG_NOSIGNAL</td>
            <td>往读端关闭的管道或socket连接中写数据时不引发SIGPIPE信号</td>
            <td>Y/td>
            <td>N</td>
        </tr>
    </table>
</div></h3>


***发送带外数据示例代码==客户端==：***

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    /* IPv4地址 */
    struct sockaddr_in server_address;
    bzero( &server_address, sizeof( server_address ) );
    server_address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &server_address.sin_addr );
    server_address.sin_port = htons( port );

    /* 创建socket */
    int sockfd = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sockfd >= 0 );
    /* 测试连接 */
    if ( connect( sockfd, ( struct sockaddr* )&server_address, sizeof( server_address ) ) < 0 )
    {
        printf( "connection failed\n" );
    }
    else
    {
        printf( "send oob data out\n" );
        /* 要发送的紧急数据 */
        const char* oob_data = "abc";
        const char* normal_data = "123";
        send( sockfd, normal_data, strlen( normal_data ), 0 );
        send( sockfd, oob_data, strlen( oob_data ), MSG_OOB );
        send( sockfd, normal_data, strlen( normal_data ), 0 );
    }

    close( sockfd );
    return 0;
}
```



***接收带外数据示例代码==服务器==：***

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

#define BUF_SIZE 1024

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    /* IPv4地址 */
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    /* 创建socket */
    int sock = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sock >= 0 );

    /* 命名socket */
    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( sock, 5 );
    assert( ret != -1 );

    /* 客户端IP地址 */
    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof( client );
    int connfd = accept( sock, ( struct sockaddr* )&client, &client_addrlength );
    if ( connfd < 0 )
    {
        printf( "errno is: %d\n", errno );
    }
    else
    {
        char buffer[ BUF_SIZE ];

        memset( buffer, '\0', BUF_SIZE );
        ret = recv( connfd, buffer, BUF_SIZE-1, 0 );
        printf( "got %d bytes of normal data '%s'\n", ret, buffer );

        memset( buffer, '\0', BUF_SIZE );
        ret = recv( connfd, buffer, BUF_SIZE-1, MSG_OOB );
        printf( "got %d bytes of oob data '%s'\n", ret, buffer );

        memset( buffer, '\0', BUF_SIZE );
        ret = recv( connfd, buffer, BUF_SIZE-1, 0 );
        printf( "got %d bytes of normal data '%s'\n", ret, buffer );

        close( connfd );
    }

    close( sock );
    return 0;
}
```



#### UDP数据读写

socket接口编程中使用于UDP数据读写的系统调用是：

```cpp
#include <sys/types.h>
#include <sys/socket.h>
/* recvfrom读取UDP缓冲区上的数据，buf和len分别指定位置和大小。
 * 因为UDP没有连接的概念，因此每次读取都需要获取发送端的地址，即sock_addr所指的内容，
 * addrlen指明地址长度。*/
ssize_t recvfrom(int sockfd, void* buf, size_t len, int flags, struct sockaddr* 							src_addr,sockelen_t* addrlen);


/* sendto往sockfd上写数据，buf和len分别指定了位置和大小。
 * dst_addr指明目标主机的socket地址，addrlen指明地址长度。 */
ssize_t sendto(int sockfd, const void* buf, size_t len, int flags, struct sockaddr* 					dst_addr, socklen_t* addrlen);
```



#### 通用数据读写函数

socket编程接口还提供了一组通用的数据读写系统调用，不仅能用于UDP数据读写，还可以用于TCP数据读写。

```cpp
#include <sys/socket.h>
ssize_t recvmsg(int sockfd, struct msghdr* msg, int flags);
ssize_t sendmsg(int sockfd, struct msghdr* msg, int flags);

/* msghdr结构体的定义如下： */
struct msghdr
{
    void* msg_name;				/* socket address */
    socklen_t mag_namelen;		/* length of socket address */
    struct * msg_iov;		/* 分散的内存块 */
    int msg_iovlen;				/* 分散内存块的数量 */
    void* msg_control;			/* 指向辅助数据的起始位置 */
    socklen_t msg_controllen;	/* 辅助数据的大小 */
    int msg_flags;				/* 复制函数中的flags参数，并在调用过程中更新 */
};
/* 对于TCP连接，msg_name字段必须被设置为0；*/


/* iovec结构体的定义如下：*/
struct iovec
{
    void* iov_base;				/* 内存起始地址 */
    size_t iov_len;				/* 这块内存的长度 */
};

```





### 带外标记

 Linux 内核检测到TCP 紧急标志时，将通知应用程序有带外数据需要接收。

内核通知应用程序带外数据到达的两种方式：

- I/O复用产生的异常事件
- `SIGURG`信号

通过如下的系统调用，可以使得应用程序获取带外数据在流数据中的位置：

```cpp
#include <sys/socket.h>
int sockatmark(int sockfd);
/* sockatmark判断sockfd是否处于带外标记，即下一个被读取到的数据是否是带外数据。
 * 如果是，sockatmark放回1，此时我们可以用带MSG_OOB标志的recv调用来接收带外数据。
 * 如果不是，则sockatmark返回-1 
 */
```





### 地址信息函数

如下的系统调用用来获取一个连接socket的本端的socket地址和远端的socket地址：

```cpp
#include <sys/socket.h>
/* getsockname返回sockfd对应的本端socket地址，并储存在address中，长度储存在addr_len中。
 * 如果长度比addr_len还长，那么socket地址将被截断。
 * 成功返回0，失败返回-1并设置errno。
 */
int getsockname(int sockfd, struct sockaddr* address, socklen_t* addr_len);


/* getpeername获取sockfd对应的远端socket地址，并储存在address中，长度储存在addr_len中。
 * 如果长度比addr_len还长，那么socket地址将被截断。
 * 成功返回0，失败返回-1并设置errno。
 */
int getpeername(int sockfd, struct sockaddr* address, socklen_t* addr_len);
```





### socket 选项

设置和读取socket文件描述符的系统调用：

```cpp
#include <sys/socket.h>
int getsockopt(int sockfd, int level, int option_name, void* option_value,
              socklen_t* restrict option_len);
int setsockopt(int sockfd,int level, int option_name, const void* option_value,
              socklen_t* restrict option_len); )
/* sockfd参数指定被操作的目标socket。
 * level参数指定操作哪个协议的选项。
 * option_name则指定选项的名字
 * option_value被操作选项的值
 * option_len被操作选项的长度
 * 成功返回0，失败返回-1并设置errno
 */
```



<style>
    td,th{
        display:table-cell;
        vertical-align:middle;
        text-align:center;
    }
</style>
<div align="center">
    <p style="font-size:large;font-weight:700">
        socket选项
    </p>
    <table border="1">
        <tr height="50">
        	<th>level</th>
            <th>option_name</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
        	<td rowspan="14">SOL_SOCKET(通用socket选项，与协议无关)</td>
            <td>SO_DEBUG</td>
            <td>int</td>
            <td>打开调试信息</td>
        </tr>
        <tr>
        	<td>SO_REUSEADDR</td>
            <td>int</td>
            <td>重用本地地址</td>
        </tr>
        <tr>
        	<td>SO_TYPE</td>
            <td>int</td>
            <td>获取socket类型</td>
        </tr>
        <tr>
        	<td>SO_ERROR</td>
            <td>int</td>
            <td>获取并清除socket错误状态</td>
        </tr>
        <tr>
        	<td>SO_DONTROUTE</td>
            <td>int</td>
            <td>不查看路由表，直接将数据发送给本地局域网络内的主机</td>
        </tr>
        <tr>
        	<td>SO_RCVBUF</td>
            <td>int</td>
            <td>TCP接收缓冲区大小</td>
        </tr>
        <tr>
        	<td>SO_SNDBUF</td>
            <td>int</td>
            <td>TCP发送缓冲区大小</td>
        </tr>
        <tr>
        	<td>SO_KEEPALIVE</td>
            <td>int</td>
            <td>发送周期性保活报文以保持连接</td>
        </tr>
        <tr>
        	<td>SO_OOBINLINE</td>
            <td>int</td>
            <td>接收到的带外数据将存留在普通数据的输入队列中（在线存留）。<br/>此时不能使用带MSG_OOB标志的读操作来读取数据.</td>
        </tr>
        <tr>
        	<td>SO_LINGER</td>
            <td>linger</td>
            <td>若有数据待发送，则延迟关闭</td>
        </tr>
        <tr>
        	<td>SO_RCVLOWAT</td>
            <td>int</td>
            <td>TCP接收缓存区低水位标记</td>
        </tr>
        <tr>
        	<td>SO_SNDLOWAT</td>
            <td>int</td>
            <td>TCP发送缓存区高水位标记</td>
        </tr>
        <tr>
        	<td>SO_RCVTIMEO</td>
            <td>timeval</td>
            <td>接收数据超时</td>
        </tr>
         <tr>
        	<td>SO_SNDTIMEO</td>
            <td>timeval</td>
            <td>发送数据超时</td>
        </tr>
         <tr>
        	<td rowspan="2">IPPROTO_IP<br/>IPv4选项</td>
            <td>IP_TOS</td>
            <td>int</td>
            <td>服务类型</td>
        </tr>
         <tr>
        	<td>IP_TTL</td>
            <td>int</td>
            <td>存活时间</td>
        </tr>
         <tr>
        	<td rowspan="4">IPPROTO_IPV6<br/>IPv6选项</td>
            <td>IPV6_NEXTHOP</td>
            <td>sockaddr_in6</td>
            <td>下一跳IP地址</td>
        </tr>
         <tr>
        	<td>IPV6_RECVPKTINFO</td>
            <td>int</td>
            <td>接收分组信息</td>
        </tr>
         <tr>
        	<td>IPV6_DONTFRAG</td>
            <td>int</td>
            <td>禁止分片</td>
        </tr>
         <tr>
        	<td>IPV6_RECVTCLASS</td>
            <td>int</td>
            <td>接收通信类型</td>
        </tr>
        <tr>
        	<td rowspan="2">IPPROTO_TCP<br/>TCP选项</td>
            <td>TCP_MAXSEG</td>
            <td>int</td>
            <td>TCP最大报文段大小</td>
        </tr>
        <tr>
        	<td>TCP_NODELAY</td>
            <td>int</td>
            <td>禁止Nagle算法</td>
        </tr>
    </table>
</div>




##### SO_REFUSEADDER选项

服务器可以通过设置socket选项SO_REFUSEADDR来强制使用被处于TIME_WAIT状态的连接占用的socket地址，具体方法如下：

```cpp
int sock = socket( PF_INET,SoCK_STREAM,0 );		/* 创建socket */
assert ( sock >= 0 );

int reuse = 1;
/* 设置socket选项为SO_REFUSEADDR */
setsockopt( sock,SOL_SOCKET,SO_REUSEADDR,&reuse,sizeof(reuse ));
/* 创建一个IPV4的地址 */
struct sockaddr_in address;
bzero( taddress,sizeof ( address ));
address.sin_family = AF_INET;
inet_pton( AF_INET,ip, &address.3in_addr );
address.sin_port = htons ( port );
int ret = bind( sock,( struct sockaddr*)saddress,sizeof( address ));
```





##### SO_RCVBUF 和 SO_SNDBUF选项

==当我们使用setsockopt设置这两个选项之后，系统都会将其加倍，并且不能小于某个值。==

***修改TCP发送缓冲区的客户端程序：***

```cpp
#include <sys/socket.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>

#define BUFFER_SIZE 512

int main( int argc, char* argv[] )
{
    if( argc <= 3 )
    {
        printf( "usage: %s ip_address port_number send_bufer_size\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    struct sockaddr_in server_address;
    bzero( &server_address, sizeof( server_address ) );
    server_address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &server_address.sin_addr );
    server_address.sin_port = htons( port );

    int sock = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sock >= 0 );

    int sendbuf = atoi( argv[3] );
    int len = sizeof( sendbuf );
    setsockopt( sock, SOL_SOCKET, SO_SNDBUF, &sendbuf, sizeof( sendbuf ) );
    getsockopt( sock, SOL_SOCKET, SO_SNDBUF, &sendbuf, ( socklen_t* )&len );
    printf( "the tcp send buffer size after setting is %d\n", sendbuf );

    if ( connect( sock, ( struct sockaddr* )&server_address, sizeof( server_address ) ) != -1 )
    {
        char buffer[ BUFFER_SIZE ];
        memset( buffer, 'a', BUFFER_SIZE );
        send( sock, buffer, BUFFER_SIZE, 0 );
    }

    close( sock );
    return 0;
}

```





***修改TCP接收缓冲区的服务器程序：***

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

#define BUFFER_SIZE 1024

int main( int argc, char* argv[] )
{
    if( argc <= 3 )
    {
        printf( "usage: %s ip_address port_number receive_buffer_size\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int sock = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sock >= 0 );
    
    int recvbuf = atoi( argv[3] );
    int len = sizeof( recvbuf );
    setsockopt( sock, SOL_SOCKET, SO_RCVBUF, &recvbuf, sizeof( recvbuf ) );
    getsockopt( sock, SOL_SOCKET, SO_RCVBUF, &recvbuf, ( socklen_t* )&len );
    printf( "the receive buffer size after settting is %d\n", recvbuf );

    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( sock, 5 );
    assert( ret != -1 );

    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof( client );
    int connfd = accept( sock, ( struct sockaddr* )&client, &client_addrlength );
    if ( connfd < 0 )
    {
        printf( "errno is: %d\n", errno );
    }
    else
    {
        char buffer[ BUFFER_SIZE ];
        memset( buffer, '\0', BUFFER_SIZE );
        while( recv( connfd, buffer, BUFFER_SIZE-1, 0 ) > 0 ){}
        close( connfd );
    }

    close( sock );
    return 0;
}
```



##### SO_RECVLOWAT 和 SO_SNDLOWAT选项

默认情况下，TCP接收缓冲区的低水位标记和TCP发送缓冲区的低水位标记都设置为1字节。



##### SO_LINGER 选项

该选项用于控制close系统调用在关闭TCP连接时的行为。

默认情况下，使用close系统调用时，close将立即返回，TCP模块负责将TCP发送缓冲区滞留的数据发送给对方。

设置SO_LINGER选项时，需要给setsockopt(getsockopt)函数传递一个linger类型的结构体，定义如下：

```cpp
#include <sys/socket.h>
struct linger
{
    int l_onoff;			/* 开启（非0）还是关闭（为0）该选项 */
    int l_linger;			/* 滞留时间 */
};
```

根据linger结构体中两个变量的值，close会产生如下三个行为之一：

1. l_onoff = 0；该选项为关闭状态，close使用默认方式关闭连接；

2. l_onoff ！= 0，l_linger=0：此时，close系统调用立即返回，TCP模块将清空发送缓冲区中的数据，同时给对方发送一个复位报文段。

3. l_onoff != 0,l_linger ！= 0：此时close的行为取决于以下两个条件：

    - 被关闭的socket对应的TCP发送缓冲区是否还有残留的数据；
    - socket是阻塞的，还是非阻塞的；

    对于阻塞的 socket，close 将等待一段长为l_linger 的时间，直到 TCP 模块发送完所有残留数据并得到对方的确认。如果这段时间内 TCP 模块没有发送完残留数据并得到对方的确认，那么 close 系统调用将返回-1并设置 errno 为EWOULDBLOCK。如果 socket 是非阻塞的，close 将立即返回，此时我们需要根据其返回值和 errno 来判断残留数据是否已经发送完毕。



### 网络信息API

#### gethostbyname和gethostbyaddr

`gethostbyname`函数根据主机名获取主机的完整信息，该函数通常在本地的`/etc/hosts`配置文件中查找主机，如果查不到再去访问DNS服务器。

`gethostbyaddr`函数根据IP地址获取主机的完整信息。

两个函数的原型如下：

```c
#include <netdb.h>
struct hostnet* gethostbyname(const char* name);
/*
 * addr：主机地址
 * len：地址长度
 * type：地址类型，可选AF_INET，AF_INET6
 */
struct hostnet* gethostbyaddr(const void* addr, size_t len, int type);

/* hostnet结构体的原型如下： */
struct hostnet
{
    char* h_name;				/* 主机名 */
    char** h_aliases;			/* 主机别名，可能有多个 */
    int h_addrtype;				/* 地址类型(地址族) */
    int h_length;				/* 地址长度 */
    char** h_addr_list;			/* 按网络字节序列出的主机IP地址表 */
};
```

#### getservbyname和getservbyport

这两个数实际上都是读取`/etc/services`文件来获取服务器的完整信息。

函数原型：

```cpp
#include <netdb.h>
struct servnet* getservbyname(const char* name, const char* proto);
struct servnet* getservbyaddr(int port, const char* proto);


/* servnet结构体原型： */
struct servnet
{
    char* s_name;				/* 服务器名称 */
    char** s_aliases;			/* 服务器别名 */
    int s_port;					/* 端口号 */
    char* s_proto;				/* 服务类型，TCP/UDP */
};
```

#### getaddrinfo

`getaddrinfo`函数既能通过主机名获得IP地址（内部使用gethostbyname函数），也可通过服务名获得端口号（内部使用getservbyname函数）。

函数原型：

```cpp
#include <netdb.h>
int getaddrinfo(const char* hostname, const char* service,const struct addrinfo* hints, 				struct addrinfo** result);
/* 
 * hostname可以接收主机名，也可以接受点分十进制表示的IP地址。
 * service可以接收服务名，也可以接受十进制表示的端口号。
 * hints是应用程序给getaddrinfo给的一些提示，以便对getaddrinfo的输出进行精准的控制，可以为NULL。
 * resul指向一个链表，用于保存结果。
 */


/* addrinfo结构体的原型：*/
struct addrinfo
{
    int ai_flags;
    int ai_family;					/* 地址族 */
    int ai_socktype;				/* 服务类型，SOCK_STREAM/SOCK_DGRAM */
    int ai_protocol;				
    socklen_t ai_addrlen;			/* socket地址的长度 */
    char* ai_canoname;				/* 主机别名 */
    struct socktaddr* ai_addr;		/* socket地址 */
    stuct addrinfo* ai_next;		/* 指向下一个ai_next指向的对象 */
}
```

<p align="center" style="font-size:large;font-weight:700"> ai_flags成员 </p>

<a href="https://sm.ms/image/ucpJTW8jnamFQBf" target="_blank"><img src="https://i.loli.net/2021/07/15/ucpJTW8jnamFQBf.png" ></a>





==**注意addinfo会隐式的分配内存，因此需要手动释放内存，避免内存泄漏。**==

```cpp
#include <netdb.h>
void freeaddrinfo(struct addinfo* res);
```



#### getnameinfo

`etnameinfo `函数能通过 socket 地址同时获得以字符串表示的主机名（内部使用的是`gethostbyaddr `函数）和服务名（内部使用的是 `getservbyport` 函数）.

函数原型：

```cpp
#include <netdb.h>
int getnameinfo(const struct sockaddr* sockaddr, socklen_t addrlen, char* host, 
               socklen_t hostlen, char* serv,socklen_t servlen,int flags);
```



<p align="center" style="font-size:large;font-weight:700"> flags参数 
	<a href="https://sm.ms/image/k3ZjeQS6MvBbiqV" target="_blank"><img src="https://i.loli.net/2021/07/15/k3ZjeQS6MvBbiqV.png" ></a>
</p>







==`gatnameinfo`和`getaddrinfo`函数成功时返回0，失败时返回错误码:==

可以使用如下函数将错误码转为字符串：

```cpp
#include <netdb.h>
const char* gai_strerror( int error );
```



<a href="https://sm.ms/image/edVRN7DmUMaP46p" target="_blank"><img src="https://i.loli.net/2021/07/15/edVRN7DmUMaP46p.png" ></a>





## 高级I/O函数

高级函数大致可以分为3类：

- 用于创建文件描述符的函数，如pipe函数，dup/dup2函数；
- 用于读写数据的函数，如readv/writev,sendfile,mmap/munmap,splice,tee函数；
- 用于控制I/O行为的函数。

### pipe函数

pipe函数用于创建一个管道，以实现**进程间通信**

函数原型:

```cpp
#include <unisted.h>
int pipe(int fd[2]);
```

==pipe函数的参数是一个包含2个int类型的数组指针，该函数成功返回0，并将一对打开的文件描述符填入其指向的数组。如果失败，则返回-1，并设置errno。==

​		通过 pipe 函数创建的这两个文件描述符 fd【0】 和 fd【1】分别构成管道的两端，往 fd【1】写人的数据可以从 fd【0】 读出。并且，fd【0】 只能用于从管道读出数据，fd【1】 则只能用于往管道写入数据，而不能反过来使用。如果要实现双向的数据传输，就应该使用两个管道。默认情况下，这一对文件描述符都是阻塞的。此时如果我们用 read系统调用来读取一个空的管道，则 read将被阻塞，直到管道内有数据可读;如果我们用 write 系统调用来往一个满的管道中写入数据，则 write 亦将被阻塞，古到管道有足够多的空闲空间可用。但如果应用程序将 fd【0】 和 fd【1】 都设置为非阻塞的，则 read 和 write 会有不同的行为。如果管道的写端文件描述符 fd【1】 的引用计数减少至 0，即没有任何进程需要往管道中写入数据，则针对该管道的读端文件描述符 fd【0】的 read操作将返回 0，即读取到了文件结束标记（End Of File，EOF）;反之，如果管道的读端文件描述符 fd【0】的引用计数减少至 0，即没有任何进程需要从管道读取数据，则针对该管道的写端文件描述符 fd【1】的 write操作将失败，并引发 SIGPIPE 信号。



创建双向pipe的函数`socketpair`，其函数原型如下：

```cpp
#include <sys/types.h>
#include <sys/socket.h>
int socketpair(int doamin, int type, int protocol, int fd[2]);
```

`soketpair`函数前三个参数和socket调用完全一样，不过`domain`只能使用本地协议族AF_UNIX，最后一个参数则和pipe系统调用参数一样。

`socketpair`函数成功时返回0，失败时返回-1并设置errno。





### dup/dup2函数

使用dup/dup2函数可以将标准输入重定向到某一个文件，或者把标准输出重定向到一个网络连接（**CGI编程**）。

其函数原型如下：

```cpp
#include <unisted.h>
int dup(int file_descriptor);
int dup2(int file_dexcriptor_one, int file_descriptor_two);
```

dup 函数创建一个新的文件描述符，该新文件描述符和原有文件描述符file_descriptor 指向相同的文件、管道或者网络连接。==**并且 dup 返回的文件描述符总是取系统当前可用的最小整数值**==。

dup2 和 dup 类似，不过它将返回第一个不小于file_descriptor_two的整数值。

dup 和dup2 系统调用失败时返回 -1 并设置 errno。



<p align="center" style="font-size:larger;font-weight:900">CGI服务器</p>

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int sock = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sock >= 0 );

    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( sock, 5 );
    assert( ret != -1 );

    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof( client );
    int connfd = accept( sock, ( struct sockaddr* )&client, &client_addrlength );
    if ( connfd < 0 )
    {
        printf( "errno is: %d\n", errno );
    }
    else
    {
        close( STDOUT_FILENO );			/* 关闭标准输出文件描述符 STDOUT_FILENO */
        /* 复制 socket 文件描述符 connfd。因为 dup 总是返回系统中最小的可用文件描述符，
         * 所以它的返回值实际上是 1，即之前关闭的标准输出文件描述符的值 */
        dup( connfd );
        printf( "abcd\n" );		/* abcd会被发送到与客户端连接对应的socket上，不会被打印到屏幕 */
        close( connfd );
    }

    close( sock );
    return 0;
}


```

==CGI服务器基本工作原理:==

我们先关闭标准输出文件描述符 STDOUT_FILENO（其值是1），然后复制 socket 文件描述符 connfd。因为 dup 总是返回系统中最小的可用文件描述符，所以它的返回值实际上是 1，即之前关闭的标准输出文件描述符的值。这样一来，服务器输出到标准输出的内容（这里是"abcd"）就会直接发送到与客户连接对应的 socket上，因此 printf调用的输出将被客户端获得（而不是显示在服务器程序的终端上）。





### readv/writev函数

readv 函数将数据从文件描述符读到分散的内存块中，即分散读;

 writev 函数则将多块分散的内存数据一并写入文件描述符中，即集中写。

函数原型：

```cpp
#include <sys/uio.h>
ssize_t readv(int fd, const struct iovec* vector, int count);
ssize_t writecv(int fd, const struct iovec* vector, int count);
```

readv 和 writev 在成功时返回读出/写入 fd的字节数，失败则返回-1 并设置 errno. `iovec`结构体的详情请见[通用数据读写函数](# 通用数据读写函数)

<p style="font-size:large;font-weight:700" align="center">web服务器上的集中写：</p>

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>

#define BUFFER_SIZE 1024
static const char* status_line[2] = { "200 OK", "500 Internal server error" };

int main( int argc, char* argv[] )
{
    if( argc <= 3 )
    {
        printf( "usage: %s ip_address port_number filename\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];			/* IP地址 */
    int port = atoi( argv[2] );			/* 端口号 */
    const char* file_name = argv[3];	/* 文件名 */

    /* 初始化socket */
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    /* 创建socket */
    int sock = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sock >= 0 );

    /* 绑定socket */
    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    /* 创建监听队列 */
    ret = listen( sock, 5 );
    assert( ret != -1 );

    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof( client );
    int connfd = accept( sock, ( struct sockaddr* )&client, &client_addrlength );
    if ( connfd < 0 )
    {
        printf( "errno is: %d\n", errno );
    }
    else
    {
        /* 用于保存HTTP应答的状态行，头部字段，和一个空行的缓存区 */
        char header_buf[ BUFFER_SIZE ];
        memset( header_buf, '\0', BUFFER_SIZE );
        /* 用于存放目标文件内容的应用程序缓存 */
        char* file_buf;
        /* 用于获取目标文件的属性，比如是否是目录，文件大小等 */
        struct stat file_stat;
        /* 记录目标文件是否是有效文件 */
        bool valid = true;
        /* 缓存区header_buf目前使用了多少字节的空间 */
        int len = 0;
        /* 这里使用的是
        int stat(const char *filename, struct stat *buf);
        函数，filename 代表文件或者文件夹的路径  
        buf代表stat结构体，保存文件信息,详情见下文。
        */
        if( stat( file_name, &file_stat ) < 0 )			/* 目标文件不存在 */
        {
            valid = false;
        }
        else
        {
            if( S_ISDIR( file_stat.st_mode ) )		/* 目标文件是目录 */
            {
                valid = false;
            }
            else if( file_stat.st_mode & S_IROTH )		/* 当前用户有读取目标文件的权限 */
            {
                /* 动态分配缓存区file_buf，并指定其大小为目标文件的大小file_stat.st_size+1，然后将目标文件读入缓存区中 */
                int fd = open( file_name, O_RDONLY );
                file_buf = new char [ file_stat.st_size + 1 ];
                memset( file_buf, '\0', file_stat.st_size + 1 );
                if ( read( fd, file_buf, file_stat.st_size ) < 0 )
                {
                    valid = false;
                }
            }
            else
            {
                valid = false;
            }
        }
        /* 如果目标文件有效，则发送正常的HTTP应答 */ 
        if( valid )
        {
            ret = snprintf( header_buf, BUFFER_SIZE-1, "%s %s\r\n", "HTTP/1.1", status_line[0] );
            len += ret;
            ret = snprintf( header_buf + len, BUFFER_SIZE-1-len, 
                             "Content-Length: %d\r\n", file_stat.st_size );
            len += ret;
            ret = snprintf( header_buf + len, BUFFER_SIZE-1-len, "%s", "\r\n" );
            struct iovec iv[2];
            iv[ 0 ].iov_base = header_buf;
            iv[ 0 ].iov_len = strlen( header_buf );
            iv[ 1 ].iov_base = file_buf;
            iv[ 1 ].iov_len = file_stat.st_size;
            ret = writev( connfd, iv, 2 );
        }
        else
        {
            ret = snprintf( header_buf, BUFFER_SIZE-1, "%s %s\r\n", "HTTP/1.1", status_line[1] );
            len += ret;
            ret = snprintf( header_buf + len, BUFFER_SIZE-1-len, "%s", "\r\n" );
            send( connfd, header_buf, strlen( header_buf ), 0 );
        }
        close( connfd );
        delete [] file_buf;
    }

    close( sock );
    return 0;
}
```

```cpp
struct stat 
{
        mode_t     st_mode;       //文件对应的模式，文件，目录等
        ino_t      st_ino;        //inode节点号
        dev_t      st_dev;        //设备号码
        dev_t      st_rdev;       //特殊设备号码
        nlink_t    st_nlink;      //文件的连接数
        uid_t      st_uid;        //文件所有者
        gid_t      st_gid;        //文件所有者对应的组
        off_t      st_size;       //普通文件，对应的文件字节数
        time_t     st_atime;      //文件最后被访问的时间
        time_t     st_mtime;      //文件内容最后被修改的时间
        time_t     st_ctime;      //文件状态改变时间
        blksize_t  st_blksize;    //文件内容对应的块大小
        blkcnt_t   st_blocks;     //伟建内容对应的块数量
};

```

> stat结构体中的st_mode 则定义了下列数种情况：
>     S_IFMT   			0170000    	文件类型的位遮罩
>     S_IFSOCK 			0140000    	scoket
>     S_IFLNK 			0120000     符号连接
>     S_IFREG 			0100000     一般文件
>     S_IFBLK 			0060000     区块装置
>     S_IFDIR 			0040000     目录
>     S_IFCHR 			0020000     字符装置
>     S_IFIFO 			0010000     先进先出
>     S_ISUID 			04000     	文件的(set user-id on execution)位
>     S_ISGID 			02000     	文件的(set group-id on execution)位
>     S_ISVTX 			01000     	文件的sticky位
>     S_IRUSR(S_IREAD) 	00400     	文件所有者具可读取权限
>     S_IWUSR(S_IWRITE)	00200     	文件所有者具可写入权限
>     S_IXUSR(S_IEXEC) 	00100     	文件所有者具可执行权限
>     S_IRGRP 			00040       用户组具可读取权限
>     S_IWGRP 			00020       用户组具可写入权限
>     S_IXGRP 			00010       用户组具可执行权限
>     S_IROTH 			00004       其他用户具可读取权限
>     S_IWOTH 			00002       其他用户具可写入权限
>     S_IXOTH 			00001       其他用户具可执行权限
>
> 上述的文件类型在POSIX中定义了检查这些类型的宏定义：
>     S_ISLNK (st_mode)    判断是否为符号连接
>     S_ISREG (st_mode)    是否为一般文件
>     S_ISDIR (st_mode)    是否为目录
>     S_ISCHR (st_mode)    是否为字符装置文件
>     S_ISBLK (s3e)        是否为先进先出
>     S_ISSOCK (st_mode)   是否为socket
>
> 若一目录具有sticky位(S_ISVTX)，则表示在此目录下的文件只能被该文件所有者、此目录所有者或root来删除或改名，在linux中，最典型的就是这个/tmp目录啦。







### sendfile函数

sendfile 函数在两个文件描述符之间直接传递数据（完全在内核中操作），从而避免了内核缓冲区和用户缓冲区之间的数据拷贝，效率很高，这被称为零拷贝。sendfile 函数的定义如下∶

```cpp
#include <sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t* offset, size_t count);
```

in_fd 参数是待读出内容的文件描述符，out_fd参数是待写入内容的文件描述符。offset参数指定从读入文件流的哪个位置开始读，如果为空，则使用读入文件流默认的起始位置。count参数指定在文件描述符 in_fd和 out_fd 之间传输的字节数。sendfile 成功时返回传输的字节数，失败则返回-1并设置 errno。

该函数的 man手册明确指出，in_fd必须是一个支持类似 mmap 函数的文件描述符，即它必须指向真实的文件，不能是 socket 和管道; 而 out_fd则必须是一个socket。



<p style="font-size:large;font-weight:700" align="center">用sendfile函数传输文件</p>

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/sendfile.h>

int main( int argc, char* argv[] )
{
    if( argc <= 3 )
    {
        printf( "usage: %s ip_address port_number filename\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );
    const char* file_name = argv[3];

    int filefd = open( file_name, O_RDONLY );           /* 打开文件 */
    assert( filefd > 0 );
    struct stat stat_buf;
    fstat( filefd, &stat_buf );                                       /* 确定文件状态 */


    /* IPv4 结构体*/
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );


    /* 创建socket */
    int sock = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sock >= 0 );

    /* 绑定socket */
    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    /* 创建监听队列*/
    ret = listen( sock, 5 );
    assert( ret != -1 );

    /* 连接客户端*/
    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof( client );
    int connfd = accept( sock, ( struct sockaddr* )&client, &client_addrlength );
    if ( connfd < 0 )
    {
        printf( "errno is: %d\n", errno );
    }
    else
    {
        sendfile( connfd, filefd, NULL, stat_buf.st_size );
        close( connfd );
    }

    close( sock );
    return 0;
}
```







### mmap函数和munmap函数

mmap 函数用于申请一段内存空间。我们可以将这段内存作为进程间通信的共享内存，也可以将文件直接映射到其中。munmap 函数则释放由 mmap 创建的这段内存空间。它们的定义如下∶

```cpp
#include <sys/mman.h>

void* mmap(void* start, size_t lenght, int prot, int flags, int fd, off_t offset);

int munmap(void* start, size_t length);
```

start 参数允许用户使用某个特定的地址作为这段内存的起始地址。如果它被设置成NULL，则系统自动分配一个地址。length 参数指定内存段的长度。prot 参数用来设置内存段的访问权限。它可以取以下几个值的按位或∶

- PROT_READ:内存段可读；
- PROT_WRITE:内存段可写；
- PROT_EXEC:内存段可执行；
- PROT_NONE:内存段不能被访问；

flags 参数控制内存段内容被修改后程序的行为。它可以被设置为以下表中的某些值（这里仅列出了常用的值）的按位或（==其中MAP_SHARED和 MAP_PRIVATE是互斥的，不能同时指定==）。

<img align="center">src="https://i.loli.net/2021/09/10/V31IQceiSMGwnsE.png"/>

fd参数是被映射文件对应的文件描述符。它一般通过 open 系统调用获得。offset 参数设置从文件的何处开始映射（对于不需要读入整个文件的情况）。

mmap 函数成功时返回指向目标内存区域的指针，失败则返回 MAP_FAILED（（void*）-1）并设置 errno。munmap 函数成功时返回 0，失败则返回-1并设置 errno。



### splice函数

splice 函数用于在两个文件描述符之间移动数据，也是零拷贝操作。定义如下：

```cpp
#include <fcntl.h>
ssize_t splice(int fd_in, loff_t* off_in, int fd_out, loff_t* off_out, size_t len,
               unsigned int flags);
```



fd_in 参数是待输入数据的文件描述符。如果 fd_in 是一个管道文件描述符，那么 off_in参数必须被设置为NULL。如果 fd_in不是一个管道文件描述符（比如 socket），那么 off_in表示从输入数据流的何处开始读取数据。此时，若off_in 被设置为 NULL，则表示从输入数据流的当前偏移位置读入;若 off_in不为 NULL，则它将指出具体的偏移位置。fd_out/off_out参数的含义与 fd_in/off_in 相同，不过用于输出数据流。len参数指定移动数据的长度,flags 参数则控制数据如何移动，它可以被设置为表6-2 中的某些值的按位或。

<img >src="https://i.loli.net/2021/09/10/XtBacWnZwhzlUK6.png" />

使用splice 函数时，==fd_in 和 fd_out必须至少有一个是管道文件描述符==。splice 函数调用成功时返回移动字节的数量。它可能返回 0，表示没有数据需要移动，这发生在从管道中读取数据（fd in 是管道文件描述符）而该管道没有被写入任何数据时。splice 函数失败时返回 -1 并设置 errno。常见的 errno 如表 6-3 所示。

<img > src="https://i.loli.net/2021/09/10/tgCz8h74qcMTusV.png"/>

<p style="font-size:large;font-weight:700" align="center">使用splice函数实现回射服务器</p>

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int sock = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sock >= 0 );

    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( sock, 5 );
    assert( ret != -1 );

    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof( client );
    int connfd = accept( sock, ( struct sockaddr* )&client, &client_addrlength );
    if ( connfd < 0 )
    {
        printf( "errno is: %d\n", errno );
    }
    else
    {
        int pipefd[2];			/* 创建管道，因为splice函数的其中一个文件描述符必须是管道 */
        assert( ret != -1 );
        ret = pipe( pipefd );
        ret = splice( connfd, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE ); 
        assert( ret != -1 );
        ret = splice( pipefd[0], NULL, connfd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE );
        assert( ret != -1 );
        close( connfd );
    }

    close( sock );
    return 0;
}
```







### tee函数

tee 函数在两个管道文件描述符之间复制数据，也是零拷贝操作。它不消耗数据，因此源文件描述符上的数据仍然可以用于后续的读操作。tee 函数的原型如下∶

```cpp
#include <fcntl.h>
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);
```

该函数的参数的含义与 splice 相同（==但 fd in 和 fd out 必须都是管道文件描述符==）。tee函数成功时返回在两个文件描述符之间复制的数据数量（字节数）。返回0表示没有复制任何数据。tee 失败时返回 -1 并设置 errno。

<p style="font-size:large;font-weight:700" align="center">同时输出数据到屏幕和文件</p>

```cpp
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>

int main( int argc, char* argv[] )
{
	if ( argc != 2 )
	{
		printf( "usage: %s <file>\n", argv[0] );
		return 1;
	}
	int filefd = open( argv[1], O_CREAT | O_WRONLY | O_TRUNC, 0666 );
	assert( filefd > 0 );

	int pipefd_stdout[2];
        int ret = pipe( pipefd_stdout );
	assert( ret != -1 );

	int pipefd_file[2];
        ret = pipe( pipefd_file );
	assert( ret != -1 );


    /* 将标准输入内容输入管道 pipefd_stdout */
	ret = splice( STDIN_FILENO, NULL, pipefd_stdout[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE );
	assert( ret != -1 );
    /* 将管道 pipefd_stdout 的输出复制到管道 pipefd_file 的输入端 */
	ret = tee( pipefd_stdout[0], pipefd_file[1], 32768, SPLICE_F_NONBLOCK ); 
	assert( ret != -1 );
    /*将管道pipefd_file的输出定向到文件描述符flefd上，从而将标准输入的内容写入文件 */
	ret = splice( pipefd_file[0], NULL, filefd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE );
	assert( ret != -1 );
    /* 将管道pipefd_stdout的输出定向到标准输出，其内容和写入文件的内容完全一致 */
	ret = splice( pipefd_stdout[0], NULL, STDOUT_FILENO, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE );
	assert( ret != -1 );

	close( filefd );
    close( pipefd_stdout[0] );
    close( pipefd_stdout[1] );
    close( pipefd_file[0] );
    close( pipefd_file[1] );
	return 0;
}

```





### fcntl函数

fcntl函数，提供了对文件描述符的各种控制操作。另外一个常见的控制文件描述符属性和行为的系统调用是 ioctl，而且ioctl 比 fcntl能够执行更多的控制。但是，对于控制文件描述符常用的属性和行为，fcntl 函数是由 POSIX 规范指定的首选方法。函数定义如下：

```cpp
#include <fcntl.h>
int fcntl(int fd, int cmd, ...);
```

fd 参数是被操作的文件描述符，cmd 参数指定执行何种类型的操作。根据操作类型的不同，该函数可能还需要第三个可选参数 arg。fcntl 函数支持的常用操作及其参数如表6-4 所示。

<style>
    td,th{
        display:table-cell;
        vertical-align:middle;
        text-align:center;
    }
</style>
<table align="center" border="1">
   <caption style="font-size:large;font-weight:700">fcntl支持的常用操作及其参数</caption>
   <tr style="font-size:medium;font-weight:900" align="center">
        <td>操作分类</td>
        <td>操作</td>
        <td>含义</td>
        <td>第三个参数的类型</td>
        <td>成功时的返回值</td>
    </tr>
    <tr>
        <td rowspan="2">复制文件描述符</td>
        <td>F_DUPFD</td>
        <td>创建一个新的文件描述符，其值大于或等于arg</td>
        <td>long</td>
        <td>新创建的文件描述符的值</td>
    </tr>
    <tr>
        <td>F_DUPFD_CLOEXEC</td>
        <td>与F_DUPFD相似，不过在创建文件描述符时设置其close_on_exec标志</td>
        <td>long</td>
        <td>新创建的文件描述符的值</td>
    </tr>
    <tr>
        <td rowspan="2">获取和设置文件描述符的标志</td>
        <td>F_GETFD</td>
        <td>获取fd的标志</td>
        <td>无</td>
        <td>fd的标志</td>
    </tr>
    <tr>
        <td>F_SETFD</td>
        <td>设置fd的标志</td>
        <td>long</td>
        <td>0</td>
    </tr>
    <tr>
        <td rowspan="2">获取和设置文件描述符的状态标志</td>
        <td>F_GETFL</td>
        <td>获取fd的状态标志，包括可由open系统调用设置的标志（O_APPEND，O_CREATE）和访问模式（O_RDONLY，O_WRONLY，O_RDWR）</td>
        <td>void</td>
        <td>fd的状态标志</td>
    </tr>
    <tr>
        <td>F_SETFL</td>
        <td>设置fd的状态标志，有些模式不可被修改</td>
        <td>long</td>
        <td>0</td>
    </tr>
    <tr>
        <td rowspan="4">管理信号</td>
        <td>F_GETOWN</td>
        <td>获得SIGIO和SIGURG信号的宿主进程的PID或者进程组的GID</td>
        <td>无</td>
        <td>信号的宿主进程的PID或GID</td>
    </tr>
    <tr>
        <td>F_SETOWN</td>
        <td>设置SIGIO和SIGURG信号的宿主进程的PID或者进程组的GID</td>
        <td>long</td>
        <td>0</td>
    </tr>
    <tr>
        <td>F_GETSIG</td>
        <td>获取当应用程序被通知fd可写或可读时，是那个信号通知该事件的</td>
        <td>无</td>
        <td>信号值，0表示SIGIO</td>
    </tr>
    <tr>
        <td>F_SETSIG</td>
        <td>设置当fd可读或可写时，系统应该触发哪个信号来通知应用程序</td>
        <td>long</td>
        <td>0</td>
    </tr>
    <tr>
        <td rowspan="2">操作管道容量</td>
        <td>F_SETPIPE_SZ</td>
        <td>设置由fd指定的管道的容量</td>
        <td>long</td>
        <td>0</td>
    </tr>
    <tr>
        <td>F_GETPIPE_SZ</td>
        <td>获取由fd指定的管道的容量</td>
        <td>无</td>
        <td>管道容量</td>
    </tr>
</table>



<p align="center" style="font-size:large;font-weight:700">设置文件描述符为非阻塞的</p>

```cpp
int setnonblocking( int fd )
{
    int old_option = fcnt1（ fd，F_GETFL ）;/*获取文件描述符旧的状态标志 */
    int new_option = old_option | O_NONBLOCK;/*设置非阻塞标志*/ 
    fcntl ( fd,F_SETFL,new_option );
    return old_option;			/* 返回文件描述符旧的状态标志，以便日后恢复该状态标志*/
}

```

此外，SIGIO 和 SIGURG 这两个信号与其他 Linux 信号不同，它们必须与某个文件描述符相关联方可使用;当被关联的文件描述符可读或可写时，系统将触发 SIGIO 信号;当被关联的文件描述符（而且必须是一个 socket）上有带外数据可读时，系统将触发 SIGURG信号。将信号和文件描述符关联的方法，就是使用 fcntl 函数为目标文件描述符指定宿主进程或进程组，那么被指定的宿主进程或进程组将捕获这两个信号。使用 SIGIO 时，还需要利用fcntl 设置其O ASYNC 标志。





## Linux 服务器程序规范



### 1. 日志

#### 1.1 Linux系统日志

 `rsyslogd `守护进程既能接收用户进程输出的日志，又能接收内核日志。用户进程是通过调用 syslog 函数生成系统日志的。该函数将日志输出到一个 UNIX本地域 socket类型（AFUNIX）的文件/devlog 中，rsyslogd则监听该文件以获取用户进程的输出。内核日志在老的系统上是通过另外一个守护进程 rklogd 来管理的，rsyslogd 利用额外的模块实现了相同的功能。内核日志由 printk 等函数打印至内核的环状缓存（ring buffer）中。环状缓存的内容直接映射到 /proc/kmsg 文件中。rsyslogd则通过读取该文件获得内核日志。

rsyslogd 守护进程在接收到用户进程或内核输入的日志后，会把它们输出至某些特定的日志文件。默认情况下，调试信息会保存至/var/log/debug 文件，普通信息保存至/var/logmessages 文件，内核消息则保存至/var/log/kern.log文件。不过，日志信息具体如何分发，可以在 rsyslogd 的配置文件中设置。rsyslogd 的主配置文件是/etc/rsyslog.conf，其中主要可以设置的项包括∶ 内核日志输入路径，是否接收 UDP 日志及其监听端口（默认是 514，见 /etc/services 文件），是否接收 TCP 日志及其监听端口，日志文件的权限，包含哪些子配置文件（比如 /etc/rsyslog.d/*.conf）。rsyslogd 的子配置文件则指定各类日志的目标存储文件。

![Linux系统日志.png](https://i.loli.net/2021/09/11/CRAMo8KJnSLZrfy.png)





#### 1.2 syslog函数

syslog函数定义：

```cpp
#include <syslog.h>
void syslog( int priority, const char* message, ...);
```

priority 参数是所谓的设施值与日志级别的按位或。设施值的默认值是LOG_USER。日志级别有如下几个∶

```cpp
#include <syslog.h>
#define LOG_EMERG	0	/* 系统不可用 */
#define LOG_ALERT	1	/* 报警，需要立即采取行动 */
#define LOG_CRIT	2	/* 非常严重的情况 */
#define LOG_ERR		3	/* 错误 */
#define LOGWARNING	4	/* 警告 */
#define LOG_NOTICE	5	/* 通知 */
#define LOG_INFO	6	/* 信息 */
#define LOG_DEBUG	7	/* 调试 */
```

下面这个函数可以改变 syslog 的默认输出方式，进一步结构化日志内容∶

```cpp
#include <syslog.h>
void openlog(const char* ident, int log opt, int facility);
```

ident 参数指定的字符串将被添加到日志消息的日期和时间之后，它通常被设置为程序的名字。logopt 参数对后续 syslog 调用的行为进行配置，它可取下列值的按位或∶

```cpp
#define LOG_PID		0x01	/* 在日志程序中包含PID */
#define LOG_CONS	0x02	/* 如果消息不能写入到文件，则打印到终端 */
#define LOG_ODELAY	0x03	/* 延迟调用日志功能，直到第一次调用syslog */
#define LOG_NDELAY	0x04	/* 不延迟打开日志功能 */
```

facility 参数可用来修改 syslog 函数中的默认设施值。



此外，日志的过滤也很重要。程序在开发阶段可能需要输出很多调试信息，而发布之后我们又需要将这些调试信息关闭。解决这个问题的方法并不是在程序发布之后删除调试代码（因为日后可能还需要用到），而是简单地设置日志掩码，使日志级别大于日志掩码的日志信息被系统忽略。下面这个函数用于设置 syslog 的日志掩码∶

```cpp
#include <syslog.h>
int setlogmask(int maskpri);
```

maskpri 参数指定日志掩码值。该函数始终会成功，它返回调用进程先前的日志掩码值。



最后，不要忘了使用如下函数关闭日志功能∶

```cpp
#include <syslog.h>
void closelog();
```





### 2. 用户信息

#### 2.1 UID,EUID,GID,EGID

下面这一组函数可以获取和设置当前进程的真实用户 ID（UID）、有效用户 ID（EUID）、真实组 ID（GID）和有效组 ID（EGID）∶

```cpp
#include <sys/types.h>
#include <unistd.h>

uld_t getuid() ;	/* 获取真实用户ID */
uid_t geteuid() ;	/* 获取有效用户ID */ 
gid_t getgid();		/* 获取真实组ID */ 
gid_t getegid();	/* 获取有效组ID */
int setuid(uid_t uid);	/* 设置真实用户ID */ 
int seteuid(uid_t uid);	/* 设置有效用户ID */
int setgid(gid_t gid);	/* 设置真实组ID */ 
int setegid(gid_t gid);	/* 设置有效组ID */
```

**需要指出的是，一个进程拥有两个用户 ID∶UID和 EUID。EUID存在的目的是方便资源访问∶ 它使得运行程序的用户拥有该程序的有效用户的权限。**



#### 2.2 切换用户

```cpp
static bool switch_to_user( uid_t user_id, gid_t gp_id )
{
    /* 确保目标用户不是root */
    if ( ( user_id == 0 ) && ( gp_id == 0 ) )
    {
        return false;
    }

    /* 确保当前用户是合法用户	 */
    gid_t gid = getgid();
    uid_t uid = getuid();
    if ( ( ( gid != 0 ) || ( uid != 0 ) ) && ( ( gid != gp_id ) || ( uid != user_id ) ) )
    {
        return false;
    }

    if ( uid != 0 )
    {
        return true;
    }

    if ( ( setgid( gp_id ) < 0 ) || ( setuid( user_id ) < 0 ) )
    {
        return false;
    }

    return true;
}
```





### 3.进程间关系

#### 3.1 进程组

Linux下每个进程都隶属于一个进程组，因此它们除了PID信息外，还有进程组 ID（PGID）。我们可以用如下函数来获取指定进程的 PGID∶

```cpp
#include <unistd.h>
pid_t getpgid(pid_t pid);
```

该函数成功时返回进程 pid 所属进程组的 PGID，失败则返回-1 并设置 errno。

下面的函数用于设置 PGID∶

```cpp
#include <unistd.h>
int setpqid(pid_t pid, pid_t pgid);
```

该函数将 PID为 pid的进程的 PGID 设置为 pgid。

如果 pid和 pgid相同，则由 pid指定的进程将被设置为进程组首领;

如果 pid为 0，则表示设置当前进程的 PGID 为 pgid;

如果 pgid为 0，则使用 pid作为目标 PGID。

setpgid 函数成功时返回 0，失败则返回-1并设置errno。

**一个进程只能设置自己或者其子进程的 PGID。并且，当子进程调用exec 系列函数后，我们也不能再在父进程中对它设置 PGID。**





#### 3.2 会话

一些有关联的进程组将形成一个会话（session）。下面的函数用于创建一个会话∶

```cpp
#include <unistd.h>
pid_t setsid(void);
```

该函数不能由进程组的首领进程调用，否则将产生一个错误。对于非组首领的进程，调用该函数不仅创建新会话，而且有如下额外效果∶

> - 调用进程成为会话的首领，此时该进程是新会话的唯一成员。
> - 新建一个进程组，其 PGID 就是调用进程的 PID，调用进程成为该组的首领。
> - 调用进程将甩开终端（如果有的话）。

该函数成功时返回新的进程组的 PGID，失败则返回 -1 并设置 erno。

Linux 进程并未提供所谓会话 ID（SID）的概念，但 Linux 系统认为它等于会话首领所在的进程组的 PGID，并提供了如下函数来读取 SID∶

```cpp
#include <unistd.h>
pid_t getsid( pid_t pid );
```



#### 3.3 用ps命令查看进程间关系

> **ps -o pid,ppid, pgid, sid, comm | less **





### 4.系统资源限制

Linux 系统资源限制可以通过如下一对函数来读取和设置∶

```cpp
#include <sys/resource.h>
int getrlimit (int resource, const struct rlimt* rlimit);
int setrlimit (int resource, const struct rlimt* rlimit);
```

rlimit参数是 rlimit 结构体类型的指针，rlimit 结构体的定义如下∶

```cpp
struct rlimit
{
    rlim_t rlim_cur;
    rlim_t rlim_max;
};
```

rlim_t 是一个整数类型，它描述资源级别。rlim_cur 成员指定资源的软限制，rlim_max成员指定资源的硬限制。软限制是一个建议性的、最好不要超越的限制，如果超越的话，系统可能向进程发送信号以终止其运行。

硬限制一般是软限制的上限。普通程序可以减小硬限制，而只有以 root 身份运行的程序才能增加硬限制。此外，我们可以使用 ulimit 命令修改当前 shell 环境下的资源限制（软限制或/和硬限制），这种修改将对该 shell启动的所有后续程序有效。我们也可以通过修改配置文件来改变系统软限制和硬限制，而且这种修改是永久的。

![getlimit和setlimit.png](https://i.loli.net/2021/09/11/ntqaFjodXyJLP9p.png)





### 5.改变工作目录和根目录

获取进程当前工作目录和改变进程工作目录的函数分别是∶

```cpp
#include <unistd.h>
char* getcwd( char* buf,size_t size ); 
int chdir( const char* path );
```

buf 参数指向的内存用于存储进程当前工作目录的绝对路径名，其大小由 size 参数指定。如果当前工作目录的绝对路径的长度（再加上一个空结束字符"\0"）超过了size，则 getcwd将返回 NULL，并设置 errno 为 ERANGE。如果 buf 为 NULL并且 size 非 0，则 getcwd可能在内部使用 malloc 动态分配内存，并将进程的当前工作目录存储在其中。如果是这种情况，则我们必须自己来释放 getcwd在内部创建的这块内存。getcwd 函数成功时返回一个指向目

标存储区（buf指向的缓存区或是 getcwd 在内部动态创建的缓存区）的指针，失败则返回NULL并设置 errno。

chdir 函数的 path 参数指定要切换到的目标目录。它成功时返回 0，失败时返回 -1 并设置errno。

改变进程根目录的函数是 chroot，其定义如下∶

```cpp
#include <unistd.h>

int chroot ( const char* path );
```



path 参数指定要切换到的目标根目录。它成功时返回 0，失败时返回 -1 并设置 errno。

chroot并不改变进程的当前工作目录，所以调用 chroot 之后，我们仍然需要使用 chdir（"/"）来将工作目录切换至新的根目录。改变进程的根目录之后，程序可能无法访问类似 /dey 的文件（和目录），因为这些文件（和目录）并非处于新的根目录之下。不过好在调用chroot 之后，进程原先打开的文件描述符依然生效，所以我们可以利用这些早先打开的文件描述符来访问调用chroot 之后不能直接访问的文件（和目录），尤其是一些日志文件。此外，只有特权进程才能改变根目录。





### 6. 服务器程序后台化

```cpp
bool daemonize()
{
    /* 创建子进程，关闭父进程，这样可以使程序在后台运行 */
    pid_t pid = fork();
    if ( pid < 0 )
    {
        return false;
    }
    else if ( pid > 0 )
    {
        exit( 0 );
    }

    /*设置文件权服掩码。当进程创建新文件（使用open（ const char*pathname，int flags， mode_t mode ）系统调用）时，文件的权限将是 mode 占 0777 */
    umask( 0 );

    /* 创建新的会话，设置本进程为进程组的首领*/
    pid_t sid = setsid();
    if ( sid < 0 )
    {
        return false;
    }

    
    if ( ( chdir( "/" ) ) < 0 )
    {
        /* Log the failure */
        return false;
    }

    /* 关闭标准输入设备、标准输出设备和标准错误输出设备 */
    close( STDIN_FILENO );
    close( STDOUT_FILENO );
    close( STDERR_FILENO );

    /* 关闭其他已经打开的文件描述符，代码省略*/
	/* 将标准输入、标准输出和标准错误输出都定向到 /dev/nul1 文件 */
    open( "/dev/null", O_RDONLY );
    open( "/dev/null", O_RDWR );
    open( "/dev/null", O_RDWR );
    return true;
}
```

以下的库函数可以完成上述功能：

```cpp
#include <unistd.h>
int daemon ( int nochdir,int noclose );
```

其中，nochdir 参数用于指定是否改变工作目录，如果给它传递 0，则工作目录将被设置为"/"（根目录），否则继续使用当前工作目录。noclose 参数为0时，标准输入、标准输出和标准错误输出都被重定向到 /dev/null 文件，否则依然使用原来的设备。该函数成功时返回0，失败则返回-1并设置 errno。















## 高性能服务器程序框架

### 1.服务器模型

#### 1.1 C/S模型

C/S 模型的逻辑很简单。服务器启动后，首先创建一个（或多个）监听 socket，并调用 bind 函数将其绑定到服务器感兴趣的端口上，然后调用 listen 函数等待客户连接。服务器稳定运行之后，客户端就可以调用 connect 函数向服务器发起连接了。由于客户连接请求是随机到达的异步事件，服务器需要使用某种 I/O模型来监听这一事件。

<img src="https://i.loli.net/2021/09/11/oA3NPQjSG27LfvC.png" alt="C-S模型.png" style="zoom:80%;" />

上图中服务器使用的是 I/O 复用技术之一的 select 系统调用。当监听到连接请求后，服务器就调用 accept 函数接受它，并分配一个逻辑单元为新的连接服务。逻辑单元可以是新创建的子进程、子线程或者其他。图中，服务器给客户端分配的逻辑单元是由 fork 系统调用创建的子进程。逻辑单元读取客户请求，处理该请求，然后将处理结果返回给客户端。客户端接收到服务器反馈的结果之后，可以继续向服务器发送请求，也可以立即主动关闭连接。如果客户端主动关闭连接，则服务器执行被动关闭连接。至此，双方的通信结束。需要注意的是，服务器在处理一个客户请求的同时还会继续监听其他客户请求，否则就变成了效率低下的串行服务器了（必须先处理完前一个客户的请求，才能继续处理下一个客户请求）。





#### 1.2 P2P模型

P2P（Peer to Peer，点对点）模型比 C/S 模型更符合网络通信的实际情况。它摒弃了以服务器为中心的格局，让网络上所有主机重新回归对等的地位。

P2P 模型使得每台机器在消耗服务的同时也给别人提供服务，这样资源能够充分、自由地共享。云计算机群可以看作 P2P 模型的一个典范。

但 P2P 模型的==缺点==也很明显：当用户之间传输的请求过多时，网络的负载将加重。



### 2. 服务器编程框架

<p align="center" style="font-size:larger;font-weight:900">服务器基本框架</p>

<a href="https://sm.ms/image/2OsuA46kdFZWlCX" target="_blank"><img src="https://i.loli.net/2021/09/11/2OsuA46kdFZWlCX.png" ></a>





<div align="center" style="font-size:larger;font-weight:900">服务器基本模块的功能描述</div>

<style>
    td,th{
        display:table-cell;
        vertical-align:middle;
        text-align:center;
    }
</style>
<div>
    <table align="center" border="1">
    	<tr style="font-size:medium;font-weight:900" align="center">
        	<td>模块</td>
        	<td>单个服务器程序</td>
        	<td>服务器集群</td>
    	</tr>
        <tr>
            <td>I/O处理单元</td>
            <td>处理客户连接，读写网络数据</td>
            <td>作为接入服务器，实现负载均衡</td>
        </tr>
        <tr>
        	<td>逻辑单元</td>
            <td>业务进程或线程</td>
            <td>逻辑服务器</td>
        </tr>
        <tr>
        	<td>网络存储单元</td>
            <td>本地数据库，缓存或文件</td>
            <td>数据库服务器</td>
        </tr>
        <tr>
            <td>请求队列</td>
            <td>各单元之间的通信方式</td>
            <td>各服务器之间的永久TCP连接</td>
        </tr>
</div>





==I/O 处理单元是服务器管理客户连接的模块==。它通常要完成以下工作 ∶ 等待并接受新的客户连接，接收客户数据，将服务器响应数据返回给客户端。但是，数据的收发不—定在I/O处理单元中执行，也可能在逻辑单元中执行，具体在何处执行取决干事件处理模式。**对于一个服务器机群来说，I/O 处理单元是一个专门的接入服务器。它实现负载均衡，从所有逻罗辑服务器中选取负荷最小的一台来为新客户服务。**

==一个逻辑单元通常是一个进程或线程==。它分析并处理客户数据，然后将结果传递给 I/O处理单元或者直接发送给客户端（具体使用哪种方式取决于事件处理模式）。**对服务器机群而言，—个逻辑单元本身就是—台逻辑服务器。服务器通常拥有多个逻辑单元，以实现对个客户任务的并行处理。**

==网络存储单元可以是数据库、缓存和文件，甚至是一台独立的服务器==。**但它不是必须的，比如 ssh、telnet 等登录服务就不需要这个单元。**

==请求队列是各单元之间的通信方式的抽象==。I/O 处理单元接收到客户请求时，需要以某种方式通知一个逻辑单元来处理该请求。同样，多个逻辑单元同时访问一个存储单元时，也需要采用某种机制来协调处理竞态条件。请求队列通常被实现为池的一部分，我们将在后面讨论池的概念。**对于服务器机群而言，请求队列是各台服务器之间预先建立的、静态的、永久的 TCP 连接。这种 TCP 连接能提高服务器之间交换数据的效率，因为它避免了动态建立TCP 连接导致的额外的系统开销。**





### 3. I/O模型

==socket 在创建的时候默认是阻塞的==。我们可以给 socket 系统调用的第 2 个参数传递 SOCK NONBLOCK标志，或者通过 fcntl 系统调用的 F SETFL 命令，将其设置为非阻塞的。阻塞和非阻塞的概念能应用于所有文件描述符，而不仅仅是 socket。我们称阻塞的文件描述符为阻塞 I/O，称非阻塞的文件描述符为非阻塞 I/O。

==针对阻塞 I/O 执行的系统调用可能因为无法立即完成而被操作系统挂起，直到等待的事件发生为止==。比如，客户端通过 connect 向服务器发起连接时，connect 将首先发送同步报文段给服务器，然后等待服务器返回确认报文段。如果服务器的确认报文段没有立即到达客户端，则 connect 调用将被挂起，直到客户端收到确认报文段并唤醒 connect 调用。socket 的基础 API 中，可能被阻塞的系统调用包括 accept、send、recv 和 connect。

==针对非阳塞 I/O 执行的系统调用则总是立即返回，而不管事件是否已经发生==。如果事件没有立即发生，这些系统调用就返回 -1，和出错的情况一样。此时我们必须根据 errno 来区分这两种情况。对 accept、send和 recv 而言，事件未发生时 errno 通常被设置成 EAGAIN（意为"再来一次"）或者 EWOULDBLOCK（意为"期望阻塞"）;对 connect 而言，errno 则被设置成 EINPROGRESS（意为"在处理中"）。

**==因此，非阻塞 I/O通常要和其他 I/O通知机制一起使用，比如 I/O 复用和 SIGIO 信号。==**

I/O复用是最常使用的 I/O 通知机制。它指的是，应用程序通过 I/O复用函数向内核注册一组事件，内核通过 I/O复用函数把其中就绪的事件通知给应用程序。Linux上常用的 I/O 复用函数是 select、poll 和 epoll wait。需要指出的是，I/O复用函数本身是阻塞的，它们能提高程序效率的原因在于它们具有同时监听多个 I/O 事件的能力。

SIGIO 信号也可以用来报告 I/O 事件。[fcntl函数中我们讲到](#fcntl函数)，我们可以为一个目标文件描述符指定宿主进程，那么被指定的宿主进程将捕获到 SIGIO 信号。这样，当目标文件描述符上有事件发生时，SIGIO 信号的信号处理函数将被触发，我们也就可以在该信号处理函数中对目标文件描述符执行非阻塞 I/O操作了。

从理论上说，==阻塞 I/O、I/O 复用和信号驱动 I/O都是同步 I/O模型==。因为在这三种 I/O模型中，I/O 的读写操作，都是在I/O事件发生之后，由应用程序来完成的。而 POSIX规范所定义的异步 I/O模型则不同。==对异步 I/O 而言，用户可以直接对 I/O执行读写操作，这些操作告诉内核用户读写缓冲区的位置，以及 I/O 操作完成之后内核通知应用程序的方式。异步 I/O的读写操作总是立即返回，而不论 I/O 是否是阻塞的，因为真正的读写操作已经由内核接管==。也就是说，同步 I/O模型要求用户代码自行执行I/O操作（将数据从内核缓冲区读入用户缓冲区，或将数据从用户缓冲区写入内核缓冲区），而异步 I/O 机制则由内核来执行I/O操作（数据在内核缓冲区和用户缓冲区之间的移动是由内核在"后台"完成的）。你可以这样认为，**==同步 I/O 向应用程序通知的是 I/O 就绪事件，而异步 I/O 向应用程序通知的是 I/O完成事件==**。Linux环境下，aio.h头文件中定义的函数提供了对异步 I/O的支持。

<div>
    <p align="center" style="font-size:larger;font-weight:900">
        <b>I/O模型对比</b>
    </p>
</div>
<style>    
    td,th
    {        
        display:table-cell;     
        vertical-align:middle;     
        text-align:center;   
    }
</style>
<div>    
    <table align="center" border="1">    
        <tr style="font-size:medium;font-weight:900" align="center">
            <td>I/O模型</td>
            <td>读写操作和阻塞阶段</td>
        </tr>
        <tr>
            <td>阻塞I/O</td>
            <td>程序阻塞于读写函数</td>
        </tr>
        <tr>
            <td>I/O复用</td>
            <td>程序阻塞于I/O复用系统调用，但可同时监听多个I/O事件。<br>对I/O本身的操作是非阻塞的。</td>
        </tr>
        <tr>
            <td>SIGIO信号</td>
            <td>信号触发读写就绪事件，用户程序执行读写操作。程序没有阻塞阶段。</td>
        </tr>
        <tr>
            <td>异步I/O</td>
            <td>内核执行读写操作并触发读写完成事件。程序没有阻塞阶段。</td>
        </tr>
    </table>





### 4.两种高效的事件处理模式

#### 4.1 Reactor模式

==Reactor 是这样一种模式，它要求主线程（I/O处理单元，下同）只负责监听文件描述上是否有事件发生，有的话就立即将该事件通知工作线程（逻辑单元，下同）。除此之外，主线程不做任何其他实质性的工作。读写数据，接受新的连接，以及处理客户请求均在工作线程中完成。==

使用同步 I/O 模型（以 epoll_wait 为例）实现的 Reactor 模式的工作流程是∶

1. 主线程往 epoll 内核事件表中注册 socket 上的读就绪事件。
2. 主线程调用epoll_wait 等待 socket 上有数据可读。
3. 当 socket 上有数据可读时，epoll_wait 通知主线程。主线程则将 socket 可读事件放入请求队列。
4. 睡眠在请求队列上的某个工作线程被唤醒，它从 socket 读取数据，并处理客户请求，然后往 epoll 内核事件表中注册该 socket 上的写就绪事件。
5. 主线程调用epoll_wait 等待 socket 可写。
6. 当 socket 可写时，epoll_wait 通知主线程。主线程将 socket 可写事件放入请求队列。
7. 睡眠在请求队列上的某个工作线程被唤醒，它往 socket 上写入服务器处理客户请求的结果。

![reactor模式](https://i.loli.net/2021/09/11/mnUqM1VjAdrw3QK.png)





工作线程从请求队列中取出事件后，将根据事件的类型来决定如何处理它：对于可读事件，执行读数据和处理请求的操作;对于可写事件，执行写数据的操作。因此Reactor 模式中，没必要区分所谓的"读工作线程"和"写工作线程"。





#### 4.2 Proactor模式

与Reactor 模式不同，Proactor 模式将所有 I/O 操作都交给主线程和内核来处理，工作线程仅仅负责业务逻辑。

使用异步 I/O 模型（以 aio_read 和 aio_write 为例）实现的 Proactor 模式的工作流程是∶

1. 主线程调用aio_read 函数向内核注册 socket上的读完成事件，并告诉内核用户读缓冲区的位置，以及读操作完成时如何通知应用程序（这里以信号为例，详情请参考 sigevent的man 手册）。
2. 主线程继续处理其他逻辑。
3. 当 socket上的数据被读入用户缓冲区后，内核将向应用程序发送一个信号，以通知应用程序数据已经可用。
4. 应用程序预先定义好的信号处理函数选择一个工作线程来处理客户请求。工作线程处理完客户请求之后，调用 aio_write 函数向内核注册 socket 上的写完成事件，并告诉内核用户写缓冲区的位置，以及写操作完成时如何通知应用程序（仍然以信号为例）。
5. 主线程继续处理其他逻辑。
6. 当用户缓冲区的数据被写人 socket 之后，内核将向应用程序发送一个信号，以通知应用程序数据已经发送完毕。
7. 应用程序预先定义好的信号处理函数选择一个工作线程来做善后处理，比如决定是否关闭 socket。

![Proactor模式.png](https://i.loli.net/2021/09/11/TApSx835ic9G6Ff.png)

连接 socket 上的读写事件是通过 aio read/aio write 向内核注册的，因此内核将通过信号来向应用程序报告连接 socket 上的读写事件。所以，主线程中的 epoll_wait 调用仅能用来检测监听 socket 上的连接请求事件，而不能用来检测连接 socket 上的读写事件。



#### 4.3 模拟Proactor模式

其原理是：主线程执行数据读写操作，读写完成之后，主线程向工作线程通知这一"完成事件"。那么从工作线程的角度来看，它们就直接获得了数据读写的结果，接下来要做的只是对读写的结果进行逻辑处理。

使用同步 I/O 模型（仍然以epoll wait 为例）模拟出的 Proactor 模式的工作流程如下：

1. 主线程往 epoll 内核事件表中注册 socket 上的读就绪事件。
2. 主线程调用epoll wait 等待 socket 上有数据可读。
3. 当 socket 上有数据可读时，epoll_wait 通知主线程。主线程从socket 循环读取数据，直到没有更多数据可读，然后将读取到的数据封装成一个请求对象并插入请求队列。
4. 睡眠在请求队列上的某个工作线程被唤醒，它获得请求对象并处理客户请求，然后往 epoll 内核事件表中注册 socket 上的写就绪事件。
5. 主线程调用 epoll_wait 等待 socket 可写。
6. 当 socket 可写时，epoll wait通知主线程。主线程往 socket上写人服务器处理客户请求的结果。

![同步IO模拟Proactor.png](https://i.loli.net/2021/09/11/MItNUln8EGyScrF.png)





### 5. 两种高效的并发模式

#### 5.1 半同步/半异步模式（half-sync/half-async）

==首先，半同步/半异步模式中的"同步"和"异步"与前面讨论的 I/O 模型中的"同步"和"异步"是完全不同的概念==。在 I/O模型中，"同步"和"异步"区分的是内核向应用程序通知的是何种 I/O 事件（是就绪事件还是完成事件），以及该由谁来完成 I/O 读写（是应用程序还是内核）。在并发模式中，"同步"指的是程序完全按照代码序列的顺序执行;"异步"指的是程序的执行需要由系统事件来驱动。常见的系统事件包括中断、信号等。

==按照同步方式运行的线程称为同步线程，按照异步方式运行的线程称为异步线程==。显然，异步线程的执行效率高，实时性强，这是很多嵌入式程序采用的模型。但编写以异步方式执行的程序相对复杂，难于调试和扩展，而日不适合干大量的并发。而同步线程则相反，它虽然效率相对较低，实时性较差，但逻辑简单。因此，对干像服务器这种既要求较好的实时性，又要求能同时处理多个客户请求的应用程序，我们就应该同时使用同步线程和异步线程来实现，即采用半同步/半异步模式来实现。

半同步/半异步模式中，同步线程用于处理客户逻辑，相当于[图中](#2. 服务器编程框架)的逻辑单元;异步线程用于处理 I/O事件，相当于[图中](#2. 服务器编程框架)的 I/O 处理单元。异步线程监听到客户请求后，就将其封装成请求对象并插入请求队列中。请求队列将通知某个工作在同步模式的工作线程来读取并处理该请求对象。具体选择哪个工作线程来为新的客户请求服务，则取决于请求队列的设计。比如最简单的轮流选取工作线程的 Round Robin 算法，也可以通过条件变量或信号量来随机地选择一个工作线程。

在服务器程序中，如果结合考虑两种事件处理模式和几种 I/O 模型，则半同步/半异步模式就存在多种变体。其中有一种变体称为半同步/半反应堆（half-sync/half-reactive）模式，如图所示。

![半同步半反应堆.png](https://i.loli.net/2021/09/11/fIdVnFtMb2g4Pjz.png)

==异步线程只有一个，由主线程来充当==。它负责监听所有socket上的事件。如果监听 socket 上有可读事件发生，即有新的连接请求到来，主线程就接受之以得到新的连接 socket，然后往 epoll内核事件表中注册该 socket 上的读写事件。如果连接 socket 上有读写事件发生，即有新的客户请求到来或有数据要发送至客户端，主线程就将该连接 socket 插入请求队列中。所有工作线程都睡眠在请求队列上，当有任务到来时，它们将通过竞争（比如申请互斥锁）获得任务的接管权。这种竞争机制使得只有空闲的工作线程才有机会来处理新任务，这是很合理的。



**==半同步 / 半反应堆模式存在如下缺点∶==**

- 主线程和工作线程共享请求队列。主线程往请求队列中添加任务，或者工作线程从请求队列中取出任务，都需要对请求队列加锁保护，从而白白耗费 CPU 时间。
- 每个工作线程在同一时间只能处理一个客户请求。如果客户数量较多，而工作线程较少，则请求队列中将堆积很多任务对象，~~客户端~~服务器的响应速度将越来越慢。如果通过增加工作线程来解决这一问题，则工作线程的切换也将耗费大量 CPU 时间。



**==更加高效的半同步/半异步模式==**:

![半同步半异步.png](https://i.loli.net/2021/09/13/TaEoPOdwizQ8sZv.png)

==主线程只管理监听 socket，连接 socket 由工作线程来管理==。当有新的连接到来时，主线程就接受之并将新返回的连接 socket 派发给某个工作线程，此后该新 socket上的任何 I/O操作都由被选中的工作线程来处理，直到客户关闭连接。主线程向工作线程派发socket 的最简单的方式，是往它和工作线程之间的管道里写数据。工作线程检测到管道上有数据可读时，就分析是否是一个新的客户连接请求到来。如果是，则把该新 socket上的读写事件注册到自己的 epoll 内核事件表中。

可见，每个线程（主线程和工作线程）都维持自己的事件循环，它们各自独立地监听不同的事件。因此，在这种高效的半同步/半异步模式中，每个线程都工作在异步模式，所以它并非严格意义上的半同步/半异步模式。



#### 5.2 领导者/追随者模式（Leader/Followers）

领导者/追随者模式是多个工作线程轮流获得事件源集合，轮流监听、分发并处理事件的一种模式。在任意时间点，程序都仅有一个领导者线程，它负责监听 I/O事件。而其他线程则都是追随者，它们休眠在线程池中等待成为新的领导者。当前的领导者如果检测到 I/O事件，首先要从线程池中推选出新的领导者线程，然后处理 I/O 事件。此时，新的领导者等待新的 I/O 事件，而原来的领导者则处理 I/O事件，二者实现了并发。

领导者/追随者模式包含如下几个组件∶ 句柄集（HandleSet）、线程集（ThreadSet）、事件处理器（EventHandler）和具体的事件处理器（ConcreteEventHandler）。

![领导者追随者模式的组件.png](https://i.loli.net/2021/09/11/9Za2tfoXGezIEm7.png)

1. 句柄集（HandleSet）：句柄（Handle）用于表示 I/O资源，在 Linux 下通常就是一个文件描述符。句柄集管理众多句柄，它使用wait_for_event 方法来监听这些句柄上的 I/O事件，并将其中的就绪事件通知给领导者线程。领导者则调用绑定到 Handle 上的事件处理器来处理事件。领导者将Handle 和事件处理器绑定是通过调用句柄集中的 register _handle 方法实现的。

2. 线程集（ThreadSet）：这个组件是所有工作线程（包括领导者线程和追随者线程）的管理者。它负责各线程之

    间的同步，以及新领导者线程的推选。线程集中的线程在任一时间必处于如下三种状态之一：

    -  Lcader∶ 线程当前处于领导者身份，负责等待句柄集上的 I/O 事件。
    - Processing： 线程正在处理事件。领导者检测到 I/O 事件之后，可以转移到 Processing状态来处理该事件，并调用 promote new leader 方法推选新的领导者;也可以指定其他追随者来处理事件（Event Handoff），此时领导者的地位不变。当处于Processing状态的线程处理完事件之后，如果当前线程集中没有领导者，则它将成为新的领导者，否则它就直接转变为追随者。
    - Follower∶线程当前处于追随者身份，通过调用线程集的 join 方法等待成为新的领导者，也可能被当前的领导者指定来处理新的任务。

    <div align="center" style="font-size:larger;font-weight:900">领导者/追随者模式中的状态转移</div>

![image.png](https://i.loli.net/2021/09/11/tz9YXnEZTVhHN1o.png)



**==需要注意的是，领导者线程推选新的领导者和追随者等待成为新领导者这两个操作都将修改线程集，因此线程集提供一个成员 Synchronizer 来同步这两个操作，以避免竞态条件。==**

3. 事件处理器和具体的事件处理器:事件处理器通常包含一个或多个回调函数 handle_event。这些回调函数用于处理事件对应的业务逻辑。事件处理器在使用前需要被绑定到某个句柄上，当该句柄上有事件发生时，领导者就执行与之绑定的事件处理器中的回调函数。具体的事件处理器是事件处理器的派生类。它们必须重新实现基类的 handle_event 方法，以处理特定的任务。



<div align="center" style="font-size:larger;font-weight:900">领导者/追随者模式流程</div>

![image.png](https://i.loli.net/2021/09/11/3b4wfSMcUqmtZxo.png)





由于领导者线程自己监听 I/O 事件并处理客户请求，因而领导者/追随者模式不需要在线程之间传递任何额外的数据，也无须像半同步/半反应堆模式那样在线程之间同步对请求队列的访问。但领导者/追随者的一个明显缺点是仅支持一个事件源集合，因此也无法让每个工作线程独立地管理多个客户连接。





### 6.有限状态机

有限状态机应用的一个实例; HTTP 请求的读取和分析。很多网络协议，包括 TCP 协议和 IP 协议，都在其头部中提供头部长度字段。程序根据该字段的值就可以知道是否接收到—个完整的协议头部。但 HTTP 协议并未提供这样的头部长度字段，并且其头部长度变化也很大，可以只有十几字节，也可以有上百字节。根据协议规定，我们判断HTTP 头部结束的依据是遇到一个空行，该空行仅包含一对回车换行符（<CR><LF>）。如果一次读操作没有读入 HTTP请求的整个头部，即没有遇到空行，那么我们必须等待客户继续写数据并再次读入。因此，我们每完成一次读操作，就要分析新读入的数据中是否有空行。不过在寻找空行的过程中，我们可以同时完成对整个 HTTP 请求头部的分析（记住，空行前面还有请求行和头部域），以提高解析 HTTP 请求的效率。代码清单使用主、从两个有限状态机实现了最简单的 HTTP 请求的读取和分析。为了使表述简洁，我们约定，直接称HTTP请求的一行（包括请求行和头部字段）为行。



<div>
    <div align="center" style="font-size:larger;font-weight:900">HTTP请求的读取和分析</div>
     <div style="font-size:larger;font-weight:900">HTTP请求</div>
    <div>
        GET&nbsp;http://www.baidu.com/index.html&nbsp;HTTP/1.0<br/>
		User-Agent:&nbsp;Wget/1.12&nbsp;(linux-gnu)<br/>
		Host:&nbsp;ww.baidu.com<br/>
		Connection:&nbsp;close<br/>
    </div>
</div>

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>

#define BUFFER_SIZE 4096		/* 缓冲区大小 */
/* 主状态机的两种状态：正在分析请求行，正在分析非请求行 */
enum CHECK_STATE { CHECK_STATE_REQUESTLINE = 0, CHECK_STATE_HEADER, CHECK_STATE_CONTENT };
/* 从状态机的三种状态，即行的状态：读取到一个完整的行，行出错，行数据不完整 */
enum LINE_STATUS { LINE_OK = 0, LINE_BAD, LINE_OPEN };
/* HTTP状态码的集合 */
enum HTTP_CODE { NO_REQUEST/*请求不完整,需要继续读取请求 */, GET_REQUEST/*获得了一个完整的客户需求*/, BAD_REQUEST/*客户请求有语法错误 */, FORBIDDEN_REQUEST/* 客户对资源没有足够的访问权限 */, INTERNAL_ERROR/*服务器内部错误 */, CLOSED_CONNECTION/*客户端已关闭连接 */ };

static const char* szret[] = { "I get a correct result\n", "Something wrong\n" };

/* 从状态机，用于解析出一行内容 */
LINE_STATUS parse_line( char* buffer, int& checked_index, int& read_index )
{
    char temp;
    /* checked_index 指向 buffer（应用程序的读缓冲区）中当前正在分析的字节，read_index指向 buffer 中客户数据的尾部的下一字节，buffer 中第0~checkedindex 字节都已分析完毕，第checked index-（read_index-1）字节由下面的循环挨个分析 */
    for ( ; checked_index < read_index; ++checked_index )
    {
        /* 获得当前要分析的字节 */
        temp = buffer[ checked_index ];
        /* 如果当前的字节是"\r"，即回车符，则说明可能读取到一个完整的行 */
        if ( temp == '\r' )
        {
            /* 如果"\r"字符碰巧是目前 buffer 中的最后一个已经被读入的客户数据，那么这次 分析没有读取到一个完整的行，返回 LINE_OPEN 以表示还需要继续读取客户数据才能进一步分析 */
            if ( ( checked_index + 1 ) == read_index )
            {
                return LINE_OPEN;
            }
            /* 如果下一个字符是"\n"，则说明我们成功读取到一个完整的行 */
            else if ( buffer[ checked_index + 1 ] == '\n' )
            {
                buffer[ checked_index++ ] = '\0';
                buffer[ checked_index++ ] = '\0';
                return LINE_OK;
            }
            /* 否则的话，说明客户发送的 HTTP 请求存在语法问题 */
            return LINE_BAD;
        }
        /* 如果当前的字节是"\n"，即换行符，则也说明可能读取到一个完整的行 */
        else if( temp == '\n' )
        {
            if( ( checked_index > 1 ) &&  buffer[ checked_index - 1 ] == '\r' )
            {
                buffer[ checked_index-1 ] = '\0';
                buffer[ checked_index++ ] = '\0';
                return LINE_OK;
            }
            return LINE_BAD;
        }
    }
    /* 如果所有内容都分析完毕也没遇到"\r"字符，则返回LINE OPEN，表示还需要继续读取客户数 据才能进一步分析*/
    return LINE_OPEN;
}

/* 分析请求行 */
HTTP_CODE parse_requestline( char* szTemp, CHECK_STATE& checkstate )
{
    char* szURL = strpbrk( szTemp, " \t" );
    /* 如果请求行中没有空白字符或"\t"字符，则HTTP请求必有问题 */
    if ( ! szURL )
    {
        return BAD_REQUEST;
    }
    *szURL++ = '\0';

    char* szMethod = szTemp;
    if ( strcasecmp( szMethod, "GET" ) == 0 )
    {
        printf( "The request method is GET\n" );
    }
    else
    {
        return BAD_REQUEST;
    }

    szURL += strspn( szURL, " \t" );
    char* szVersion = strpbrk( szURL, " \t" );
    if ( ! szVersion )
    {
        return BAD_REQUEST;
    }
    *szVersion++ = '\0';
    szVersion += strspn( szVersion, " \t" );
    if ( strcasecmp( szVersion, "HTTP/1.1" ) != 0 )
    {
        return BAD_REQUEST;
    }

    if ( strncasecmp( szURL, "http://", 7 ) == 0 )
    {
        szURL += 7;
        szURL = strchr( szURL, '/' );
    }

    if ( ! szURL || szURL[ 0 ] != '/' )
    {
        return BAD_REQUEST;
    }

    //URLDecode( szURL );
    printf( "The request URL is: %s\n", szURL );
    /* HTTP请求行处理完毕，状态转移到头部字段的分析 */
    checkstate = CHECK_STATE_HEADER;
    return NO_REQUEST;
}

/* 分析头部字段 */
HTTP_CODE parse_headers( char* szTemp )
{
    /* 遇到一个空行，说明我们得到了一个正确的 HTTP 请求 */
    if ( szTemp[ 0 ] == '\0' )
    {
        return GET_REQUEST;
    }
    else if ( strncasecmp( szTemp, "Host:", 5 ) == 0 )	 /* 处理"HOST"头部字段 */
    {
        szTemp += 5;
        szTemp += strspn( szTemp, " \t" );
        printf( "the request host is: %s\n", szTemp );
    }
    else
    {
        printf( "I can not handle this header\n" );
    }

    return NO_REQUEST;
}

/* 分析 HTTP 请求的入口函数*/
HTTP_CODE parse_content( char* buffer, int& checked_index, CHECK_STATE& checkstate, int& read_index, int& start_line )
{
    LINE_STATUS linestatus = LINE_OK;			/* 记录当前行的状态 */
    HTTP_CODE retcode = NO_REQUEST;				/* 记录HTTP请求的处理结果 */
    /* 主状态机，用于从 buffer 中取出所有完整的行 */
    while( ( linestatus = parse_line( buffer, checked_index, read_index ) ) == LINE_OK )
    {
        char* szTemp = buffer + start_line;		/* start line是行在buffer 中的起始位置 */
        start_line = checked_index;				/* 记录下一行的起始位置*/
        switch ( checkstate )
        {
            case CHECK_STATE_REQUESTLINE:		/*第一个状态，分析请求行*/
            {
                retcode = parse_requestline( szTemp, checkstate );
                if ( retcode == BAD_REQUEST )
                {
                    return BAD_REQUEST;
                }
                break;
            }
            case CHECK_STATE_HEADER:			/*第二个状态，分析头部字段*/
            {
                retcode = parse_headers( szTemp );
                if ( retcode == BAD_REQUEST )
                {
                    return BAD_REQUEST;
                }
                else if ( retcode == GET_REQUEST )
                {
                    return GET_REQUEST;
                }
                break;
            }
            default:
            {
                return INTERNAL_ERROR;
            }
        }
    }
    /* 若没有读取到一个完整的行，则表示还需要继续读取客户数据才能进一步分析 */
    if( linestatus == LINE_OPEN )		
    {
        return NO_REQUEST;
    }
    else
    {
        return BAD_REQUEST;
    }
}

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );
    
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );
    
    int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
    assert( listenfd >= 0 );
    
    int ret = bind( listenfd, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );
    
    ret = listen( listenfd, 5 );
    assert( ret != -1 );
    
    struct sockaddr_in client_address;
    socklen_t client_addrlength = sizeof( client_address );
    int fd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
    if( fd < 0 )
    {
        printf( "errno is: %d\n", errno );
    }
    else
    {
        char buffer[ BUFFER_SIZE ];					/* 读取缓冲区 */
        memset( buffer, '\0', BUFFER_SIZE );
        int data_read = 0;
        int read_index = 0;							/* 当前已经读取了多少字节的客户数据 */
        int checked_index = 0;						/* 当前已经分析了多少字节的客户数据 */
        int start_line = 0;
        CHECK_STATE checkstate = CHECK_STATE_REQUESTLINE;	/* 设置主状态机的初始状态 */
        while( 1 )
        {
            data_read = recv( fd, buffer + read_index, BUFFER_SIZE - read_index, 0 );
            if ( data_read == -1 )
            {
                printf( "reading failed\n" );
                break;
            }
            else if ( data_read == 0 )
            {
                printf( "remote client has closed the connection\n" );
                break;
            }
    
            read_index += data_read;
            /* 分析目前已经获得的所有客户数据*/
            HTTP_CODE result = parse_content( buffer, checked_index, checkstate, read_index, start_line );
            if( result == NO_REQUEST )			/* 尚未得到一个完整的 HTTP请求 */
            {
                continue;
            }
            else if( result == GET_REQUEST )	/* 得到一个完整的、正确的HTTP请求 */
            {
                send( fd, szret[0], strlen( szret[0] ), 0 );
                break;
            }
            else
            {
                send( fd, szret[1], strlen( szret[1] ), 0 );
                break;
            }
        }
        close( fd );
    }
    
    close( listenfd );
    return 0;
}

```

> [strbreak函数](https://www.cplusplus.com/reference/cstring/strpbrk/)
>
> [strcasecmp函数](https://linux.die.net/man/3/strcasecmp)
>
> [strspn函数](https://www.cplusplus.com/reference/cstring/strspn/?kw=strspn)

<img src="https://i.loli.net/2021/09/13/NBjW8hEI6vXVnSJ.png" alt="从状态机状态转移.png" style="zoom:80%;" />

从状态机的初始状态是 LINE OK，其原始驱动力来自于buffer中新到达的客户数据。在 main 函数中，我们循环调用 recv 函数往 buffer 中读入客户数据。每次成功读取数据后，我们就调用 parse_ content 函数来分析新读入的数据。parse_content 函数首先要做的就是调用 parse line 函数来获取一个行。现在假设服务器经过一次recv 调用之后，buffer 的内容以及部分变量的值如图 8-16a 所示。parse line 函数处理后的结果如图8-16b 所示，它挨个检查图 8-16a所示的 buffer 中checked_index 到（read_index-1）之间的字节，判断是否存在行结束符，并更新 checkedindex的值。当前 buffer 中不存在行结束符，所以 parse line返回LINE OPEN。接下来，程序继续调用 recv 以读取更多客户数据，这次读操作后 buffer 中的内容以及部分变量的值如图8-16c 所示。然后 parse_line 函数就又开始处理这部分新到来的数据，如图 8-16d 所示。这次它读取到了一个完整的行，即"HOST∶localhost\r\n"。此时，parse line 函数就可以将这行内容递交给 parse content 函数中的主状态机来处理了。

<img src="https://i.loli.net/2021/09/13/brfqMlFo1GQeUX3.png" alt="parser_line函数.png" style="zoom:80%;" />

主状态机使用checkstate 变量来记录当前的状态。如果当前的状态是 CHECK_STATE_REQUESTLINE，则表示 parse_line 函数解析出的行是请求行，于是主状态机调用 parserequestline来分析请求行;如果当前的状态是 CHECK_STATE_HEADER，则表示 parse line函数解析出的是头部字段，于是主状态机调用 parse_headers 来分析头部字段。checkstate 变量的初始值是 CHECK_STATE_REQUESTLINE，parse_requestline 函数在成功地分析完请求行之后将其设置为 CHECK_STATE HEADER，从而实现状态转移。







### 7. 提高服务器性能的其他建议

#### 7.1 池

既然服务器的硬件资源"充裕"，那么提高服务器性能的一个很直接的方法就是以空间换时间，即"浪费"服务器的硬件资源，以换取其运行效率。这就是池（pool）的概念。池是一组资源的集合，这组资源在服务器启动之初就被完全创建好并初始化，这称为静态资源分配。当服务器进入正式运行阶段，即开始处理客户请求的时候，如果它需要相关的资源，就可以直接从池中获取，无须动态分配。很显然，直接从池中取得所需资源比动态分配资源的速度要快得多，因为分配系统资源的系统调用都是很耗时的。当服务器处理完—个客户连接后，可以把相关的资源放回池中，无须执行系统调用来释放资源。从最终的效果来看，池相当于服务器管理系统资源的应用层设施，它避免了服务器对内核的频繁访问。



根据不同的资源类型，池可分为多种，常见的有内存池、进程池、线程池和连接池。它们的含义都很明确。

1. 内存池：内存池通常用于socket 的接收缓存和发送缓存。对于某些长度有限的客户请求，比如HTTP请求，预先分配一个大小足够（比如 5000 字节）的接收缓存区是很合理的。当客户请求的长度超过接收缓冲区的大小时，我们可以选择丢弃请求或者动态扩大接收缓冲区。
2. 进程池和线程池：进程池和线程池都是并发编程常用的"伎俩"。当我们需要一个工作进程或工作线程来处理新到来的客户请求时，我们可以直接从进程池或线程池中取得一个执行实体，而无须动态地调用 fork 或 pthread_create 等函数来创建进程和线程。
3. 连接池：连接池通常用于服务器或服务器机群的内部永久连接。每个逻辑单元可能都需要频繁地访问本地的某个数据库。简单的做法是∶逻辑单元每次需要访问数据库的时候，就向数据库程序发起连接，而访问完毕后释放连接。很显然，这种做法的效率太低。一种解决方案是使用连接池。连接池是服务器预先和数据库程序建立的一组连接的集合。当某个逻辑单元需要访问数据库时，它可以直接从连接池中取得一个连接的实体并使用之。待完成数据库的访问之后，逻辑单元再将该连接返还给连接池。

#### 7.2 数据复制

高性能服务器应该避免不必要的数据复制，尤其是当数据复制发生在用户代码和内核之间的时候。如果内核可以直接处理从 socket 或者文件读入的数据，则应用程序就没必要将这些数据从内核缓冲区复制到应用程序缓冲区中。这里说的"直接处理"指的是应用程序不关心这些数据的内容，不需要对它们做任何分析。

此外，用户代码内部（不访问内核）的数据复制也是应该避免的。举例来说，当两个工作进程之间要传递大量的数据时，我们就应该考虑使用共享内存来在它们之间直接共享这些数据，而不是使用管道或者消息队列来传递。





#### 7.3 上下文切换和锁

并发程序必须考虑上下文切换（context switch）的问题，即进程切换或线程切换导致的的系统开销。即使是 LI/O 密集型的服务器，也不应该使用过多的工作线程（或工作进程，下同），否则线程间的切换将占用大量的 CPU 时间，服务器真正用于处理业务逻辑的 CPU 时间的比重就显得不足了。[半同步/半异步模式](#5.1 半同步/半异步模式（half-sync/half-async）)是一种比较合理的解决方案，它允许一个线程同时处理多个客户连接。此外，多线程服务器的一个优点是不同的线程可以同时运行在不同的CPU 上。当线程的数量不大于 CPU 的数目时，上下文的切换就不是问题了。



并发程序需要考虑的另外一个问题是共享资源的加锁保护。锁通常被认为是导致服务器效率低下的一个因素，因为由它引入的代码不仅不处理任何业务逻辑，而且需要访问内核资源。因此，服务器如果有更好的解决方案，就应该避免使用锁。显然，半同步/半异步模式就半同步/半反应堆模式的效率高。如果服务器必须使用"锁"，则可以考虑减小锁的粒度，比如使用读写锁。当所有工作线程都只读取一块共享内存的内容时，读写锁并不会增加系统的额外开销。只有当其中某—个工作线程需要写这块内在存时，系统才必须去锁住这块区域。









## I/O复用

I/O 复用使得程序能同时监听多个文件描述符，这对提高程序的性能至关重要。通常，网络程序在下列情况下需要使用 I/O复用技术∶

- 客户端程序需要同时监听多个socket。
- 客户端程序需要同时处理用户输入和网络连接。
- TCP服务要同时处理监听socket和连接socket。
- 服务器要同时处理TCP请求和UDP请求。
- 服务器要同时监听多个端口，或者处理多种请求。

需要指出的是，I/O 复用虽然能同时监听多个文件描述符，但它本身是阻塞的。并且当多个文件描述符同时就绪时，如果不采取额外的措施，程序就只能按顺序依次处理其中的每—个文件描述符，这使得服务器程序看起来像是串行工作的。如果要实现并发，只能使用多进程或多线程等编程手段。



### 1.select系统调用

**==select 系统调用的用途是∶在一段指定时间内，监听用户感兴趣的文件描述符上的可读、可写和异常等事件。==**

#### 1.1 select API

select系统调用的原型：

```cpp
#include <sys/select.h>
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* expectfds, struct timeval* timeout);
```
1. ndfs参数指定被监听的文件描述符的总数。他通常被设置为select监听的所有文件描述符的最大值+1,因为文件描述符是从0开始的。
2. readfds、writefds 和 exceptfds 参数分别指向可读、可写和异常等事件对应的文件描述符集合。应用程序调用 select 函数时，通过这 3个参数传入自己感兴趣的文件描述符。select 调用返回时，内核将修改它们来通知应用程序哪些文件描述符已经就绪。这三个类型都是fd_set类型的结构体，其原形如下：
```cpp
#include <typesizes.h>
#define __FD_SETSIZE 1024

#include <sys/select.h>
#define FD_SETSIZE __FD_SETSIZE
typedef long int __fd_mask;
#undef __NFDBITS
#define __NFDBITS (8 * (int) sizeof(__fd_mask))
typedef struct
{
#ifdef __USE_XOPEN
	__fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
#define __FDS_BITS(set) ((set)->fds_bits)
#else
	__fd_mask __fds_bits[__FD_SETSIZE / __NFBITS];
#define __FDS_BITS(set) ((set)->__fds_bits)
#endif
}fd_set;
```


由以上定义可见，fd_set 结构体仅包含一个整型数组，该数组的每个元素的每一位（bit）标记一个文件描述符。fd_set 能容纳的文件描述符数量由 FD SETSIZE指定，这就限制了select 能同时处理的文件描述符的总量。由于位操作过于烦琐，我们应该使用下面的一系列宏来访问 fd_set 结构体中的位∶

```cpp
#include <sys/select.h>
FD_ZERO(fd_set* fdset);				/* 清楚fd_set的所有位 */
FD_SET(int fd, fd_set* fdset);		/* 设置fdset的位fd */
FD_SET(int fd, fd_set* fdset);		/* 清除fdset的位fd */
int FD_ISSET(int fd, fd_set* fdset);	/* 测试fdset的位fd是否被设置 */
```

3. timeout 参数用来设置 select 函数的超时时间。它是一个 timeval 结构类型的指针，采用指针参数是因为内核将修改它以告诉应用程序 select 等待了多久。不过我们不能完全信任select 调用返回后的 timeout 值，比如调用失败时 timeout 值是不确定的。timeval 结构体的定 义如下∶

```cpp
struct timeval
{
	long tv_sec; /* 秒数*/
	long tv_usec; /* 微秒数*/
};

```

由以上定义可见，select 给我们提供了一个微秒级的定时方式。如果给 timeout变量的tv_sec 成员和 tv_usec 成员都传递 0，则 select将立即返回。如果给 timeout传递 NULL，则select 将一直阻塞，直到某个文件描述符就绪。
select 成功时返回就绪（可读、可写和异常）文件描述符的总数。如果在超时时间内没有任何文件描述符就绪，select将返回 0。select失败时返回-1并设置 errno。如果在selec等待期间，程序接收到信号，则 select 立即返回-1，并设置 errno为 EINTR。


#### 1.2 文件描述符继续条件
在网络编程中，**下列情况下 socket 可读∶**
- socket 内核接收缓存区中的字节数大于或等于其低水位标记 SO_RCVLOWAT。此时我们可以无阻塞地读该 socket，并且读操作返回的字节数大于 0。
-  socket 通信的对方关闭连接。此时对该 socket 的读操作将返回 0。
-  监听 socket 上有新的连接请求。
-  socket 上有未处理的错误。此时我们可以使用 getsockopt 来读取和清除该错误。
**下列情况socket可写：**
- socket 内核发送缓存区中的可用字节数大于或等于其低水位标记 SO SNDLOWAT。此时我们可以无阻塞地写该 socket，并且写操作返回的字节数大于 0。
- socket 的写操作被关闭。对写操作被关闭的 socket执行写操作将触发一个 SIGPIPE 信号。
- socket 使用非阻塞 connect 连接成功或者失败（超时）之后。
- socket 上有未处理的错误。此时我们可以使用 getsockopt 来读取和清除该错误。

#### 1.3 处理带外数据
socket 上接收到普通数据和带外数据都将使 select 返回，但 socket 处于不同的就绪状态 ∶ 前者处于可读状态，后者处于异常状态。

<div align="center" style="font-size:larger;font-weight:900">同时接收普通数据和带外数据</div>

```cpp

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>

int main( int argc, char* argv[] )
{
	if( argc <= 2 )
	{
		printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
		return 1;
	}
	const char* ip = argv[1];
	int port = atoi( argv[2] );
	printf( "ip is %s and port is %d\n", ip, port );

	int ret = 0;
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

	int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
	assert( listenfd >= 0 );

    ret = bind( listenfd, ( struct sockaddr* )&address, sizeof( address ) );
	assert( ret != -1 );

	ret = listen( listenfd, 5 );
	assert( ret != -1 );

	struct sockaddr_in client_address;
    socklen_t client_addrlength = sizeof( client_address );
	int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
	if ( connfd < 0 )
	{
		printf( "errno is: %d\n", errno );
		close( listenfd );
	}

	char remote_addr[INET_ADDRSTRLEN];
	printf( "connected with ip: %s and port: %d\n", inet_ntop( AF_INET, &client_address.sin_addr, remote_addr, INET_ADDRSTRLEN ), ntohs( client_address.sin_port ) );

	char buf[1024];
    fd_set read_fds;
    fd_set exception_fds;

    FD_ZERO( &read_fds );
    FD_ZERO( &exception_fds );

    int nReuseAddr = 1;
	setsockopt( connfd, SOL_SOCKET, SO_OOBINLINE, &nReuseAddr, sizeof( nReuseAddr ) );
	while( 1 )
	{
		memset( buf, '\0', sizeof( buf ) );
		/* 每次调用select前都要重新在read_fds和exception_fds中设置文件描述符 connfd，因为事件发生之后，文件描述符集合将被内核修改 */
		FD_SET( connfd, &read_fds );
		FD_SET( connfd, &exception_fds );

        ret = select( connfd + 1, &read_fds, NULL, &exception_fds, NULL );
		printf( "select one\n" );
        if ( ret < 0 )
        {
                printf( "selection failure\n" );
            	break;
        }
		/* 对于可读事件，采用普通的recv 函数读取数据 */	
    	if ( FD_ISSET( connfd, &read_fds ) )
		{
    		ret = recv( connfd, buf, sizeof( buf )-1, 0 );
			if( ret <= 0 )
			{
				break;
			}
			printf( "get %d bytes of normal data: %s\n", ret, buf );
		}
		/* 对于异常事件，采用带 MSG 0OB 标志的 recv 函数读取带外数据 */
		else if( FD_ISSET( connfd, &exception_fds ) )
        {
    		ret = recv( connfd, buf, sizeof( buf )-1, MSG_OOB );
			if( ret <= 0 )
			{
				break;
			}
			printf( "get %d bytes of oob data: %s\n", ret, buf );
        }

	}

	close( connfd );
	close( listenfd );
	return 0;
}
```

#### 9.2 poll系统调用
poll系统调用和select类似，也是在指定时间内轮询一定数量的文件描述符，以测试其中是否有就绪者。poll 的原型如下∶
```cpp
#include <poll.h>
int poll( struct pollfd fds,nfds_t nfds,int timeout );
```
1. fds 参数是一个 pollfd结构类型的数组，它指定所有我们感兴趣的文件描述符上发生的可读、可写和异常等事件。pollfd 结构体的定义如下∶
```cpp
struct pollfd
{
	int fd;			/* 文件描述符 */
	short events;	/* 注册的事件 */
	short revents;	/* 实际发生的事件
}
```
其中，fd成员指定文件描述符; events 成员告诉 poll 监听 fd上的哪些事件，它是一系列事件的按位或;revents 成员则由内核修改，以通知应用程序 fd上实际发生了哪些事件。pol支持的事件类型如表所示。

<div align="center" style="font-size:large;font-weight:900">poll事件类型</div>
<div align="center">
	<table align="center">
	<tr align="center" style="font-weight:700">
		<td>事件</td>
		<td>描述</td>
		<td>是否可作为输入</td>
		<td>是否可作为输出</td>
	</tr>
	<tr>
	<tr>
		<td>POLLIN</td>
		<td>数据（包括普通数据和优先数据）可读</td>
		<td>是</td>
		<td>是</td>
	</tr>
		<td>POLLRDNORM</td>
		<td>普通数据可读</td>
		<td>是</td>
		<td>是</td>
	</tr>
	<tr>
		<td>POLLRDBAND</td>
		<td>优先级带数据可读</td>
		<td>是</td>
		<td>是</td>
	</tr>
	<tr>
		<td>POLLPRI</td>
		<td><高优先级数据可读/td>
		<td>是</td>
		<td>是</td>
	</tr>
	<tr>
		<td>POLLOUT</td>
		<td>数据（包括普通数据和优先数据）可写</td>
		<td>是</td>
		<td>是</td>
	</tr>
	<tr>
		<td>POLLWRNORM</td>
		<td>普通数据可写</td>
		<td>是</td>
		<td>是</td>
	</tr>
	<tr>
		<td>POLLWRBAND</td>
		<td>优先级带数据可写</td>
		<td>是</td>
		<td>是</td>
	</tr>
	<tr>
		<td>POLLRDHUP</td>
		<td>TCP连接被对方关闭，或者对方关闭了写操作</td>
		<td>是</td>
		<td>是</td>
	</tr>
	<tr>
		<td>POLLERR</td>
		<td>错误</td>
		<td>否</td>
		<td>是</td>
	</tr>
	<tr>
		<td>POLLHUP</td>
		<td>挂起</td>
		<td>否</td>
		<td>否</td>
	</tr>
	<tr>
		<td>POLLNVAL</td>
		<td>文件描述符没有打开</td>
		<td>否</td>
		<td>是</td>
	</tr>
	</table>
</div>

2. nfds 参数指定被监听事件集合 fds 的大小。其类型 nfds t的定义如下:
```cpp
typedef unsigned long int nfds_t;
```

3. timeout参数指定poll的超时值，单位是毫秒。当timeout 为-1时，poll 调用将永远阻塞，直到某个事件发生; 当 timeout 为 0时，poll 调用将立即返回。


### 3 epoll系列系统调用
#### 3.1 内核事件表

epoll是 Linux 特有的I/O 复用函数。它在实现和使用上与 select、poll有很大差异。首先，epoll使用一组函数来完成任务，而不是单个函数。其次，epoll把用户关心的文件描述符上的事件放在内核里的一个事件表中，从而无须像 select 和 poll 那样每次调用都要重复传入文件描述符集或事件集。但 epoll需要使用一个额外的文件描述符，来唯一标识内核中的这个事件表。这个文件描述符使用如下 epoll_create 函数来创建∶
```cpp
#include <sys/epoll.h>
int epoll_create(int size);
```

size 参数现在并不起作用，只是给内核一个提示，告诉它事件表需要多大。该函数返回的文件描述符将用作其他所有 epoll系统调用的第一个参数，以指定要访问的内核事件表。

下面的函数用来操作 epoll 的内核事件表∶
```c++
#include <sys/epoll.h>
int epoll_ctl( int epfd,int op,int fd,struct epoll_event event )
```
fd参数是要操作的文件描述符，op 参数则指定操作类型。操作类型有如下 3 种∶
- EPOLL_CTL_ADD:往时间表中注册fd上的事件。
- EPOLL_CTL_MOD:修改fd上的注册事件。
- EPOLL_CTL_DEL:删除fd上的注册事件。
event参数指定事件，它是epoll_event结构指针类型。epoll_event的定义如下:
```cpp
struct epoll_event
{
	__unit32_t events;	/* epoll事件 */
	epoll_data_t data;	/* 用户数据 */
};

```

其中events 成员描述事件类型。epoll支持的事件类型和 poll基本相同。表示 epoll事件
类型的宏是在 poll对应的宏前加上"E"，比如 epoll的数据可读事件是 EPOLLIN。但epoll有
两个额外的事件类型——EPOLLET和 EPOLLONESHOT。它们对于epoll的高效运作非常关
键，我们将在后面讨论它们。data 成员用于存储用户数据，其类型 epolldata_t 的定义如下∶
```cpp
typedef union epoll_date
{
	void* ptr;
	int fd;
	uint_32_t u32;
	uint64_t u64;
}epoll_data_t;
```

epoll_data_t 是一个联合体，其4个成员中使用最多的是 fd，它指定事件所从属的目标文件描述符。ptr成员可用来指定与 fd 相关的用户数据。但由于epoll data t是一个联合体，我们不能同时使用其 ptr 成和 fd 成员，因此，如果要将文件描述符和用户数据关联起来，以实现快速的数据访问，只能使用其他手段.比如放弃使用 epoll data_t的 fd 成员，而在 ptr 指向的用户数据中包含 fd。
epoll_ctl 成功时返回0，失败则返回 -1 并设置 errno。


#### 3.2 epoll_wait函数
epoll系列系统调用的主要接口是 epoll_wait 函数。它在一段超时时间内等待一组文件描述符上的事件，其原型如下∶
```cpp
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```
该函数成功时返回就绪的文件描述符的个数，失败时返回-1 并设置 errno。
meout 参数的含义与 poll接口的 timeout 参数相同。maxevents 参数指定最多监听多少个事件，它必须大于 0。
epoll_wait 函数如果检测到事件，就将所有就绪的事件从内核事件表（由 epfd 参数指定）中复制到它的第二个参数events 指向的数组中。这个数组只用于输出 epoll_wait 检测到的就绪事件，而不像 select 和 poll 的数组参数那样既用于传入用户注册的事件，又用于输出内核检测到的就绪事件。这就极大地提高了应用程序索引就绪文件描述符的效率。

<div align="center" style="font-size:larger;font-weight:900">poll和epoll在使用上的差别</div>

```cpp
/* 如何索引poll返回的就绪文件描述符 */
int ret = poll(fds, MAX_EVENT_NUMBER, -1);
for(int i = 0;i < MAX_EVENT_NUMBER; i++)
{
	if(fds[i].events & POLLIN)
	{
		int sockfd = fds[i].fd;
		/* 处理sockfd */
	}
}

/* 如何索引epoll返回的文件描述符 */
int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
/* 仅便利就绪的ret个文件描述符 */
for(int i = 0; i < ret; ++i)
{
	int sockfd = events[i].data.fd;
	/* sockfd肯定就绪，直接处理 */

}
```

#### 3.3 LT和ET模式

epoll 对文件描述符的操作有两种模式∶ LT（Level Trigger，电平触发）模式和 ET（Edge Trigger，边沿触发）模式。LT模式是默认的工作模式，这种模式下 epoll相当于一个效率较高的 poll。当往 epoll 内核事件表中注册一个文件描述符上的 EPOLLET 事件时，epoll将以ET 模式来操作该文件描述符。ET模式是 epoll 的高效工作模式。

**==对于采用LT工作模式的文件描述符，当 epoll wait 检测到其上有事件发生并将此
事件通知应用程序后，应用程序可以不立即处理该事件。这样，当应用程序下一次调用
epoll_wait 时，epollwait 还会再次向应用程序通告此事件，直到该事件被处理。而对于采用 ET工作模式的文件描述符，当 epoll wait 检测到其上有事件发生并将此事件通知应用程序后，应用程序必须立即处理该事件，因为后续的 epoll wait 调用将不再向应用程序通知这一事件。可见，ET 模式在很大程度上降低了同一个 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。==**

<div align="center" style="font-size=larger;font-weight=900">ET和LT模式的差别</div>

```cpp

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>

#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 10

/* 将文件描述符设置成非阻塞的 */
int setnonblocking( int fd )
{
    int old_option = fcntl( fd, F_GETFL );
    int new_option = old_option | O_NONBLOCK;
    fcntl( fd, F_SETFL, new_option );
    return old_option;
}
/* 将文件描迷符 fd上的EPOLLIN注册到epollfd指示的epoll内核事件表中，参数enable_et指定 是否对 fd启用 ET 模式*/
void addfd( int epollfd, int fd, bool enable_et )
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN;
    if( enable_et )
    {
        event.events |= EPOLLET;
    }
    epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
    setnonblocking( fd );
}
/* LT 模式的工作流程*/
void lt( epoll_event* events, int number, int epollfd, int listenfd )
{
    char buf[ BUFFER_SIZE ];
    for ( int i = 0; i < number; i++ )
    {
        int sockfd = events[i].data.fd;
        if ( sockfd == listenfd )
        {
            struct sockaddr_in client_address;
            socklen_t client_addrlength = sizeof( client_address );
            int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
            addfd( epollfd, connfd, false );
        }
        else if ( events[i].events & EPOLLIN )
        {
            printf( "event trigger once\n" );
            memset( buf, '\0', BUFFER_SIZE );
            int ret = recv( sockfd, buf, BUFFER_SIZE-1, 0 );
            if( ret <= 0 )
            {
                close( sockfd );
                continue;
            }
            printf( "get %d bytes of content: %s\n", ret, buf );
        }
        else
        {
            printf( "something else happened \n" );
        }
    }
}
/* ET 模式的工作流程*/
void et( epoll_event* events, int number, int epollfd, int listenfd )
{
    char buf[ BUFFER_SIZE ];
    for ( int i = 0; i < number; i++ )
    {
        int sockfd = events[i].data.fd;
        if ( sockfd == listenfd )
        {
            struct sockaddr_in client_address;
            socklen_t client_addrlength = sizeof( client_address );
            int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
            addfd( epollfd, connfd, true );
        }
        else if ( events[i].events & EPOLLIN )
        {
			/* 这段代码不会被重复触发，所以我们循环读取数据，以确保把 socket 读缓存中的 所有数据读出*/
            printf( "event trigger once\n" );
            while( 1 )
            {
                memset( buf, '\0', BUFFER_SIZE );
                int ret = recv( sockfd, buf, BUFFER_SIZE-1, 0 );
                if( ret < 0 )
                {
					/* 对于非阻塞 IO，下面的条件成立表示数据已经全部读取完毕。此后，
epol1 就能再次触发 sockfd上的EPOLLIN 事件，以驱动下一次读操作*/
                    if( ( errno == EAGAIN ) || ( errno == EWOULDBLOCK ) )
                    {
                        printf( "read later\n" );
                        break;
                    }
                    close( sockfd );
                    break;
                }
                else if( ret == 0 )
                {
                    close( sockfd );
                }
                else
                {
                    printf( "get %d bytes of content: %s\n", ret, buf );
                }
            }
        }
        else
        {
            printf( "something else happened \n" );
        }
    }
}

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    int ret = 0;
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
    assert( listenfd >= 0 );

    ret = bind( listenfd, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( listenfd, 5 );
    assert( ret != -1 );

    epoll_event events[ MAX_EVENT_NUMBER ];
    int epollfd = epoll_create( 5 );
    assert( epollfd != -1 );
    addfd( epollfd, listenfd, true );

    while( 1 )
    {
        int ret = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, -1 );
        if ( ret < 0 )
        {
            printf( "epoll failure\n" );
            break;
        }
    
        lt( events, ret, epollfd, listenfd );
        //et( events, ret, epollfd, listenfd );
    }

    close( listenfd );
    return 0;
}
```
> WARNING: 每个使用 ET 模式的文件描述符都应该是非阻塞的。如果文件描述符是阻塞的，那么读或写操作将会因为没有后续的事件而一直处于阻塞状态（饥渴状态）。


#### 3.4 EPOLLONESHOT事件

即使我们使用 ET 模式，一个 socket 上的某个事件还是可能被触发多次。这在并发程序中就会引起一个问题。比如一个线程（或进程，下同）在读取完某个socket上的数据后开始处理这些数据，而在数据的处理过程中该socket上又有新数据可读（EPOLLIN 再次被触发），此时另外一个线程被唤醒来读取这些新的数据。于是就出现了两个线程同时操作一个socket 的局面。这当然不是我们期望的。我们期望的是一个 socket 连接在任一时刻都只被一个线程处理。这一点可以使用 epoll 的 EPOLLONESHOT 事件实现。
对于注册了 EPOLLONESHOT 事件的文件描述符，操作系统最多触发其上注册的一个可
读、可写或者异常事件，且只触发一次，除非我们使用 epoll ctl 函数重置该文件描述符上注册的 EPOLLONESHOT事件。这样，当一个线程在处理某个 socket 时，其他线程是不可能有机会操作该 socket 的。但反过来思考，注册了EPOLLONESHOT事件的 socket一且被某个线程处理完毕，该线程就应该立即重置这个 socket 上的 EPOLLONESHOT事件，以确保这个socket 下一次可读时，其 EPOLLIN 事件能被触发，进而让其他工作线程有机会继续处理这个socket。

<div align="center" style="font-size:larger;font-weight:900">使用EPOLLONESHOT事件</div>

```cpp
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>

#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 1024
struct fds
{
   int epollfd;
   int sockfd;
};

int setnonblocking( int fd )
{
    int old_option = fcntl( fd, F_GETFL );
    int new_option = old_option | O_NONBLOCK;
    fcntl( fd, F_SETFL, new_option );
    return old_option;
}
/* 将fd上的EPOLLIN和EPOLLET事件注册到epollfd指示的epoll内核事件表中，参数 oneshot 指定是否注册 fd 上的 EPOLLONESHOT 事件*/
void addfd( int epollfd, int fd, bool oneshot )
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET;
    if( oneshot )
    {
        event.events |= EPOLLONESHOT;
    }
    epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
    setnonblocking( fd );
}
/* 重置fd上的事件。这样操作之后，尽管 fd 上的EPOLLONESHOT事件被注册，但是操作系统仍然会触 发 fd上的 EPOLLIN 事件，且只触发一次*/
void reset_oneshot( int epollfd, int fd )
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
    epoll_ctl( epollfd, EPOLL_CTL_MOD, fd, &event );
}

/* 工作线程 */
void* worker( void* arg )
{
    int sockfd = ( (fds*)arg )->sockfd;
    int epollfd = ( (fds*)arg )->epollfd;
    printf( "start new thread to receive data on fd: %d\n", sockfd );
    char buf[ BUFFER_SIZE ];
    memset( buf, '\0', BUFFER_SIZE );
    while( 1 )
    {
        int ret = recv( sockfd, buf, BUFFER_SIZE-1, 0 );
        if( ret == 0 )
        {
            close( sockfd );
            printf( "foreiner closed the connection\n" );
            break;
        }
        else if( ret < 0 )
        {
            if( errno == EAGAIN )
            {
                reset_oneshot( epollfd, sockfd );
                printf( "read later\n" );
                break;
            }
        }
        else
        {
            printf( "get content: %s\n", buf );
            sleep( 5 );
        }
    }
    printf( "end thread receiving data on fd: %d\n", sockfd );
}

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    int ret = 0;
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
    assert( listenfd >= 0 );

    ret = bind( listenfd, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( listenfd, 5 );
    assert( ret != -1 );

    epoll_event events[ MAX_EVENT_NUMBER ];
    int epollfd = epoll_create( 5 );
    assert( epollfd != -1 );
	/* 注意，监听 socket listenfd 上是不能注册EPOLLONESHOT事件的，否则应用程序只能处理 一个客户连接!因为后续的客户连接请求将不再触发 listenfd 上的EPOLLIN 事件 */
    addfd( epollfd, listenfd, false );

    while( 1 )
    {
        int ret = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, -1 );
        if ( ret < 0 )
        {
            printf( "epoll failure\n" );
            break;
        }
    
        for ( int i = 0; i < ret; i++ )
        {
            int sockfd = events[i].data.fd;
            if ( sockfd == listenfd )
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof( client_address );
                int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
				/* 对每个非监听文件描述符都注册 EPOLLONESHOT 事件*/
                addfd( epollfd, connfd, true );
            }
            else if ( events[i].events & EPOLLIN )
            {
                pthread_t thread;
                fds fds_for_new_worker;
                fds_for_new_worker.epollfd = epollfd;
                fds_for_new_worker.sockfd = sockfd;
				/* 新启动一个工作线程为sockfd服务*/
                pthread_create( &thread, NULL, worker, ( void* )&fds_for_new_worker );
            }
            else
            {
                printf( "something else happened \n" );
            }
        }
    }

    close( listenfd );
    return 0;
}
```

**尽管一个socket 在不同时间可能被不同的线程处理，但同一时刻肯定只有一个线程在为它服务。这就保证了连接的完整性，从而避免了很多可能的竞态条件。**


### 4. 三组I/O复用函数的比较

我们讨论了select、poll 和 epoll三组 I/O 复用系统调用，这 3 组系统调用都能同时监听多个文件描述符。它们将等待由 timeout 参数指定的超时时间，直到一个或者多个文件描述符上有事件发生时返回，返回值是就绪的文件描述符的数量。返回 0表示没有事件发生。

==这3组函数都通过某种结构体变量来告诉内核监听哪些文件描述符上的哪些事件，并使用该结构体类型的参数来获取内核处理的结果。select 的参数类型fd_set没有将文件描述符和事件绑定，它仅仅是一个文件描述符集合，因此select需要提供3个这种类型的参数来分别传入和输出可读、可写及异常等事件。这一方面使得select不能处理更多类型的事件，另一方面由于内核对fd_set集合的在线修改，应用程序下次调用select前不得不重置这3个fd_set集合。poll的参数类型pollfd则多少“聪明”一些。它把文件描述符和事件都定义其中，任何事件都被统一处理，从而使得编程接口简洁得多。并且内核每次修改的是pollfd结构体的revents成员，而events成员保持不变，因此下次调用poll时应用程序无须重置pollfd类型的事件集参数。由于每次select和 poll调用都返回整个用户注册的事件集合（其中包括就绪的和未就绪的)，所以应用程序索引就绪文件描述符的时间复杂度为О (n)。epoll则采用与select和 poll完全不同的方式来管理用户注册的事件。它在内核中维护一个事件表，并提供了一个独立的系统调用epoll_ctl来控制往其中添加、删除、修改事件。这样，每次 epoll_wait调用都直接从该内核事件表中取得用户注册的事件，而无须反复从用户空间读人这些事件。epoll_wait系统调用的events参数仅用来返回就绪的事件，这使得应用程序索引就绪文件描述符的时间复杂度达到O ( 1)。==

==poll和 epoll_wait分别用nfds 和 maxevents参数指定最多监听多少个文件描述符和事件。这两个数值都能达到系统允许打开的最大文件描述符数目，即65 535 ( cat/proc/sys/fs/file-max)。而select 允许监听的最大文件描述符数量通常有限制。虽然用户可以修改这个限制，但这可能导致不可预期的后果。==

==**从实现原理上来说，select和 poll采用的都是轮询的方式，即每次调用都要扫描整个注册文件描述符集合，并将其中就绪的文件描述符返回给用户程序，因此它们检测就绪事件的算法的时间复杂度是O (n)。epoll_wait则不同，它采用的是回调的方式。内核检测到就绪的文件描述符时，将触发回调函数，回调函数就将该文件描述符上对应的事件插入内核就绪事件队列。**内核最后在适当的时机将该就绪事件队列中的内容拷贝到用户空间。因此epoll_wait无须轮询整个文件描述符集合来检测哪些事件已经就绪，其算法时间复杂度是O (1)。但是，当活动连接比较多的时候，epoll_wait的效率未必比select和 poll 高，因为此时回调函数被触发得过于频繁。所以epoll_wait适用于连接数量多，但活动连接较少的情况。==

![三个IO复用函数的区别.png](https://i.loli.net/2021/09/14/G8RBrfxjLHmIqsA.png)


### 5. I/O复用的高级应用之一：非阻塞connect


```cpp
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <assert.h>
#include <stdio.h>
#include <time.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <string.h>

#define BUFFER_SIZE 1023

int setnonblocking( int fd )
{
    int old_option = fcntl( fd, F_GETFL );
    int new_option = old_option | O_NONBLOCK;
    fcntl( fd, F_SETFL, new_option );
    return old_option;
}

/* 超时连接函数，参数分别是服务器IP地址、端口号和超时时间（毫秒）。函数成功时返回已经处于连接 状态的 socket，失败则返回-1 */
int unblock_connect( const char* ip, int port, int time )
{
    int ret = 0;
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int sockfd = socket( PF_INET, SOCK_STREAM, 0 );
    int fdopt = setnonblocking( sockfd );
    ret = connect( sockfd, ( struct sockaddr* )&address, sizeof( address ) );
    if ( ret == 0 )
    {
		/* 如果连接成功，则恢复 sockfd的属性，并立即返回之 */
        printf( "connect with server immediately\n" );
        fcntl( sockfd, F_SETFL, fdopt );
        return sockfd;
    }
    else if ( errno != EINPROGRESS )
    {
		/* 如果连接没有立即建立，那么只有当errno 是EINPROGRESS 时才表示连接还在进行 否则出错返回*/
        printf( "unblock connect not support\n" );
        return -1;
    }

    fd_set readfds;
    fd_set writefds;
    struct timeval timeout;

    FD_ZERO( &readfds );
    FD_SET( sockfd, &writefds );

    timeout.tv_sec = time;
    timeout.tv_usec = 0;

    ret = select( sockfd + 1, NULL, &writefds, NULL, &timeout );
    if ( ret <= 0 )
    {
		/* select 超时或者出错，立即返回 */
        printf( "connection time out\n" );
        close( sockfd );
        return -1;
    }

    if ( ! FD_ISSET( sockfd, &writefds  ) )
    {
        printf( "no events on sockfd found\n" );
        close( sockfd );
        return -1;
    }

    int error = 0;
    socklen_t length = sizeof( error );
	/* 调用 getsockopt来获取并清除 sockfd 上的错误 */
    if( getsockopt( sockfd, SOL_SOCKET, SO_ERROR, &error, &length ) < 0 )
    {
        printf( "get socket option failed\n" );
        close( sockfd );
        return -1;
    }

	/* 镨误号不为0表示连接出错 */
    if( error != 0 )
    {
        printf( "connection failed after select with the error: %d \n", error );
        close( sockfd );
        return -1;
    }
    
	/* 连接成功 */
    printf( "connection ready after select with the socket: %d \n", sockfd );
    fcntl( sockfd, F_SETFL, fdopt );
    return sockfd;
}

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    int sockfd = unblock_connect( ip, port, 10 );
    if ( sockfd < 0 )
    {
        return 1;
    }
    shutdown( sockfd, SHUT_WR );
    sleep( 200 );
    printf( "send data out\n" );
    send( sockfd, "abc", 3, 0 );
    //sleep( 600 );
    return 0;
}
```



### 6. I/O复用的高级应用二：聊天室程序


<div align="center" style="font-size:larger;font-weight:900">聊天室程序客户端</div>

```cpp

#define _GNU_SOURCE 1
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <poll.h>
#include <fcntl.h>

#define BUFFER_SIZE 64

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    struct sockaddr_in server_address;
    bzero( &server_address, sizeof( server_address ) );
    server_address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &server_address.sin_addr );
    server_address.sin_port = htons( port );

    int sockfd = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sockfd >= 0 );
    if ( connect( sockfd, ( struct sockaddr* )&server_address, sizeof( server_address ) ) < 0 )
    {
        printf( "connection failed\n" );
        close( sockfd );
        return 1;
    }

    pollfd fds[2];
	/*注册文件描述符0（标准输入）和文件描述符 sockfd 上的可读事件 */
    fds[0].fd = 0;
    fds[0].events = POLLIN;
    fds[0].revents = 0;
    fds[1].fd = sockfd;
    fds[1].events = POLLIN | POLLRDHUP;
    fds[1].revents = 0;
    char read_buf[BUFFER_SIZE];
    int pipefd[2];
    int ret = pipe( pipefd );
    assert( ret != -1 );

    while( 1 )
    {
        ret = poll( fds, 2, -1 );
        if( ret < 0 )
        {
            printf( "poll failure\n" );
            break;
        }

        if( fds[1].revents & POLLRDHUP )
        {
            printf( "server close the connection\n" );
            break;
        }
        else if( fds[1].revents & POLLIN )
        {
            memset( read_buf, '\0', BUFFER_SIZE );
            recv( fds[1].fd, read_buf, BUFFER_SIZE-1, 0 );
            printf( "%s\n", read_buf );
        }

        if( fds[0].revents & POLLIN )
        {
			/* 使用 splice将用户输入的数据直接写到sockfd上（零拷贝）*/
            ret = splice( 0, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE );
            ret = splice( pipefd[0], NULL, sockfd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE );
        }
    }
    
    close( sockfd );
    return 0;
}
```


<div align="center" style="font-size:larger;font-weight:900">聊天室程序服务端</div>

```cpp

#define _GNU_SOURCE 1
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <poll.h>

#define USER_LIMIT 5
#define BUFFER_SIZE 64
#define FD_LIMIT 65535

/*客户数据∶客户端 socket地址、待写到客户端的数据的位置、从客户端读入的数据 */
struct client_data
{
    sockaddr_in address;
    char* write_buf;
    char buf[ BUFFER_SIZE ];
};

int setnonblocking( int fd )
{
    int old_option = fcntl( fd, F_GETFL );
    int new_option = old_option | O_NONBLOCK;
    fcntl( fd, F_SETFL, new_option );
    return old_option;
}

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    int ret = 0;
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
    assert( listenfd >= 0 );

    ret = bind( listenfd, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( listenfd, 5 );
    assert( ret != -1 );

	/* 创建 users 数组，分配 FD LIMIT个 client data 对象。可以预期∶每个可能的 socket 连接都可以获得一个这样的对象，并且 socket 的值可以直接用来索引（作为数组的下标）socket 连接对应的client_data 对象，这是将 socket 和客户数据关联的简单而高效的方式 */
    client_data* users = new client_data[FD_LIMIT];
    pollfd fds[USER_LIMIT+1];
    int user_counter = 0;
    for( int i = 1; i <= USER_LIMIT; ++i )
    {
        fds[i].fd = -1;
        fds[i].events = 0;
    }
    fds[0].fd = listenfd;
    fds[0].events = POLLIN | POLLERR;
    fds[0].revents = 0;

    while( 1 )
    {
        ret = poll( fds, user_counter+1, -1 );
        if ( ret < 0 )
        {
            printf( "poll failure\n" );
            break;
        }
    
        for( int i = 0; i < user_counter+1; ++i )
        {
            if( ( fds[i].fd == listenfd ) && ( fds[i].revents & POLLIN ) )
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof( client_address );
                int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
                if ( connfd < 0 )
                {
                    printf( "errno is: %d\n", errno );
                    continue;
                }
                if( user_counter >= USER_LIMIT )
                {
                    const char* info = "too many users\n";
                    printf( "%s", info );
                    send( connfd, info, strlen( info ), 0 );
                    close( connfd );
                    continue;
                }
				/* 对于新的连接，同时修改 fds和users数组。前文已经提到，users[connfd]对应于新连接文件描述符connfd 的客户数据*/
                user_counter++;
                users[connfd].address = client_address;
                setnonblocking( connfd );
                fds[user_counter].fd = connfd;
                fds[user_counter].events = POLLIN | POLLRDHUP | POLLERR;
                fds[user_counter].revents = 0;
                printf( "comes a new user, now have %d users\n", user_counter );
            }
            else if( fds[i].revents & POLLERR )
            {
                printf( "get an error from %d\n", fds[i].fd );
                char errors[ 100 ];
                memset( errors, '\0', 100 );
                socklen_t length = sizeof( errors );
                if( getsockopt( fds[i].fd, SOL_SOCKET, SO_ERROR, &errors, &length ) < 0 )
                {
                    printf( "get socket option failed\n" );
                }
                continue;
            }
            else if( fds[i].revents & POLLRDHUP )
            {
                users[fds[i].fd] = users[fds[user_counter].fd];
                close( fds[i].fd );
                fds[i] = fds[user_counter];
                i--;
                user_counter--;
                printf( "a client left\n" );
            }
            else if( fds[i].revents & POLLIN )
            {
                int connfd = fds[i].fd;
                memset( users[connfd].buf, '\0', BUFFER_SIZE );
                ret = recv( connfd, users[connfd].buf, BUFFER_SIZE-1, 0 );
                printf( "get %d bytes of client data %s from %d\n", ret, users[connfd].buf, connfd );
                if( ret < 0 )
                {
                    if( errno != EAGAIN )
                    {
                        close( connfd );
                        users[fds[i].fd] = users[fds[user_counter].fd];
                        fds[i] = fds[user_counter];
                        i--;
                        user_counter--;
                    }
                }
                else if( ret == 0 )
                {
                    printf( "code should not come to here\n" );
                }
				/*如果接收到客户数据，则通知其他 socket 连接准备写数据 */
                else
                {
                    for( int j = 1; j <= user_counter; ++j )
                    {
                        if( fds[j].fd == connfd )
                        {
                            continue;
                        }
                        
                        fds[j].events |= ~POLLIN;
                        fds[j].events |= POLLOUT;
                        users[fds[j].fd].write_buf = users[connfd].buf;
                    }
                }
            }
            else if( fds[i].revents & POLLOUT )
            {
                int connfd = fds[i].fd;
                if( ! users[connfd].write_buf )
                {
                    continue;
                }
                ret = send( connfd, users[connfd].write_buf, strlen( users[connfd].write_buf ), 0 );
                users[connfd].write_buf = NULL;
				/* 写完数据后需要重新注册 fds【i】上的可读事件 */
                fds[i].events |= ~POLLOUT;
                fds[i].events |= POLLIN;
            }
        }
    }

    delete [] users;
    close( listenfd );
    return 0;
}
```


### 7. I/O复用的高级应用三：同时处理TCP 和UDP请求


<div align="center" style="font-size:larger;font-weight:900">同时处理TCP 和 UDP 服务的回射服务器</div>

```cpp

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>

#define MAX_EVENT_NUMBER 1024
#define TCP_BUFFER_SIZE 512
#define UDP_BUFFER_SIZE 1024

int setnonblocking( int fd )
{
    int old_option = fcntl( fd, F_GETFL );
    int new_option = old_option | O_NONBLOCK;
    fcntl( fd, F_SETFL, new_option );
    return old_option;
}

void addfd( int epollfd, int fd )
{
    epoll_event event;
    event.data.fd = fd;
    //event.events = EPOLLIN | EPOLLET;
    event.events = EPOLLIN;
    epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
    setnonblocking( fd );
}

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    int ret = 0;
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
    assert( listenfd >= 0 );

    ret = bind( listenfd, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( listenfd, 5 );
    assert( ret != -1 );

    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );
    int udpfd = socket( PF_INET, SOCK_DGRAM, 0 );
    assert( udpfd >= 0 );

    ret = bind( udpfd, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    epoll_event events[ MAX_EVENT_NUMBER ];
    int epollfd = epoll_create( 5 );
    assert( epollfd != -1 );
    addfd( epollfd, listenfd );
    addfd( epollfd, udpfd );

    while( 1 )
    {
        int number = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, -1 );
        if ( number < 0 )
        {
            printf( "epoll failure\n" );
            break;
        }
    
        for ( int i = 0; i < number; i++ )
        {
            int sockfd = events[i].data.fd;
            if ( sockfd == listenfd )
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof( client_address );
                int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
                addfd( epollfd, connfd );
            }
            else if ( sockfd == udpfd )
            {
                char buf[ UDP_BUFFER_SIZE ];
                memset( buf, '\0', UDP_BUFFER_SIZE );
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof( client_address );

                ret = recvfrom( udpfd, buf, UDP_BUFFER_SIZE-1, 0, ( struct sockaddr* )&client_address, &client_addrlength );
                if( ret > 0 )
                {
                    sendto( udpfd, buf, UDP_BUFFER_SIZE-1, 0, ( struct sockaddr* )&client_address, client_addrlength );
                }
            }
            else if ( events[i].events & EPOLLIN )
            {
                char buf[ TCP_BUFFER_SIZE ];
                while( 1 )
                {
                    memset( buf, '\0', TCP_BUFFER_SIZE );
                    ret = recv( sockfd, buf, TCP_BUFFER_SIZE-1, 0 );
                    if( ret < 0 )
                    {
                        if( ( errno == EAGAIN ) || ( errno == EWOULDBLOCK ) )
                        {
                            break;
                        }
                        close( sockfd );
                        break;
                    }
                    else if( ret == 0 )
                    {
                        close( sockfd );
                    }
                    else
                    {
                        send( sockfd, buf, ret, 0 );
                    }
                }
            }
            else
            {
                printf( "something else happened \n" );
            }
        }
    }

    close( listenfd );
    return 0;
}
```



### 8. 超级服务xinted

Linux 因特网服务 inetd 是超级服务。它同时管理着多个子服务，即监听多个端口。现在Linux 系统上使用的 inetd 服务程序通常是其升级版本 xinetd。xinetd 程序的原理与 inetd 相同，但增加了一些控制选项，并提高了安全性。


#### 8.1 xinted配置文件

xinetd采用 /etc/xinetd.conf 主配置文件和/etc/xinetd.d 目录下的子配置文件来管理所有服务。主配置文件包含的是通用选项，这些选项将被所有子配置文件继承。不过子配置文件可以覆盖这些选项。每一个子配置文件用于设置一个子服务的参数。


![xinted.png](https://i.loli.net/2021/09/14/LXYWxC91zoT2Bsm.png)



#### 8.2 xinted工作流程

![xinted工作流程.png](https://i.loli.net/2021/09/14/ce9CJFQBUjANVZl.png)








## 信号

信号是由用户、系统或者进程发送给目标进程的信息，以通知目标进程某个状态的改变或系统异常。Linux 信号可由如下条件产生:
- 对于前台进程，用户可以通过输入特殊的终端字符来给它发送信号。比如输入Ctrl+C；
- 系统异常。比如浮点异常和非法内存段访问。
- 系统状态变化。比如 alarm定时器到期将引起SIGALRM信号。
- 运行kill命令或调用kill函数。
服务器程序必须处理(或至少忽略）一些常见的信号，以免异常终止。

### 1. Linux信号概述

#### 1.1 发送信号

Linux 下，一个进程给其他进程发送信号的 API是 kill 函数。其定义如下∶

```cpp
#include <sys/types.h>
#include <signal.h>
int kill(pid_t pid. int sig);
```

该函数把信号 sig 发送给目标进程;目标进程由 pid参数指定，其可能的取值及含义如表所示:

<div align="center" style="font-weight:900;font-size:larger">kill函数的pid参数及其含义 </div>

<div align="center">
	<table align="center">
		<tr style="font-size:large;font-weight:700">
			<td>pid参数</td>
			<td>含义</td>
		</tr>
		<tr>
			<td>pid > 0 </td>
			<td> 信号发送给PID为pid的进程 </td
		</tr>
		<tr>
			<td>pid = 0</td>
			<td>信号发送给本进程组的其他进程</td>
		</tr>
		<tr>
			<td>pid = -1</td>
			<td>信号发送给init进程之外的所有进程，但发送者需要拥有权限</td>
		</tr>
		<tr>
			<td>pid < -1 </td>
			<td>信号发送给组ID 为-pid的进程组中的所有成员</td>
		</tr>
	</table>
</div>


Linux 定义的信号值都大于 0，如果 sig取值为 0，则 kill 函数不发送任何信号。

该函数成功时返回 0，失败则返回 -1 并设置 errno。几种可能的 errno 如表所示:

<div align="center" style="font-weight:900;font-size:larger">kill函数出错的情况</div>

<div align="center">
	<table align="center">
		<tr style="font-size:large;font-weight:700">
			<td>errno</td>
			<td>含义</td>
		</tr>
		<tr>
			<td>EINVAL</td>
			<td>无效的信号</td>
		</tr>
		<tr>
			<td>EPERM</td>
			<td>该进程没有给另外的进程发送信号的权限</td>
		</tr>
		<tr>
			<td>ESRCH</td>
			<td>目标进程或进程组不存在</td>
		</tr>
	</table>
</div>


#### 1.2 信号处理方式

目标进程在收到信号时，需要定义一个接收函数来处理之。信号处理函数的原型如下：
```cpp
#include <signal.h>
typedef void (*__sighandler_t)(int);
```
信号处理函数只带有一个整型参数，该参数用来指示信号类型。信号处理函数应该是可重入的，否则很容易引发一些竞态条件。所以在信号处理函数中严禁调用一些不安全的函数。

除了用户自定义信号处理函数外，bits/signum.h头文件中还定义了信号的两种其他处理方式——SIG_IGN 和 SIG_DEL∶

```cpp
#inclue <bits/signum.h>
#define SIG_DEL ((__sighandler_t) 0)
#define SIG_IGN ((__sighandler_t) 1)
```

SIG_IGN 表示忽略目标信号，SIG_DFL 表示使用信号的默认处理方式。信号的默认处理方式有如下几种∶==结束进程（Term）、忽略信号（Ign）、结束进程并生成核心转储文件（Core）、暂停进程（Stop），以及继续进程（Cont）。==


#### 1.3 Linux信号

Linux 的可用信号都定义在 bits/signum.h 头文件中，其中包括标准信号和 POSIX 实时信号。

#### 1.4 中断系统调用
如果程序在执行处于阻塞状态的系统调用时接收到信号，并且我们为该信号设置了信号处理函数，则默认情况下系统调用将被中断，并且errno 被设置为 EINTR。我们可以使用 sigaction 函数为信号设置 SA RESTART标志以自动重启被该信号中断的系统调用。

对于默认行为是暂停进程的信号（比如 SIGSTOP、SIGTTIN），如果我们没有为它们设置信号处理函数，则它们也可以中断某些系统调用（比如 connect、epoll wait）。


### 2.信号函数

#### 2.1signal系统调用
要为一个信号设置处理函数，可以使用下面的 signal 系统调用：
```cpp
#include <signal.h>
_sighandelr_t signal(int sig, _sighandler_t _handler);
```
sig 参数指出要捕获的信号类型。_handler 参数是_sighandler_t 类型的函数指针，用于指定信号 sig 的处理函数。
signal 函数成功时返回一个函数指针，该函数指针的类型也是_sighandler t。这个返回值是前一次调用 signal 函数时传入的函数指针，或者是信号 sig对应的默认处理函数指针SIG DEF（如果是第一次调用 signal 的话）。
signal 系统调用出错时返回 SIG_ERR，并设置 errno。


#### 2.2 sigaction系统调用
设置信号处理函数的更安全的接口是如下的系统调用∶
```cpp
#include <signal.h>
int sigaction(int sig, const struct sigaction* act, struct sigaction* oact);
```

sig 参数指出要捕获的信号类型，act 参数指定新的信号处理方式，oact 参数则输出信号先前的处理方式（如果不为 NULL 的话）。act 和 oact都是 sigaction结构体类型的指针，
sigaction 结构体描述了信号处理的细节，其定义如下∶

```cpp
struct aigaction
{
#ifdef __USE_POSIX199309
	union
	{
		_sighandler_t sa_handler;
		void (*sa_sigaction) (int, siginfo_t*, void*);
	}_sigaction_handler;

#define sa_handler  __sigaction_handler.sa_handler
#define sa_sgaction	__sigaction_handler.sig_action

#else
	__sighandler_t sa_handler;
#endif
	
	__sigset_t	sa_mask;
	int sa_flags;
	void (*sa_restorer) (void);
};
```

该结构体中的 sa_hander 成员指定信号处理函数。sa_mask 成员设置进程的信号掩码（确切地说是在进程原有信号掩码的基础上增加信号掩码），以指定哪些信号不能发送给本进程。sa mask是信号集 sigset t（ sigset t的同义词）类型，该类型指定一组信号.sa flags 成员用于设置程序收到信号时的行为，其可选值如表所示。


<div align="center" style="font-weight:900;font-size:larger">sa_flags选项</div>

<div align="center">
	<table align="center">
		<tr style"font-size:large;font-weight:700">
			<td>选项</td>
			<td>含义</td>
		</tr>
		<tr>
			<td>SA_NOCLDSTOP</td>
			<td>如果siaction的sig参数是SICGHLD,则设置该标志表示子进程暂停时不生成SIGCHLD信号<td>
		</tr>
		<tr>
			<td>SA_NOCLDWAIT</td>
			<td>如果 sigaction的 sig 参数是 SIGCHLD，则设置该标志表示子进程结束时不产生僵尸进程</td>
		</tr>
		<tr>
			<td>SA_SIGINFO</td>
			<td>使用 sa_sigaction作为信号处理函数（而不是默认的 sa_handler），它给进程提供更多相关的信息</td>
		</tr>
		<tr>
			<td>SA_ONSTACK</td>
			<td>调用由 sigaltstack 函数设置的可选信号栈上的信号处理函数</td>
		</tr>
		<tr>
			<td>SA_RESTART</td>
			<td>重新调用被该信号终止的系统调用</td>
		</tr>
		<tr>
			<td>SA_NODEFER</td>
			<td>当接收到信号并进入其信号处理函数时，不屏蔽该信号。默认情况下，我们期望进程在处理一个信号时，不再接受同类型信号，避免竟态条件</td>
		</tr>
		<tr>
			<td>SA_RESETHAND</td>
			<td>信号处理函数执行完以后，恢复信号的默认处理方式</td>
		</tr>
		<tr>
			<td>SA_INTERRUPT</td>
			<td>中断系统调用</td>
		</tr>
		<tr>
			<td>SA_NOMASK</td>
			<td>同SA_NODEFER</td>
		</tr>
		<tr>
			<td>SA_ONESHOT</td>
			<td>同SA_RESETHAND</td>
		</tr>
		<tr>
			<td>SA_STACK</td>
			<td>同SA_ONSTACK</td>
		</tr>
	</table>
</div>


==sa_restorer 成员已经过时，最好不要使用==。sigaction成功时返回0，失败则返回-1并设置errmo



### 3.信号集
#### 3.1 信号集函数
Linux 使用数据结构 sigset_t来表示一组信号。其定义如下∶
```cpp
#include <bits/sigset.h>
#define _SIGSET_NWORDS (1024 / (8 * sizeof(unsigned long int)))
typedef struct
{
	unsigned long int __val[__SIGSET_NWORDS];
}__sigset_t;
```


由该定义可见，sigset_t实际上是一个长整型数组，数组的每个元素的每个位表示一个信号。这种定义方式和文件描述符集 fd_set类似。Linux 提供了如下一组函数来设置、修改删除和查询信号集∶

```cpp
include <signal.h>
int sigemptyset (sigset_t*_set);	/*清空信号集 */
int sigfillset (sigset_t* _set);	/* 在信号集中设置所有信号*/
int sigaddset (sigset_t*__set, fnt _signo); /* 将信号 signo添加至信号集中*/
int sigdelset (sigset_t* __set, int _signo);  /*将信号_signo 从信号集中剧除*/
int sigismember (_const sigset_t* __set，int signo);/* 测试 signo 是否在信号集中 */
```

#### 3.2进程信号掩码
我们可以利用 sigaction 结构体的 sa mask 成员来设置进程的信号掩码。此外，如下函数也可以用于设置或查看进程的信号掩码∶

```cpp
#include <signal.h>
int sigprocmask(int _how, const sigset_t* _set, sigset* _oset);
```

set 参数指定新的信号掩码，_oset 参数则输出原来的信号掩码（如果不为 NULL 的话）。如果 set参数不为 NULL，则 how 参数指定设置进程信号掩码的方式，其可选值如表所示。


<div align="center" style="font-weight:900;font-size:larger">_how参数</div>

<div align="center">
	<table align="center">
		<tr style"font-size:large;font-weight:700">
			<td>_how参数</td>
			<td>含义</td>
		</tr>
		<tr>
			<td>SIG_BLOCK</td>
			<td>新的进程信号掩码是其当前值和_set 指定信号集的并集</td>
		</tr>
		<tr>
			<td>SIG_UNBLOCK</td>
			<td>新的进程信号掩码是其当前值和～_set 信号集的交集，因此_set 指定的信号集将不被屏蔽</td>
		</tr>
		<tr>
			<td>SIG_SETMASK</td>
			<td>直接将进程信号掩码设置为_set</td>
		</tr>
	</table>
</div>


如果_set 为 NULL，则进程信号掩码不变，此时我们仍然可以利用 oset 参数来获得进程当前的信号掩码。
sigprocmask 成功时返回 0，失败则返回 -1并设置errno。

#### 3.3被挂起的信号
设置进程信号掩码后，被屏蔽的信号将不能被进程接收。如果给进程发送一个被屏蔽的信号，则操作系统将该信号设置为进程的一个被挂起的信号。如果我们取消对被挂起信号的屏蔽，则它能立即被进程接收到。如下函数可以获得进程当前被挂起的信号集∶
```cpp
#include <signal.h>
int sigpending(sigset_t* set);
```

set 参数用于保存被挂起的信号集。显然，进程即使多次接收到同一个被挂起的信号，sigpending 函数也只能反映一次。并且，当我们再次使用 sigprocmask 使能该挂起的信号时，该信号的处理函数也只被触发一次。
sigpending 成功时返回 0，失败时返回 -1并设置 errno。

### 4.统一事件源
信号是一种异步事件∶ 信号处理函数和程序的主循环是两条不同的执行路线。很显然，信号处理函数需要尽可能快地执行完毕，以确保该信号不被屏蔽（前面提到过，为了避免
一些竞态条件，信号在处理期间，系统不会再次触发它）太久。一种典型的解决方案是∶==把信号的主要处理逻辑放到程序的主循环中，当信号处理函数被触发时，它只是简单地通知主循环程序接收到信号，并把信号值传递给主循环，主循环再根据接收到的信号值执行目标信号对应的逻辑代码。==
信号处理函数通常使用管道来将信号"传递"给主循环 ∶ 信号处理函数往管道的写端写入信号值，主循环则从管道的读端读出该信号值。那么主循环怎么知道管道上何时有数据可读呢?这很简单，我们只需要使用 I/O复用系统调用来监听管道的读端文件描述符上的可读事件。如此一来，信号事件就能和其他 I/O事件一样被处理，即统一事件源。


<div align="center" style="font-weight:900;font-size:larger">统一事件源</div>

```cpp
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>

#define MAX_EVENT_NUMBER 1024
static int pipefd[2];	/* 信号处理函数和主循环通信管道 */

int setnonblocking( int fd )
{
    int old_option = fcntl( fd, F_GETFL );
    int new_option = old_option | O_NONBLOCK;
    fcntl( fd, F_SETFL, new_option );
    return old_option;
}

void addfd( int epollfd, int fd )
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET;
    epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
    setnonblocking( fd );
}

/* 信号处理函数 */
void sig_handler( int sig )
{
	/* 保留原来的errno，在函数最后恢复，以保证函数的可重入性*/
    int save_errno = errno;
    int msg = sig;
    send( pipefd[1], ( char* )&msg, 1, 0 );
    errno = save_errno;
}

/* 设置信号的处理函数*/
void addsig( int sig )
{
    struct sigaction sa;
    memset( &sa, '\0', sizeof( sa ) );
    sa.sa_handler = sig_handler;
    sa.sa_flags |= SA_RESTART;
    sigfillset( &sa.sa_mask );
    assert( sigaction( sig, &sa, NULL ) != -1 );
}

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    int ret = 0;
    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
    assert( listenfd >= 0 );

    ret = bind( listenfd, ( struct sockaddr* )&address, sizeof( address ) );
    if( ret == -1 )
    {
        printf( "errno is %d\n", errno );
        return 1;
    }

    ret = listen( listenfd, 5 );
    assert( ret != -1 );

    epoll_event events[ MAX_EVENT_NUMBER ];
    int epollfd = epoll_create( 5 );
    assert( epollfd != -1 );
    addfd( epollfd, listenfd );

	/*使用 socketpair 创建管道，注册 pipefd[0] 上的可读事件*/
    ret = socketpair( PF_UNIX, SOCK_STREAM, 0, pipefd );
    assert( ret != -1 );
    setnonblocking( pipefd[1] );
    addfd( epollfd, pipefd[0] );

    // add all the interesting signals here
    addsig( SIGHUP );
    addsig( SIGCHLD );
    addsig( SIGTERM );
    addsig( SIGINT );
    bool stop_server = false;

    while( !stop_server )
    {
        int number = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, -1 );
        if ( ( number < 0 ) && ( errno != EINTR ) )
        {
            printf( "epoll failure\n" );
            break;
        }
    
        for ( int i = 0; i < number; i++ )
        {
            int sockfd = events[i].data.fd;
			/* 如果就绪的文件描述符是listenfd，则处理新的连接 */
            if( sockfd == listenfd )
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof( client_address );
                int connfd = accept( listenfd, ( struct sockaddr* )&client_address, &client_addrlength );
                addfd( epollfd, connfd );
            }
			/* 如果就绪的文件描述符是 pipefd[0]，则处理信号 */
            else if( ( sockfd == pipefd[0] ) && ( events[i].events & EPOLLIN ) )
            {
                int sig;
                char signals[1024];
                ret = recv( pipefd[0], signals, sizeof( signals ), 0 );
                if( ret == -1 )
                {
                    continue;
                }
                else if( ret == 0 )
                {
                    continue;
                }
				/*因为每个信号值占1字节，所以按字节来逐个接收信号。我们以 SIGTERM 为例，来说明如何安全地终止服务器主循环 */
                else
                {
                    for( int i = 0; i < ret; ++i )
                    {
                        //printf( "I caugh the signal %d\n", signals[i] );
                        switch( signals[i] )
                        {
                            case SIGCHLD:
                            case SIGHUP:
                            {
                                continue;
                            }
                            case SIGTERM:
                            case SIGINT:
                            {
                                stop_server = true;
                            }
                        }
                    }
                }
            }
            else
            {
            }
        }
    }

    printf( "close fds\n" );
    close( listenfd );
    close( pipefd[1] );
    close( pipefd[0] );
    return 0;
}

```


### 5.网络编成相关信号

#### 5.1 SIGHUP
当挂起进程的控制终端时，SIGHUP 信号将被触发。对于没有控制终端的网络后台程序而言，它们通常利用SIGHUP 信号来强制服务器重读配置文件。

xinetd程序在接收到SIGHUP信号之后将调用hard_reconfig 函数（见xinetd源码),它循环读取/etc/xinetd.d/目录下的每个子配置文件，并检测其变化。如果某个正在运行的子服务的配置文件被修改以停止服务，则xinetd主进程将给该子服务进程发送SIGTERM信号以结束它。如果某个子服务的配置文件被修改以开启服务，则xinetd将创建新的socket并将其绑定到该服务对应的端口上。


#### 5.2 SIGPIPE
默认情况下，往一个读端关闭的管道或 socket 连接中写数据将引发 SIGPIPE 信号。我们需要在代码中捕获并处理该信号，或者至少忽略它，因为程序接收到 SIGPIPE 信号的默认行为是结束进程，而我们绝对不希望因为错误的写操作而导致程序退出。引起 SIGPIPE 信号的写操作将设置 errno为 EPIPE。

第5章提到，我们可以使用send函数的MSG_NOSIGNAL标志来禁止写操作触发SIGPIPE信号。在这种情况下，我们应该使用send 函数反馈的errno值来判断管道或者socket连接的读端是否已经关闭。
此外，我们也可以利用IO复用系统调用来检测管道和socket连接的读端是否已经关闭。以poll 为例，当管道的读端关闭时，写端文件描述符上的POLLHUP事件将被触发﹔当socket连接被对方关闭时，socket上的 POLLRDHUP事件将被触发。


#### 5.3 SIGURG

在 Linux 环境下，内核通知应用程序带外数据到达主要有两种方法∶一种是第 9 章介绍的 I/O 复用技术，select 等系统调用在接收到带外数据时将返回，并向应用程序报告 socket上的异常事件，[代码清单](##### 1.3 处理带外数据)给出了一个这方面的例子;另外一种方法就是使用 SIGURG 信号，如以下代码所示。

```cpp

#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <signal.h>
#include <fcntl.h>

#define BUF_SIZE 1024

static int connfd;


/* STGURG信号的处理函数 */
void sig_urg( int sig )
{
    int save_errno = errno;
    
    char buffer[ BUF_SIZE ];
    memset( buffer, '\0', BUF_SIZE );
    int ret = recv( connfd, buffer, BUF_SIZE-1, MSG_OOB );
    printf( "got %d bytes of oob data '%s'\n", ret, buffer );

    errno = save_errno;
}

void addsig( int sig, void ( *sig_handler )( int ) )
{
    struct sigaction sa;
    memset( &sa, '\0', sizeof( sa ) );
    sa.sa_handler = sig_handler;
    sa.sa_flags |= SA_RESTART;
    sigfillset( &sa.sa_mask );
    assert( sigaction( sig, &sa, NULL ) != -1 );
}

int main( int argc, char* argv[] )
{
    if( argc <= 2 )
    {
        printf( "usage: %s ip_address port_number\n", basename( argv[0] ) );
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi( argv[2] );

    struct sockaddr_in address;
    bzero( &address, sizeof( address ) );
    address.sin_family = AF_INET;
    inet_pton( AF_INET, ip, &address.sin_addr );
    address.sin_port = htons( port );

    int sock = socket( PF_INET, SOCK_STREAM, 0 );
    assert( sock >= 0 );

    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );
    assert( ret != -1 );

    ret = listen( sock, 5 );
    assert( ret != -1 );

    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof( client );
    connfd = accept( sock, ( struct sockaddr* )&client, &client_addrlength );
    if ( connfd < 0 )
    {
        printf( "errno is: %d\n", errno );
    }
    else
    {
        addsig( SIGURG, sig_urg );
		/* 使用 SIGURG信号之前，我们必须设置 socket的宿主进程或进程组 */
        fcntl( connfd, F_SETOWN, getpid() );

        char buffer[ BUF_SIZE ];
        while( 1 )			/* 循环接受普通数据 */
        {
            memset( buffer, '\0', BUF_SIZE );
            ret = recv( connfd, buffer, BUF_SIZE-1, 0 );
            if( ret <= 0 )
            {
                break;
            }
            printf( "got %d bytes of normal data '%s'\n", ret, buffer );
        }

        close( connfd );
    }

    close( sock );
    return 0;
}

```





## 定时器

网络程序需要处理的第三类事件是定时事件，比如定期检测一个客户连接的活动状态。服务器程序通常管理着众多定时事件，因此有效地组织这些定时事件，使之能在预期的时间点被触发且不影响服务器的主要逻辑，对于服务器的性能有着至关重要的影响。为此，我们要将每个定时事件分别封装成定时器，并使用某种容器类数据结构，比如链表、排序链表和时间轮，将所有定时器串联起来，以实现对定时事件的统一管理。本章主要讨论的就是两种高效的管理定时器的容器:时间轮和时间堆。

不过，在讨论如何组织定时器之前，我们先要介绍定时的方法。定时是指在一段时间之后触发某段代码的机制，我们可以在这段代码中依次处理所有到期的定时器。换言之，定时机制是定时器得以被处理的原动力。Linux提供了三种定时方法，它们是:
- socket 选项 SO_RCVTIMEO 和 SO_SNDTIMEO。
- SIGALRM信号。
- I/O复用系统调用的超时参数。


