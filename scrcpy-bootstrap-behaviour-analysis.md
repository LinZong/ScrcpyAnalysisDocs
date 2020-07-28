## scrcpy 启动流程分析

### 写在前面

scrcpy 是一款可以把安卓手机的屏幕投到电脑上，并支持在电脑端对手机进行操作（包括但不限于模拟触控，拖拽传文件，安装Apk，录屏等）的跨平台工具软件。相比于市面上的其他软件，scrcpy做到了轻量，便携，且不需要显式地在手机端安装任何辅助软件。

笔者使用这款软件已经有一段时间了，对这款软件的实现比较感兴趣，但苦于一直没有时间对这款软件的具体实现进行剖析总结，一直感到有点遗憾。趁着最近手头上的工作闲了下来，特意花了点时间下载了scrcpy的源码进行阅读。并总结了一些有关于scrcpy实现的一些心得体会，不敢独享，故写下了这篇文章。

**注意，在读源码之前，建议先按照scrcpy的文档配置好编译环境并且自己动手编译一次scrcpy的client和server。这样有助于配置IDE的代码高亮功能。**

**注意，本文的源码阅读环境为macOS 10.15，并不会去分析不属于本平台的代码段。**阅读的代码分支及版本为:[master 5086e7b](https://github.com/Genymobile/scrcpy/commit/5086e7b744e9edfa9d1d56cf5ae2ad3b0ae32ddf)。

### 0. 解析启动参数

scrcpy是使用命令行参数来控制程序启动后的行为的，所以程序启动的第一件事就是去解析用户传了哪些参数给scrcpy。在这里scrcpy的做法是：先设置一套普遍使用且能保证程序正常启动的默认值，再从argc和argv中解析出用户需要进行自定义的参数，在覆盖掉默认值的同时会顺便校验参数是否合法，如果参数不合法则拒绝启动。

main函数入口定义了一个保存启动参数的结构体: 

```c
struct scrcpy_cli_args args = {
        .opts = SCRCPY_OPTIONS_DEFAULT,
        .help = false,
        .version = false,
};
```

``SCRCPY_OPTIONS_DEFALT``宏定义了程序启动时所需的所有参数的默认值：

```c
#define SCRCPY_OPTIONS_DEFAULT { \
    .serial = NULL, \
    .crop = NULL, \
    .record_filename = NULL, \
    .window_title = NULL, \
    .push_target = NULL, \
    .render_driver = NULL, \
    .codec_options = NULL, \
    .log_level = SC_LOG_LEVEL_INFO, \
    .record_format = RECORDER_FORMAT_AUTO, \
    .port_range = { \
        .first = DEFAULT_LOCAL_PORT_RANGE_FIRST, \
        .last = DEFAULT_LOCAL_PORT_RANGE_LAST, \
    }, \
    .max_size = DEFAULT_MAX_SIZE, \
    .bit_rate = DEFAULT_BIT_RATE, \
    .max_fps = 0, \
    .lock_video_orientation = DEFAULT_LOCK_VIDEO_ORIENTATION, \
    .rotation = 0, \
    .window_x = WINDOW_POSITION_UNDEFINED, \
    .window_y = WINDOW_POSITION_UNDEFINED, \
    .window_width = 0, \
    .window_height = 0, \
    .display_id = 0, \
    .show_touches = false, \
    .fullscreen = false, \
    .always_on_top = false, \
    .control = true, \
    .display = true, \
    .turn_screen_off = false, \
    .render_expired_frames = false, \
    .prefer_text = false, \
    .window_borderless = false, \
    .mipmaps = true, \
    .stay_awake = false, \
    .force_adb_forward = false, \
}
```

这些参数的命名言简意赅，通俗易懂。如果读者们想知道参数具体的含义，可以``scrcpy --help 2>&1 | grep <keywords>``看解释。

（为什么要 ``2>&1``呢？那是因为``cli.c``文件中定义了`fprintf(stderr, "...")`，默认帮助信息是输出到标准错误流的，而grep是从标准输入流读的，所以要做流的重定向。）

然后进入到方法``scrcpy_parse_args(&args, argc,argv)``：

此方法主要是使用了`getopt_long`方法来帮助解析从命令行传入的参数。这里稍微解释一下`getopt_long(argc, argv. required_param_mask, long_options, int* longindex)`：

>`argc`: **argument count**, 记录了命令行参数的个数(包括命令本身)
>
>`argv`: **argument vector**, 记录了命令行参数的具体内容
>
>`optstring`: 作为`getopt()`的第三个参数，用于规定**合法选项**(option)以及选项是否带**参数**(argument)。一般为合法选项字母构成的字符串，如果字母后面带上冒号`:`就说明该选项**必须**有参数。如`"ht:"`说明有两个选项`-h`和`-t`且后者(`-t`)必须带有参数(如`-t 60`)。
>
>`longopts数组`: 用于规定**合法长选项**以及长选项是否带**参数**(argument)。涉及到的`option结构体`声明如下
>
>`longindex`: 一般设置为`NULL`; 如果不为`NUll`, 指向每次找到的**长选项**在`longopts`的位置，可以通过该值(即索引)找到当前**长选项**的具体信息。

其中``long_options``数组中的`option`结构体定义如下:

```c
struct option {
	/* name of long option */
	const char *name;
	/*
	 * one of no_argument, required_argument, and optional_argument:
	 * whether option takes an argument
	 */
	int has_arg;
	/* if not NULL, set *flag to val when option found */
	int *flag;
	/* if flag not NULL, value to set *flag to; else return value */
	int val;
};
```

因此，`getopt_long`的工作模式就非常简单：

* getopt_long读到一个长选项或者是短选项的时候，返回短选项的字母或者是长选项的val。
* 选项的具体值是存在全局变量``optarg``里面，指向下一个待处理变量的下标存在`optind`全局变量中。
* 根据`getopt_long`的返回值可以知道现在正在处理的是哪个参数，根据对应的参数写对应的解析逻辑，用一个大switch语句块路由到对应的参数处理代码，处理完后存进`scrcpy_options`里。

如果参数解析失败，`scrcpy_parse_args`将会返回false, 在外层就能做到如果传入的参数不合法，程序就会立马退出，不再进行接下来的初始化逻辑了。

### 1. 解析日志等级、libavformat网络库初始化

完成参数解析，并且把参数保存到`scrcpy_cli_args`结构里面后，接下来是解析日志输出等级。目前scrcpy底层的日志实现还是使用SDL自带的日志框架，需要做日志等级字符串 -> scrcpy内部enum -> SDL日志等级enum的转换。具体转换方法如下所示:

```c
static bool
parse_log_level(const char *s, enum sc_log_level *log_level)
{
    if (!strcmp(s, "debug"))
    {
        *log_level = SC_LOG_LEVEL_DEBUG;
        return true;
    }

    if (!strcmp(s, "info"))
    {
        *log_level = SC_LOG_LEVEL_INFO;
        return true;
    }

    if (!strcmp(s, "warn"))
    {
        *log_level = SC_LOG_LEVEL_WARN;
        return true;
    }

    if (!strcmp(s, "error"))
    {
        *log_level = SC_LOG_LEVEL_ERROR;
        return true;
    }

    LOGE("Could not parse log level: %s", s);
    return false;
}
```

转换完成后，会返回一个`SDL_LogPriority`，随后调用

```c
SDL_LogSetAllPriority(sdl_log);
```

设置日志等级。

接下来的两个if比较简单：如果用户需要看help，就打印帮助文档，退出；如果用户需要看版本号，就打印版本号，退出。

否则，继续向下执行，调用`avformat_network_init`初始化`libavformat`的网络相关库。看注释可以发现其实这个调用已经是被废弃的，仅仅是用来兼容当libavformat连接到旧版本的网络库的时候，解决一些旧版GnuTLS或者OpenSSL库的线程安全问题而已。

```c
/**
 * Do global initialization of network libraries. This is optional,
 * and not recommended anymore.
 *
 * This functions only exists to work around thread-safety issues
 * with older GnuTLS or OpenSSL libraries. If libavformat is linked
 * to newer versions of those libraries, or if you do not use them,
 * calling this function is unnecessary. Otherwise, you need to call
 * this function before any other threads using them are started.
 *
 * This function will be deprecated once support for older GnuTLS and
 * OpenSSL libraries is removed, and this function has no purpose
 * anymore.
 */
int avformat_network_init(void);
```

### 2. server 启动

在介绍 server 启动之前，有必要对scrcpy的工作模式做一个简单的了解。

scrcpy是一个标准的C/S模式应用程序，server侧是一个在Android的shell下运行的Java程序，client侧则是在电脑上运行的一个C程序。可能会有读者觉得奇怪，为什么手机侧的软件是server呢？不应该是电脑作为server，然后手机侧的client连接到server, 向server回传信息吗？有关这个问题，scrcpy的开发人员们是这样看待的：他们认为手机端不断的推画面流到电脑上，手机不断的"serving video stream"，因此它应该是一个server，而电脑端其实是在"receiving and decoding video stream"，属于一个consumer，因此应该作为一个client。联想另外一种常见的C/S模型应用程序：Web服务器及浏览网页的浏览器，他们的C/S定位和scrcpy的工作模式是类似的；因此采用和他们一样的C/S定义标准其实是更加有意义的。

>不过要特别注意的是，上述关系也就仅在从数据发送接收的角度去定义server和client时有效；如果从网络的层面上看待"server"和"client"，那么手机和电脑的角色就反过来了：scrcpy默认使用的是adb reverse模式，它可以把电脑上的某个端口映射到手机上。因此当scrcpy以adb reverse模式启动的时候，电脑端的应用程序会打开一个端口，并将此端口映射到手机上，然后电脑开始监听这个端口，等待手机侧的连接。在网络层面上，监听端口接受连接的是server，而主动发起连接的是client。
>
>此外，如果当scrcpy以adb forward模式启动的时候，网络层面上的server和client又发生了变化：adb forward模式会把安卓的某个端口映射成电脑的端口 ，在这种模式下工作的scrcpy会先打开安卓端的一个端口，然后映射到电脑的本地端口，接下来手机端的程序就会监听刚才打开的端口，电脑侧的scrcpy不断的尝试连接这个端口。这个时候，手机侧的程序变成了server；相对的，电脑侧的程序就是client了。
>
>最后，本文讲解启动流程的时候将以adb reverse模式工作的scrcpy为基准，adb forward模式的介绍会一笔带过。因为在多数情况下，scrcpy 都是工作在adb reverse模式下的。

首先进行一下流程

```c
    bool record = !!options->record_filename;
    struct server_params params = {
        .log_level = options->log_level,
        .crop = options->crop,
        .port_range = options->port_range,
        .max_size = options->max_size,
        .bit_rate = options->bit_rate,
        .max_fps = options->max_fps,
        .lock_video_orientation = options->lock_video_orientation,
        .control = options->control,
        .display_id = options->display_id,
        .show_touches = options->show_touches,
        .stay_awake = options->stay_awake,
        .codec_options = options->codec_options,
        .force_adb_forward = options->force_adb_forward,
    };
    if (!server_start(&server, options->serial, &params)) {
        return false;
    }
```

进入server的启动逻辑。三个参数，server是一个定义在`scrcpy.c`文件的**全局变量**，options->serial则是在scrcpy启动时传入的设备序列号。（这在一台电脑同时连接了多个安卓设备或启动了多个安卓虚拟机的时候起到对连接设备的标识作用，params则是上面列举出的服务器启动所需的相关参数。

跟进`server_start`方法，

首先将服务器启动参数列表中的尝试端口范围保存到server结构中（端口范围默认是[27183,27199]）。

检测有没有指定serial，如果有，把serial复制一份，保存在server->serial中。复制不成功的话报错退出。

然后开始把手机端的应用程序（以下如果没有歧义，全部用server指代手机侧的scrcpy。如果出现歧义则会对歧义部分上下文加以澄清）通过adb命令push到手机侧。这个动作是由`push_server(serial)`方法实现的。

`push_server`内部首先会获取server的路径。获取的方法是先尝试读取环境变量`SCRCPY_SERVER_PATH`，如果读到了，就以这个环境变量的值为基准，复制一份，返回出去；否则就读取`DEFAULT_SERVER_PATH`（默认portable版本是`/usr/local/share/scrcpy/scrcpy-server`。这个路径通过宏组装，写死在了代码里）然后返回。将返回的路径通过`is_regular_file`校验，校验通过确实是正常的文件后，发出adb push命令，将`scrcpy-server` push到手机上（手机端默认的保存路径是 `DEVICE_SERVER_PATH /data/local/tmp/scrcpy-server.jar`，最后检查adb 命令是否成功完成了。如果没有成功完成，那么终止scrcpy client的运行。

Server 就绪后，client开始创建监听的socket。跟入`enable_tunnel_any_port(server, params->port_range,params->force_adb_forward)`，再跟`enable_tunnel_reverse_any_port`，初步看上去就能看到一个大for循环，本质上就是逐一尝试port_range中的全部端口。

先执行`adb_reverse`方法，本质上就是拼接一条这样的命令:

```shell
adb reverse localabstract:scrcpy tcp:27183
```

这个命令一般情况下都会成功的。如果程序执行到这条命令的时候失败了，那说明adb环境一定有严重的问题、这种情况下就不再进行其他端口的尝试了，一定都是失败的：

```c
for (;;) {
    if (!enable_tunnel_reverse(server->serial, port)) {
    // the command itself failed, it will fail on any port
        return false;
    }
   // rest...
}
```

当执行完`adb reverse`命令后，adb server就已经建立了一个电脑端口 -> 手机端口的映射（为了方便后文的讲解，这里假定scrcpy将电脑的27183映射到了手机的27183上，此端口号也是scrcpy默认端口范围中的第一个端口号），此时电脑端开始初始化socket，监听本地地址上的27183端口:

```c
static socket_t
listen_on_port(uint16_t port) {
#define IPV4_LOCALHOST 0x7F000001
    return net_listen(IPV4_LOCALHOST, port, 1);
}

socket_t
net_listen(uint32_t addr, uint16_t port, int backlog) {
    socket_t sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock == INVALID_SOCKET) {
        perror("socket");
        return INVALID_SOCKET;
    }

    int reuse = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, (const void *) &reuse,
                   sizeof(reuse)) == -1) {
        perror("setsockopt(SO_REUSEADDR)");
    }

    SOCKADDR_IN sin;
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(addr); // htonl() harmless on INADDR_ANY
    sin.sin_port = htons(port);

    if (bind(sock, (SOCKADDR *) &sin, sizeof(sin)) == SOCKET_ERROR) {
        perror("bind");
        net_close(sock);
        return INVALID_SOCKET;
    }

    if (listen(sock, backlog) == SOCKET_ERROR) {
        perror("listen");
        net_close(sock);
        return INVALID_SOCKET;
    }

    return sock;
}
```

一般而言起监听的server侧（注意，这里讨论的是网络层面上的sever，不是scrcpy的server）经历的流程不外乎是``socket() -> bind() -> listen() -> accept()``。

```c
int socket(int domain, int type, int protocol);
```

> - domain：即协议域，又称为协议族（family）。常用的协议族有，AF_INET、AF_INET6、AF_LOCAL（或称AF_UNIX，Unix域socket）、AF_ROUTE等等。协议族决定了socket的地址类型，在通信中必须采用对应的地址，如AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合、AF_UNIX决定了要用一个绝对路径名作为地址。
> - type：指定socket类型。常用的socket类型有，SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET等等。
> - protocol：指定协议。常用的协议有，IPPROTO_TCP、IPPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC等，它们分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议。
>
> 注意：并不是上面的type和protocol可以随意组合的，如SOCK_STREAM不可以跟IPPROTO_UDP组合。当protocol为0时，会自动选择type类型对应的默认协议。
>
> 当我们调用**socket**创建一个socket时，返回的socket描述字它存在于协议族（address family，AF_XXX）空间中，但没有一个具体的地址。如果想要给它赋值一个地址，就必须调用bind()函数，否则就当调用connect()、listen()时系统会自动随机分配一个端口。

阅读代码，不难发现scrcpy在此处的操作是先开了一个protocol domain为IPV4、socket type为TCP(SOCK_STREAM)，默认选择了TCP为stream socket的协议。然后针刚才创建出来的socket, 设置其工作在通用套接字层（SOL_SOCKET），设置socket的选项为可重用socket地址（SO_REUSEADDR）。

socket初始化完成之后，需要把socket绑定到 一个具体的地址上，此时scrcpy调用bind函数，把socket绑定到127.0.0.1:27183的地址上；绑定完成后开始监听这个socket。**代码运行至此，scrcpy的server侧其实已经能开始连接client的监听socket而不会出一些什么IOException，SOCKET_ERROR之类的错误，但是，此时client仍未进入到开始accept socket connection的代码段。这意味着client的socket也还一直在LISTENING的状态，是没办法响应server的connect()的！**Don't worry, we will see the socket accept event loop soon.

从`listen_on_port()`方法返回之后的代码段就只是一些socket创建失败的打日志，以及端口重试工作了。没有太大的特别。只要port range中的其中一个端口能正确启动adb reverse and create a socket and begin listening client's connection (network layer)，那么`enable_tunnel_any_port`的任务就完成了。

通过上面的源码阅读我们知道，scrcpy的client是可以接受用户传入一组端口来协商启动的，因此在没有真正创建出socket之前，程序是不知道到底选择了哪个端口的；在端口选好、socket完成创建之后，需要让scrcpy server开始连接client刚刚创建出来的socket。是的，接下来scrcpy将再次执行adb命令，启动scrcpy-server。

```c
static process_t
execute_server(struct server *server, const struct server_params *params) {
    char max_size_string[6];
    char bit_rate_string[11];
    char max_fps_string[6];
    char lock_video_orientation_string[5];
    char display_id_string[6];
    sprintf(max_size_string, "%"PRIu16, params->max_size);
    sprintf(bit_rate_string, "%"PRIu32, params->bit_rate);
    sprintf(max_fps_string, "%"PRIu16, params->max_fps);
    sprintf(lock_video_orientation_string, "%"PRIi8, params->lock_video_orientation);
    sprintf(display_id_string, "%"PRIu16, params->display_id);
    const char *const cmd[] = {
        "shell",
        "CLASSPATH=" DEVICE_SERVER_PATH,
        "app_process",
#ifdef SERVER_DEBUGGER
# define SERVER_DEBUGGER_PORT "5005"
# ifdef SERVER_DEBUGGER_METHOD_NEW
        /* Android 9 and above */
        "-XjdwpProvider:internal -XjdwpOptions:transport=dt_socket,suspend=y,server=y,address="
# else
        /* Android 8 and below */
        "-agentlib:jdwp=transport=dt_socket,suspend=y,server=y,address="
# endif
            SERVER_DEBUGGER_PORT,
#endif
        "/", // unused
        "com.genymobile.scrcpy.Server",
        SCRCPY_VERSION,
        log_level_to_server_string(params->log_level),
        max_size_string,
        bit_rate_string,
        max_fps_string,
        lock_video_orientation_string,
        server->tunnel_forward ? "true" : "false",
        params->crop ? params->crop : "-",
        "true", // always send frame meta (packet boundaries + timestamp)
        params->control ? "true" : "false",
        display_id_string,
        params->show_touches ? "true" : "false",
        params->stay_awake ? "true" : "false",
        params->codec_options ? params->codec_options : "-",
    };
#ifdef SERVER_DEBUGGER
    LOGI("Server debugger waiting for a client on device port "
         SERVER_DEBUGGER_PORT "...");
    // From the computer, run
    //     adb forward tcp:5005 tcp:5005
    // Then, from Android Studio: Run > Debug > Edit configurations...
    // On the left, click on '+', "Remote", with:
    //     Host: localhost
    //     Port: 5005
    // Then click on "Debug"
#endif
    return adb_execute(server->serial, cmd, sizeof(cmd) / sizeof(cmd[0]));
}
```

可能有读者读完这段代码后会感到困惑，诶，为什么它不把刚才选择的端口用启动参数的方式传给server啊？不然的话server怎么知道刚才创建socket的时候选择的到底是哪个端口啊？

其实这个问题的答案非常简单，还记得上面的adb reverse命令吗？

```shell
adb reverse localabstract:scrcpy tcp:27183
```

这句命令会把刚才我们创建的本地socket命名为"scrcpy"。这个名字在client侧和server侧都是通的，因此server侧只要执行

```java
// ...before
private static final String SOCKET_NAME = "scrcpy";
// SOCKRT
videoSocket = connect(SOCKET_NAME);
try {
     controlSocket = connect(SOCKET_NAME);
} catch (IOException | RuntimeException e) {
     videoSocket.close();
     throw e;
}
// ...rest
```

就可以连接到client创建的socket了。其他的server启动命令就平淡无奇了，只是把client接收到的一些参数通过命令行参数的方式发给server而已。

接下来，client 创建了一个守护线程，代码如下所示

```c
// If the server process dies before connecting to the server socket, then
    // the client will be stuck forever on accept(). To avoid the problem, we
    // must be able to wake up the accept() call when the server dies. To keep
    // things simple and multiplatform, just spawn a new thread waiting for the
    // server process and calling shutdown()/close() on the server socket if
    // necessary to wake up any accept() blocking call.

/*
	当server进程在开始连接client的socket之前就死掉了的话，
*/
    server->wait_server_thread =
        SDL_CreateThread(run_wait_server, "wait-server", server);
    if (!server->wait_server_thread) {
        cmd_terminate(server->process);
        cmd_simple_wait(server->process, NULL); // ignore exit code
        goto error2;
    }
```



### 3. SDL 初始化及配置

### 4. 连接到server

### 5. 读取配置信息

### 6. 杂项初始化

### 7. 手机画面推流初始化

### 8. 控制器初始化

### 9. 让SDL开始渲染画面

### 10. 控制器发出预设指令（如有）

### 11. 进入全屏模式（如果需要）

### 12. 进入事件循环

### 13. 应用退出前收尾



