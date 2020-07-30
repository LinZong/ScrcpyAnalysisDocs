## scrcpy 启动流程分析

### 写在前面

scrcpy 是一款可以把安卓手机的屏幕投到电脑上，并支持在电脑端对手机进行操作（包括但不限于模拟触控，拖拽传文件，安装Apk，录屏等）的跨平台工具软件。相比于市面上的其他软件，scrcpy做到了轻量，便携，且不需要显式地在手机端安装任何辅助软件。

笔者使用这款软件已经有一段时间了，对这款软件的实现比较感兴趣，但苦于一直没有时间对这款软件的具体实现进行剖析总结，一直感到有点遗憾。趁着最近手头上的工作闲了下来，特意花了点时间下载了scrcpy的源码进行阅读。并总结了一些有关于scrcpy实现的一些心得体会，不敢独享，故写下了这篇文章。

**注意，在读源码之前，建议先按照scrcpy的文档配置好编译环境并且自己动手编译一次scrcpy的client和server。这样有助于配置IDE的代码高亮功能。**

**注意，本文的源码阅读环境为macOS 10.15，并不会去分析不属于本平台的代码段。** 阅读的代码分支及版本为:[master 5086e7b](https://github.com/Genymobile/scrcpy/commit/5086e7b744e9edfa9d1d56cf5ae2ad3b0ae32ddf)。

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
    server->wait_server_thread =
        SDL_CreateThread(run_wait_server, "wait-server", server);
    if (!server->wait_server_thread) {
        cmd_terminate(server->process);
        cmd_simple_wait(server->process, NULL); // ignore exit code
        goto error2;
    }
```

> 当server进程在开始连接client的socket之前就死掉了的话，client会一直停在accpet()调用上(As we all know, accpet() is a blocking call)。为了解决这个问题，我们必须在server死掉的时候唤醒accpet()调用。为了使得这个动作简单，并且能方便的跨平台，我们只是简单开多一条线程，等待server的进程死了之后，在必要的情况下，调用socket的shutdown()/close()方法唤醒accpet()的阻塞调用。
>
> PS: adb shell那句命令执行之后，只要server进程没死，这一句命令调用是会一直阻塞等待返回的。因此只要开条新线等待adb shell命令返回，就能立马知道server是不是死了。

完成这个操作的代码也很简单，如下所示:

```c
static int
run_wait_server(void *data) {
    struct server *server = data;
    cmd_simple_wait(server->process, NULL); // ignore exit code
    // no need for synchronization, server_socket is initialized before this
    // thread was created
    if (server->server_socket != INVALID_SOCKET
            && !atomic_flag_test_and_set(&server->server_socket_closed)) {
        // On Linux, accept() is unblocked by shutdown(), but on Windows, it is
        // unblocked by closesocket(). Therefore, call both (close_socket()).
        close_socket(server->server_socket);
    }
    LOGD("Server terminated");
    return 0;
}
```

这里重点关注if的条件判断:如果监听server连接的socket已经被创建了，但是`server->server_socket_closed`标志并没有置起来，那么现在的情况就是server还没连接过client就死了，这个时候需要触发`close_socket(server_socket)`。

上面的代码正常执行完成后，如果不出意外，就会设上`server->tunnel_enabled=true`，然后返回。否则就是一些异常处理的分支。比较简单，读者们有空的话看看就好。

执行至此，server的初始化就完成了。

### 3. SDL 初始化及配置

SDL的初始化调用的方法为`sdl_init_and_configure(options->display, options->render_driver)`，这个方法是比较简单的，贴代码也没什么成本，如下所示：（反正SDL等下还要设置渲染配置，那个更复杂）

```c
// init SDL and set appropriate hints
static bool
sdl_init_and_configure(bool display, const char *render_driver) {
    uint32_t flags = display ? SDL_INIT_VIDEO : SDL_INIT_EVENTS;
    if (SDL_Init(flags)) {
        LOGC("Could not initialize SDL: %s", SDL_GetError());
        return false;
    }

    atexit(SDL_Quit);

    if (!display) {
        return true;
    }

    if (render_driver && !SDL_SetHint(SDL_HINT_RENDER_DRIVER, render_driver)) {
        LOGW("Could not set render driver");
    }

    // Linear filtering
    if (!SDL_SetHint(SDL_HINT_RENDER_SCALE_QUALITY, "1")) {
        LOGW("Could not enable linear filtering");
    }

#ifdef SCRCPY_SDL_HAS_HINT_MOUSE_FOCUS_CLICKTHROUGH
    // Handle a click to gain focus as any other click
    if (!SDL_SetHint(SDL_HINT_MOUSE_FOCUS_CLICKTHROUGH, "1")) {
        LOGW("Could not enable mouse focus clickthrough");
    }
#endif

#ifdef SCRCPY_SDL_HAS_HINT_VIDEO_X11_NET_WM_BYPASS_COMPOSITOR
    // Disable compositor bypassing on X11
    if (!SDL_SetHint(SDL_HINT_VIDEO_X11_NET_WM_BYPASS_COMPOSITOR, "0")) {
        LOGW("Could not disable X11 compositor bypass");
    }
#endif

    // Do not minimize on focus loss
    if (!SDL_SetHint(SDL_HINT_VIDEO_MINIMIZE_ON_FOCUS_LOSS, "0")) {
        LOGW("Could not disable minimize on focus loss");
    }

    // Do not disable the screensaver when scrcpy is running
    SDL_EnableScreenSaver();

    return true;
}
```

上面的代码块基本上没什么技术含量，告诉SDL的初始化模式（是要渲染画面还是只监听事件就好），然后设置一些渲染的Hint，就完事了。

### 4. 连接到server

在这之前，client已经以同步的方式创建了socket，启动了server，不过我们现在还没有调用过accept()。不过因为accpet()的作用是阻塞的顺序接受传入连接，所以无论是先accpet()再被server connect，还是先被server connect再accpet()，这都是没问题的。不需要做额外的同步。

进入`server_connect_to`方法，在adb reverse模式下，client连续的accpet两个socket：一个是视频流socket，一个是控制流socket。

>  server侧会以相同的顺序连接client的监听socket，并且反正使用的socket类型是TCP，server侧会随机选择端口去连接client，只要一个socket连接的四元组不同，就不会冲突。

两个socket都accept完成后，就已经不再需要client的welcome socket了，因此client直接把那个监听localhost:27183的socket给关掉。

```c
// we don't need the server socket anymore
if (!atomic_flag_test_and_set(&server->server_socket_closed)) {
    // close it from here
    close_socket(server->server_socket);
    // otherwise, it is closed by run_wait_server()
}
```

顺理成章地，adb reverse的转发能力也不再需要了，执行`disable_tunnel`方法把adb reverse关掉。

```c
disable_tunnel(server); // ignore failure
server->tunnel_enabled = false;
return true;
```

至此，如无意外，client侧就已经成功地建立起了与手机端的server的两条socket连接了，可以经由这两条socket连接，开始和server进行通信了。

#### 5. 读取配置信息

建立起socket连接，对于scrcpy的启动来说只是万里长征的第一步。此时我们还没有建立起client和server的视频流连接、控制流连接、文件传输流连接。在这些连接中，最重要的，用户最看重的，同时也是最麻烦的，就是视频流连接。但是在手机的屏幕画面信息（分辨率，屏幕旋转方向）没有发生变化的时候，视频流中是不会这些额外的描述当前屏幕画面信息的。因此，需要借助server去收集手机屏幕当前的画面信息。

```c
char device_name[DEVICE_NAME_FIELD_LENGTH];
    struct size frame_size;

    // screenrecord does not send frames when the screen content does not
    // change therefore, we transmit the screen size before the video stream,
    // to be able to init the window immediately
    if (!device_read_info(server.video_socket, device_name, &frame_size)) {
        goto end;
    }
```

```c
// Client side: read device screen info.

bool
device_read_info(socket_t device_socket, char *device_name, struct size *size) {
    unsigned char buf[DEVICE_NAME_FIELD_LENGTH + 4];
    int r = net_recv_all(device_socket, buf, sizeof(buf));
    if (r < DEVICE_NAME_FIELD_LENGTH + 4) {
        LOGE("Could not retrieve device information");
        return false;
    }
    // in case the client sends garbage
    buf[DEVICE_NAME_FIELD_LENGTH - 1] = '\0';
    // strcpy is safe here, since name contains at least
    // DEVICE_NAME_FIELD_LENGTH bytes and strlen(buf) < DEVICE_NAME_FIELD_LENGTH
    strcpy(device_name, (char *) buf);
    size->width = (buf[DEVICE_NAME_FIELD_LENGTH] << 8)
            | buf[DEVICE_NAME_FIELD_LENGTH + 1];
    size->height = (buf[DEVICE_NAME_FIELD_LENGTH + 2] << 8)
            | buf[DEVICE_NAME_FIELD_LENGTH + 3];
    return true;
}
```

```java
// Server side: send device screen info
DesktopConnection connection = new DesktopConnection(videoSocket, controlSocket);
Size videoSize = device.getScreenInfo().getVideoSize();
connection.send(Device.getDeviceName(), videoSize.getWidth(), videoSize.getHeight());
return connection;
```

阅读代码不难知道，屏幕尺寸信息存在了数据包的后四个字节，存储格式如下

```
....|width高8位|width低8位|height高8位|height低8位|
```

### 6. 杂项初始化

拿到了手机的屏幕分辨率信息后，就可以开始初始化一些与屏幕相关的逻辑了。需要进行的初始化逻辑主要有以下几点：

当scrcpy带有启动参数`display`时，会:

* 初始化FPS计数器
  * 给FPS计数器初始化一个mutex变量。
  * 给FPS计数器初始化一个状态变化的condition变量。
  * 将counter->started变量原子性地设为0。
* 初始化视频缓冲区
  * 初始化一个解码缓冲区 `av_frame_alloc()`。
  * 初始化一个渲染缓冲区。
  * 初始化一个视频缓冲区的mutex变量。
  * 如果要求render expired frames，那么创建一个渲染expired frame的consume condition变量。
* (If needed) 初始化文件传输控制器
  * 创建文件传输控制器的mutex变量。
  * 创建文件传输控制器的event condition变量
  * 如果带序列号运行，给对应的file_handler设置serial属性。
  * 初始化一些杂七杂八的标志位，阅读代码不难理解。
* 初始化解码器
  * 只有一行：把解码器的Video Buffer指向刚刚初始化好的视频缓冲区。

当scrcpy带有启动单数`record`时，会:

* 初始化屏幕录制器
  * 复制一份参数中的filename到recoder->filename中。
  * 初始化mutex和queue_cond
  * 初始化一些杂七杂八的标志位，阅读代码不难理解。

### 7. 手机画面推流初始化

首先执行一段赋值的代码，把准备好的`video_socket`，`decoder`,和`recoder`赋值到`stream`结构中：

```c
void stream_init(struct stream *stream, socket_t socket,
            struct decoder *decoder, struct recorder *recorder) {
    stream->socket = socket;
    stream->decoder = decoder,
    stream->recorder = recorder;
    stream->has_pending = false;
}
```

然后创建了一条新线程，来执行`run_stream`方法。跟进`run_stream`方法查看代码。代码比较多也比较长，读者可能要集中精力，认真理解：

```c
static int
run_stream(void *data) {
    struct stream *stream = data;

    AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);
    if (!codec) {
        LOGE("H.264 decoder not found");
        goto end;
    }

    stream->codec_ctx = avcodec_alloc_context3(codec);
    if (!stream->codec_ctx) {
        LOGC("Could not allocate codec context");
        goto end;
    }

    if (stream->decoder && !decoder_open(stream->decoder, codec)) {
        LOGE("Could not open decoder");
        goto finally_free_codec_ctx;
    }

    if (stream->recorder) {
        if (!recorder_open(stream->recorder, codec)) {
            LOGE("Could not open recorder");
            goto finally_close_decoder;
        }

        if (!recorder_start(stream->recorder)) {
            LOGE("Could not start recorder");
            goto finally_close_recorder;
        }
    }

    stream->parser = av_parser_init(AV_CODEC_ID_H264);
    if (!stream->parser) {
        LOGE("Could not initialize parser");
        goto finally_stop_and_join_recorder;
    }

    // We must only pass complete frames to av_parser_parse2()!
    // It's more complicated, but this allows to reduce the latency by 1 frame!
    stream->parser->flags |= PARSER_FLAG_COMPLETE_FRAMES;

    for (;;) {
        AVPacket packet;
        bool ok = stream_recv_packet(stream, &packet);
        if (!ok) {
            // end of stream
            break;
        }

        ok = stream_push_packet(stream, &packet);
        av_packet_unref(&packet);
        if (!ok) {
            // cannot process packet (error already logged)
            break;
        }
    }

    LOGD("End of frames");

    if (stream->has_pending) {
        av_packet_unref(&stream->pending);
    }

    av_parser_close(stream->parser);
finally_stop_and_join_recorder:
    if (stream->recorder) {
        recorder_stop(stream->recorder);
        LOGI("Finishing recording...");
        recorder_join(stream->recorder);
    }
finally_close_recorder:
    if (stream->recorder) {
        recorder_close(stream->recorder);
    }
finally_close_decoder:
    if (stream->decoder) {
        decoder_close(stream->decoder);
    }
finally_free_codec_ctx:
    avcodec_free_context(&stream->codec_ctx);
end:
    notify_stopped();
    return 0;
}
```

* 搜索、创建H264的编码器
* 为编码器分配编码上下文
* 将编码器绑定到解码器关联的结构上
  * 给解码器绑定与编码器相关的上下文
  * 用搜到的编码器（参数传入）初始化刚才创建的上下文
* 根据录像格式查找对应的复用器
* 创建AVFormatContext上下文
* 设置录好视频的元数据中的"comment"字段为"Recorded by scrcpy 版本号"
* 创建一个AVStream对象，这个对象看上去是用来把解码后的视频流重新编码的输出流。然后设置一些输出流的属性。
* 根据给定的filenam和标志位打开一个AVIOContext。这个上下文接下来将会被用于保存录制完成的视频。

完成这些操作后，会开始尝试启动recorder。启动的步骤是这样的：

* 创建了一条用于录制的线程，并且执行方法`run_recorder`。

在`run_recorder`内部，是一个大的事件循环。这个事件循环主要就是保证同步地从`recorder->queue`中取packet，对最后一帧及没有PTS信息的控制帧做处理，最后把数据包写到recorder的AVFormatContext中。完成一帧的录制。

**俗话说得好，巧妇难为无米之炊。**到目前为止，虽然scrcpy已经花了很大的精力去初始化一系列必要的模块，但是我们还没有真正的将编解码器与手机端推过来的视频流进行对接。接下来我们将要告诉解码器，网络中传输的数据包应该用哪种流类型的parser去解析。下面的这段代码就能很好的完成任务：

```c
stream->parser = av_parser_init(AV_CODEC_ID_H264);
// We must only pass complete frames to av_parser_parse2()!
// It's more complicated, but this allows to reduce the latency by 1 frame!
stream->parser->flags |= PARSER_FLAG_COMPLETE_FRAMES;
```

搞完了这些乱七八糟的东西之后，就真正的开始了从网络中接收视频流数据包、解析、最后推到解码器和录屏器的队列中，等待渲染和编码保存了。这个操作的代码并不复杂，我们把代码复制过来，一点一点的看：

```c
for (;;) {
        AVPacket packet;
        bool ok = stream_recv_packet(stream, &packet);
        if (!ok) {
            // end of stream
            break;
        }

        ok = stream_push_packet(stream, &packet);
        av_packet_unref(&packet);
        if (!ok) {
            // cannot process packet (error already logged)
            break;
        }
    }
```

先看`stream_recv_packet`方法：

因为注释说了，视频流的一个包是长这种样子的：

```c
    // The video stream contains raw packets, without time information. When we
    // record, we retrieve the timestamps separately, from a "meta" header
    // added by the server before each raw packet.
    //
    // The "meta" header length is 12 bytes:
    // [. . . . . . . .|. . . .]. . . . . . . . . . . . . . . ...
    //  <-------------> <-----> <-----------------------------...
    //        PTS        packet        raw packet
    //                    size
    //
    // It is followed by <packet_size> bytes containing the packet/frame.
```

PTS是8字节，packet size是4字节，总计就有12字节的包头，其中包头的定长字段len告知我们这个包一共有多长。这个方法干的事情也就是从视频流传输的socket中解析一个含有meta信息的视频包数据，存到传入的指针的data字段里。

从这个方法中，我们可以得知，当一个包没有pts信息时，pts字段十进制为-1（二进制就是64个1），否则，包的pts信息存在低63位，第64位为0。这也是代码

```c
assert(pts == NO_PTS || (pts & 0x8000000000000000) == 0);
```

的作用：为了避免接收到损坏的包。接下来的代码就是开辟存储包数据的空间（allocate），然后从网络接受meta中指定的数据长度，封装进packet，返回。

数据包准备好后，要把数据包推给视频流处理相关的队列（比如渲染队列和录屏队列）。接下来执行的就是完成上述工作的`stream_push_packet`方法。

首先函数处理数据包为配置包的情况，因为配置数据包不携带帧信息，因此不能立马把配置包送到队列中解析渲染，而是应该把配置包和别的有帧信息的包粘在一起，在帧解析的事件循环中再跟随者带有帧信息的包一起解析。所以函数一进来马上就开始处理这个配置包的粘包操作，把配置包和其他数据包(pending包)合并在一起。然后判断：如果是配置包，则主要影响的是recorder，会把配置包推给recorder的处理队列：

```c
if (is_config) {
        // config packet
        bool ok = process_config_packet(stream, packet);
        if (!ok) {
            return false;
        }
} 

static bool
process_config_packet(struct stream *stream, AVPacket *packet) {
    if (stream->recorder && !recorder_push(stream->recorder, packet)) {
        LOGE("Could not send config packet to recorder");
        return false;
    }
    return true;
}

bool
recorder_push(struct recorder *recorder, const AVPacket *packet) {
    mutex_lock(recorder->mutex);
    assert(!recorder->stopped);

    if (recorder->failed) {
        // reject any new packet (this will stop the stream)
        return false;
    }

    struct record_packet *rec = record_packet_new(packet);
    if (!rec) {
        LOGC("Could not allocate record packet");
        return false;
    }

    queue_push(&recorder->queue, next, rec); // 核心代码，待处理数据包入队
    cond_signal(recorder->queue_cond); // 唤醒录制事件循环的队列包阻塞等待的condition wait。具体见上文。

   // 主要是怕消费和生产抢队列中的数据包，出现race condition，所以录制的队列数据包操作要同步。这很自然，没啥毛病。
    mutex_unlock(recorder->mutex);
    return true;
}

```

否则，对刚才的视频流数据包进行解析，分别推给decoder和recorder(如有)：

```c
static bool
stream_parse(struct stream *stream, AVPacket *packet) {
    uint8_t *in_data = packet->data;
    int in_len = packet->size;
    uint8_t *out_data = NULL;
    int out_len = 0;
    int r = av_parser_parse2(stream->parser, stream->codec_ctx,
                             &out_data, &out_len, in_data, in_len,
                             AV_NOPTS_VALUE, AV_NOPTS_VALUE, -1);

    // PARSER_FLAG_COMPLETE_FRAMES is set
    assert(r == in_len);
    (void) r;
    assert(out_len == in_len);

    if (stream->parser->key_frame == 1) {
        packet->flags |= AV_PKT_FLAG_KEY;
    }

    bool ok = process_frame(stream, packet);
    if (!ok) {
        LOGE("Could not process frame");
        return false;
    }

    return true;
}
```

关键在`process_frame`方法中。

```c
static bool
process_frame(struct stream *stream, AVPacket *packet) {
    if (stream->decoder && !decoder_push(stream->decoder, packet)) {
        return false;
    }

    if (stream->recorder) {
        packet->dts = packet->pts;

        if (!recorder_push(stream->recorder, packet)) {
            LOGE("Could not send packet to recorder");
            return false;
        }
    }

    return true;
}
```

跟进`decoder_push`方法阅读代码：

```c
bool
decoder_push(struct decoder *decoder, const AVPacket *packet) {
// the new decoding/encoding API has been introduced by:
// <http://git.videolan.org/?p=ffmpeg.git;a=commitdiff;h=7fc329e2dd6226dfecaa4a1d7adf353bf2773726>
    int ret;
    if ((ret = avcodec_send_packet(decoder->codec_ctx, packet)) < 0) {
        LOGE("Could not send video packet: %d", ret);
        return false;
    }
    ret = avcodec_receive_frame(decoder->codec_ctx,
                                decoder->video_buffer->decoding_frame);
    if (!ret) {
        // a frame was received
        push_frame(decoder);
    } else if (ret != AVERROR(EAGAIN)) {
        LOGE("Could not receive video frame: %d", ret);
        return false;
    }
    return true;
  // 上面的代码省略了没有新解码API时的Fallback方法。如果读者有兴趣可以自行在scrcpy源码中查看#else宏定义的fallback代码。
}
```

这个`decoder_push`把数据包推给decoder, 等解码器对数据包进行解码后，从解码器的视频缓冲区中把帧拿出来，看看帧OK了没有。如果OK了，就会把帧数据给到decoder的video_buffer，然后通知SDL说有新的帧要绘制了。通知SDL的方法没什么意思，我们重点关注的是把帧提交到video_buffer的操作：

```c
void
video_buffer_offer_decoded_frame(struct video_buffer *vb,
                                 bool *previous_frame_skipped) {
    mutex_lock(vb->mutex);
    if (vb->render_expired_frames) {
        // wait for the current (expired) frame to be consumed
        while (!vb->rendering_frame_consumed && !vb->interrupted) {
            cond_wait(vb->rendering_frame_consumed_cond, vb->mutex);
        }
    } else if (!vb->rendering_frame_consumed) {
        fps_counter_add_skipped_frame(vb->fps_counter);
    }

    video_buffer_swap_frames(vb); // 标准的双缓冲！

    *previous_frame_skipped = !vb->rendering_frame_consumed;
    vb->rendering_frame_consumed = false;

    mutex_unlock(vb->mutex);
}
```

看上面代码段中的注释，结合上面拿到解码后的帧的代码

```c
avcodec_receive_frame(decoder->codec_ctx,
                                decoder->video_buffer->decoding_frame);
```

不难知道这双缓冲到底是怎么做的：先把解码完的帧保存在“所谓的”decoding_frame结构中，然后对rendering_frame和decoding_frame做交换，最后通知SDL绘制新的帧。这样就避免了相同的帧在decoding和rendering中复制；

```c
static void
video_buffer_swap_frames(struct video_buffer *vb) {
    AVFrame *tmp = vb->decoding_frame;
    vb->decoding_frame = vb->rendering_frame;
    vb->rendering_frame = tmp;
}
```

只要交换指针，就等于提交了新的帧去渲染，非常的make sense，节约资源。这就是典型的双缓冲，典型的Zero copy，CAS设计。

下面有个if代码可以提醒一下：

```c
if (stream->has_pending) {
		// the pending packet must be discarded (consumed or error)
		stream->has_pending = false;
		av_packet_unref(&stream->pending);
}
```

其中因为`stream->has_pending`是在处理config包的时候被set上去的，如果走到了下面的逻辑（不是config包）的时候，`stream->has_pending`还是true的话，其实pending的帧已经被处理过了，直接设回false就可以了。

最后就是简单的释放pending包资源。完成一帧的接收和渲染。

### 8. 控制器初始化

至此，有关视频流的接收初始化工作就全部完成了。我们可以松口气，开始看看并不这么麻烦的控制器初始化以及控制数据包的格式。 

```c
bool
controller_init(struct controller *controller, socket_t control_socket) {
    cbuf_init(&controller->queue);

    if (!receiver_init(&controller->receiver, control_socket)) {
        return false;
    }

    if (!(controller->mutex = SDL_CreateMutex())) {
        receiver_destroy(&controller->receiver);
        return false;
    }

    if (!(controller->msg_cond = SDL_CreateCond())) {
        receiver_destroy(&controller->receiver);
        SDL_DestroyMutex(controller->mutex);
        return false;
    }

    controller->control_socket = control_socket;
    controller->stopped = false;

    return true;
}
```

```c
#define cbuf_init(PCBUF) (void) ((PCBUF)->head = (PCBUF)->tail = 0)

(void) ((&controller->queue)->head = (&controller->queue)->tail = 0)
```

首先执行`controller_init`方法，此方法进入后先初始化了控制器的消息队列，然后执行`receiver_init`方法，这个方法只是给`receiver`创建一些mutex变量，方法返回后继续向下执行，为`controller`创建mutex和消息来临时的condition，比较常规，没有太多特别的。

控制器初始化完成后，开始创建控制器事件处理线程：

```c
bool
controller_start(struct controller *controller) {
    LOGD("Starting controller thread");

    controller->thread = SDL_CreateThread(run_controller, "controller",
                                          controller);
    if (!controller->thread) {
        LOGC("Could not start controller thread");
        return false;
    }

    if (!receiver_start(&controller->receiver)) {
        controller_stop(controller);
        SDL_WaitThread(controller->thread, NULL);
        return false;
    }

    return true;
}
```
```c
static int
run_controller(void *data) {
    struct controller *controller = data;

    for (;;) {
        mutex_lock(controller->mutex);
        while (!controller->stopped && cbuf_is_empty(&controller->queue)) {
            cond_wait(controller->msg_cond, controller->mutex);
        }
        if (controller->stopped) {
            // stop immediately, do not process further msgs
            mutex_unlock(controller->mutex);
            break;
        }
        struct control_msg msg;
        bool non_empty = cbuf_take(&controller->queue, &msg);
        assert(non_empty);
        (void) non_empty;
        mutex_unlock(controller->mutex);

        bool ok = process_msg(controller, &msg);
        control_msg_destroy(&msg);
        if (!ok) {
            LOGD("Could not write msg to socket");
            break;
        }
    }
    return 0;
}
```

这又是一个非常标准且枯燥的事件循环，没有太多可圈可点的地方。不过倒是可以花点时间看看控制消息的结构体定义：

```c
struct control_msg {
    enum control_msg_type type;
    union {
        struct {
            enum android_keyevent_action action;
            enum android_keycode keycode;
            enum android_metastate metastate;
        } inject_keycode;
        struct {
            char *text; // owned, to be freed by SDL_free()
        } inject_text;
        struct {
            enum android_motionevent_action action;
            enum android_motionevent_buttons buttons;
            uint64_t pointer_id;
            struct position position;
            float pressure;
        } inject_touch_event;
        struct {
            struct position position;
            int32_t hscroll;
            int32_t vscroll;
        } inject_scroll_event;
        struct {
            char *text; // owned, to be freed by SDL_free()
            bool paste;
        } set_clipboard;
        struct {
            enum screen_power_mode mode;
        } set_screen_power_mode;
    };
};
```

这个结构体也很清晰明了，接下来看看控制消息序列化的具体方法，看看控制消息结构是如何序列化成网络上传输的数据包的：

```c
size_t
control_msg_serialize(const struct control_msg *msg, unsigned char *buf) {
    buf[0] = msg->type;
    switch (msg->type) {
        case CONTROL_MSG_TYPE_INJECT_KEYCODE:
            buf[1] = msg->inject_keycode.action;
            buffer_write32be(&buf[2], msg->inject_keycode.keycode);
            buffer_write32be(&buf[6], msg->inject_keycode.metastate);
            return 10;
        case CONTROL_MSG_TYPE_INJECT_TEXT: {
            size_t len =
                write_string(msg->inject_text.text,
                             CONTROL_MSG_INJECT_TEXT_MAX_LENGTH, &buf[1]);
            return 1 + len;
        }
        case CONTROL_MSG_TYPE_INJECT_TOUCH_EVENT:
            buf[1] = msg->inject_touch_event.action;
            buffer_write64be(&buf[2], msg->inject_touch_event.pointer_id);
            write_position(&buf[10], &msg->inject_touch_event.position);
            uint16_t pressure =
                to_fixed_point_16(msg->inject_touch_event.pressure);
            buffer_write16be(&buf[22], pressure);
            buffer_write32be(&buf[24], msg->inject_touch_event.buttons);
            return 28;
        case CONTROL_MSG_TYPE_INJECT_SCROLL_EVENT:
            write_position(&buf[1], &msg->inject_scroll_event.position);
            buffer_write32be(&buf[13],
                             (uint32_t) msg->inject_scroll_event.hscroll);
            buffer_write32be(&buf[17],
                             (uint32_t) msg->inject_scroll_event.vscroll);
            return 21;
        case CONTROL_MSG_TYPE_SET_CLIPBOARD: {
            buf[1] = !!msg->set_clipboard.paste;
            size_t len = write_string(msg->set_clipboard.text,
                                      CONTROL_MSG_CLIPBOARD_TEXT_MAX_LENGTH,
                                      &buf[2]);
            return 2 + len;
        }
        case CONTROL_MSG_TYPE_SET_SCREEN_POWER_MODE:
            buf[1] = msg->set_screen_power_mode.mode;
            return 2;
        case CONTROL_MSG_TYPE_BACK_OR_SCREEN_ON:
        case CONTROL_MSG_TYPE_EXPAND_NOTIFICATION_PANEL:
        case CONTROL_MSG_TYPE_COLLAPSE_NOTIFICATION_PANEL:
        case CONTROL_MSG_TYPE_GET_CLIPBOARD:
        case CONTROL_MSG_TYPE_ROTATE_DEVICE:
            // no additional data
            return 1;
        default:
            LOGW("Unknown message type: %u", (unsigned) msg->type);
            return 0;
    }
}
```

控制消息最大长度为4096字节，定义在宏`CONTROL_MSG_SERIALIZED_MAX_SIZE`中。消息体的第一个字节存储消息类型。目前scrcpy client支持传送以下几种消息类型：

* 键盘输入事件（把client的键盘输入转发到手机server）
* 文本事件（inject the text as input events）
* 触摸事件（client窗口点击转发到手机的触摸事件）
* 滚动事件
* 剪贴板事件（store the text in the device clipboard and paste）
* 屏幕开关事件（点亮屏幕或关闭屏幕）
* 返回或者点亮屏幕事件
* 下拉任务栏
* 收起任务栏
* 获取设备的剪贴板（就是把server的剪贴板同步到本地）
* 旋转设备

除了文本消息和剪贴板消息可能是变长消息体外，其他的消息都是定长的。因此先重点关注一下变长消息体的结构，以`CONTROL_MSG_TYPE_INJECT_TEXT`为案例分析。点进`write_string`方法，首先进行的是`utf8_truncation_index`。因为inject text的文本最大长度限制是300，当传入待inject的文本超过指定长度时，需要进行截断。而文本是使用UTF8进行编码的，UTF8在表达多字节字符集的时候有前导比特位告知当前表达的字符需要占用多少个字节。

> UTF8单字节符（可用比特位以x来表示）:
>
> 0xxx xxxx
>
> 表示范围：0000 0000 -> 0111 1111
>
> UTF8 双字节符号
>
> 110x xxxx 10xx xxxx
>
> 表示范围: 1100 0010 1000 0000 -> 1101 1111 1011 1111
>
> UTF8 三字节符号
>
> 1110 xxxx 10xx xxxx 10xx xxxx
>
> UTF8 四字节符号
>
> 1111 0xxx 10xx xxxx 10xx xxxx 10xx xxxx
>
> 后面两个多字节符号的UTF-8表示范围自己推导。笔者实在是懒得推导了。

> 另外，笔者在阅读这一段代码的时候搞不明白，为什么最长消息体是4095 (Buffer长度是4096字节，刨去一个存消息类型的字节)的情况下，inject text的最大文本长度限制是300。猜想可能和安卓的某些剪贴板机制有关，待查。

Therefore, this method should verify whether the last byte of truncated buffer is a new UTF-8 codepoint mark byte (begin with 1100 0000) or not. If we cut the byte followed by the UTF-8 codepoint mark byte, the message cannot parse correctly. So the method should verify every bytes truncated end-to-front, ensure that no more broken UTF-8 codepoint in the end of sending buffer.

因此，这个方法要检查准备截断的消息体的最后一个字节是不是一个新的UTF-8多字节码点前导标记字节(以1100 0000开头)。因为如果截断的是UTF-8多字节码点前导标记以后的字节，那么消息体将不能被正确的解析。所以，方法必须从后往前检查发送缓冲中截断后的字节，把截断后可能残余的UTF-8码点字节都一并截掉。

> 限于笔者低下的表达能力，笔者觉得这段描述UTF-8编码格式的文字用中文写实在是费劲，因此写了一段英文的版本，如果读者在阅读中文版本的时候感觉到困惑，不妨阅读英文版本以及有关UTF-8 encoding format的文档 [UTF-8 编码格式 from wikipedia](https://en.wikipedia.org/wiki/UTF-8#Description)。

完成消息体字节的截断后，方法返回截断完成后发送字节的长度。接下来，把截断后的文本复制到消息体中，对消息体完成序列化，准备把消息体发送出去。从宏观上看，实现如下代码：

```c
static bool
process_msg(struct controller *controller,
              const struct control_msg *msg) {
    unsigned char serialized_msg[CONTROL_MSG_SERIALIZED_MAX_SIZE];
    int length = control_msg_serialize(msg, serialized_msg);
    if (!length) {
        return false;
    }
    int w = net_send_all(controller->control_socket, serialized_msg, length);
    return w == length;
}
```

### 9. 让SDL开始渲染画面

接下来，scrcpy会将一大堆参数传给`screen_init_rendering`方法，开始创建窗口，及绘制手机画面。对于scrcpy而言，上面介绍的控制器、剪贴板、UTF-8编码，从严格意义上来说只能算是锦上添花；而高效地解析视频流，并把画面显示在电脑屏幕上，才是scrcpy这个软件小而美的精髓。我们将开始关注画面的渲染是如何初始化的：

```c
if (!screen_init_rendering(&screen, window_title, frame_size,
                                   options->always_on_top, options->window_x,
                                   options->window_y, options->window_width,
                                   options->window_height,
                                   options->window_borderless,
                                   options->rotation, options-> mipmaps)) {
            goto end;
}
```

在开始阅读`screen_init_recording`方法之前，先了解几个简单的概念:

* frame_size：帧大小。代表手机端屏幕视频流所传输的画面的宽高像素大小。
* rotation：是否旋转。默认是竖屏（1080x1920），旋转过来就是横屏（1920x1080）。
* window_size：窗体大小，受到`content_size`和`window_width`、`window_height`相关。
* content_size：内容大小，通过frame_size和rotation状态计算出来的，未经裁剪的显示画面大小。

方法首先看看当前画面是不是要旋转的，如果要旋转的话就把原本的宽高调整一下位置；然后根据content_size和window_size(width and height)和initial window size组合起来考虑，计算出要创建的窗口的window_size，接下来就是设置一堆flags，不断的调用各种SDL的方法，创建窗口、创建渲染器、初始化OpenGL，创建屏幕绘图材质，更新画面，开始持续输出视频流。

计算出成比例的长宽

```c
// initially, there is no current size, so use the frame size as current size
// req_width and req_height, if not 0, are the sizes requested by the user
static inline struct size
get_initial_optimal_size(struct size content_size, uint16_t req_width,
                         uint16_t req_height) {
    struct size window_size;
    if (!req_width && !req_height) {
        window_size = get_optimal_size(content_size, content_size);
    } else {
        if (req_width) {
            window_size.width = req_width;
        } else {
            // compute from the requested height
            window_size.width = (uint32_t) req_height * content_size.width
                              / content_size.height;
        }
        if (req_height) {
            window_size.height = req_height;
        } else {
            // compute from the requested width
            window_size.height = (uint32_t) req_width * content_size.height
                               / content_size.width;
        }
    }
    return window_size;
}
```

```c
// return the optimal size of the window, with the following constraints:
//  - it attempts to keep at least one dimension of the current_size (i.e. it
//    crops the black borders)
//  - it keeps the aspect ratio
//  - it scales down to make it fit in the display_size
// 中文翻译版 Chinese Version
// 返回窗口的一个优化后的大小，受以下几个条件的约束：
// - 尝试保持current_size中至少一条边的长度
// - 保持画面比例
// - 画面经过伸缩以适应display_size的大小（a.k.a viewport size）
static struct size
get_optimal_size(struct size current_size, struct size content_size) {
    if (content_size.width == 0 || content_size.height == 0) {
        // avoid division by 0
        return current_size;
    }

    struct size window_size;

    struct size display_size;
    if (!get_preferred_display_bounds(&display_size)) {
        // could not get display bounds, do not constraint the size
        window_size.width = current_size.width;
        window_size.height = current_size.height;
    } else {
        window_size.width = MIN(current_size.width, display_size.width);
        window_size.height = MIN(current_size.height, display_size.height);
    }

    if (is_optimal_size(window_size, content_size)) {
        return window_size;
    }

    bool keep_width = content_size.width * window_size.height
                    > content_size.height * window_size.width;
    if (keep_width) {
        // remove black borders on top and bottom
        window_size.height = content_size.height * window_size.width
                           / content_size.width;
    } else {
        // remove black borders on left and right (or none at all if it already
        // fits)
        window_size.width = content_size.width * window_size.height
                          / content_size.height;
    }

    return window_size;
}
```

代码首先去获取现在屏幕有多大的可用空间（对于macOS来说，可用空间就是除掉上面的菜单栏和底下的dock的一整行之后剩余的显示空间），然后用屏幕最大可用空前去裁剪请求的current_size。因为不能大过屏幕的可用空间，否则内容就会显示不全了。然后计算被可用空间裁剪过的size是不是和content_size保持纵横比的，如果是就可以直接返回了，否则要重新计算纵横比再返回。详细逻辑看上面代码块的后半段，不难理解。

从计算窗体大小的`get_initial_optimal_size`方法返回后，scrcpy会开始根据一些meson在configure阶段收集的系统信息，例如你现在的系统是不是开启了HiDPIng模式，你是否设置了always on top的标记等等，为待创建的SDL window设置一系列的标志位。然后开始创建窗口：

```c
screen->window = SDL_CreateWindow(window_title, x, y,
                                      window_size.width, window_size.height,
                                      window_flags);
```

以及创建2D窗体内容渲染上下文

```c
screen->renderer = SDL_CreateRenderer(screen->window, -1,
                                          SDL_RENDERER_ACCELERATED);
```

渲染上下文创建完成后，需要获取SDL采用的渲染器，对于一些比较普遍的渲染器（如OpenGL），需要进行特别的设置。代码读到这里，读者们也不难发现，我们在控制台中经常看到有关当前操作系统采用的渲染器的日志

```shell
2020-07-30 08:54:07.015 scrcpy[77680:2429283] INFO: scrcpy 1.14 <https://github.com/Genymobile/scrcpy>
/usr/local/Cellar/scrcpy/1.14_1/share/scrcpy/scrcpy-server: 1 file pushed, 0 skipped. 9.0 MB/s (33142 bytes in 0.004s)
[server] INFO: Device: Xiaomi MIX 2S (Android 10)
2020-07-30 08:54:08.091 scrcpy[77680:2429283] INFO: Created renderer: metal
2020-07-30 08:54:08.091 scrcpy[77680:2429283] INFO: Renderer: metal <- 看这个
2020-07-30 08:54:08.093 scrcpy[77680:2429283] INFO: Initial texture: 1080x2160
```

其实就是这一句

```c
LOGI("Renderer: %s", renderer_name ? renderer_name : "(unknown)");
```

输出的。

接下来是解析xpm格式的图标，scrcpy的团队发现使用SDL库的`SDL2_image`方法读取图片在MSYS2编译的Windows环境下有问题，因此他们团队不得已造了个轮子，不依赖SDL的方法来解析图像了。由于笔者对xpm格式的定义实在是没什么兴趣，因此解析icon文件的代码就略过了，有兴趣的读者可以自行深入学度`read_xpm()`方法，相信一定会有收获。

随后是创建有关于当前屏幕的2D渲染材质。又因为安卓屏幕录制视频流的编码是YV12，所以这里创建的2D渲染材质设置了像素格式为`SDL_PIXELFORMAT_YV12`，数据访问来源是`SDL_TEXTUREACCESS_STREAMING`（在网络流中获得的材质像素数据）。

### 10. 控制器发出预设指令（如有）

如果scrcpy的启动参数中需要把屏幕关掉，那么当代码执行到这一步后，会构造一个类型为`CONTROL_MSG_TYPE_SET_SCREEN_POWER_MODE`，结果是让手机关闭屏幕的消息给server。方法调用后其实是会原子性地给controller的事件队列插入一个新的，关闭屏幕的消息，等待下一个事件循环后controller就会把这个消息序列化发出去给server，server处理后通知系统关闭屏幕。

### 11. 进入全屏模式（如果需要）

如果scrcpy的启动参数中需要把scrcpy设全屏，那么代码执行到这一步之后，就会进行最大化的操作。看了源码发现设为全屏后需要立即调用`screen_render`刷新屏幕方向变化后的画面。具体细节可以看源码，这部分代码比较简单。

### 12. 进入事件循环

主线程中进行的事件循环主要有两个作用：1. 对窗口的最大最小化渲染做处理：

```c

#ifdef CONTINUOUS_RESIZING_WORKAROUND
// On Windows and MacOS, resizing blocks the event loop, so resizing events are
// not triggered. As a workaround, handle them in an event handler.
//
// <https://bugzilla.libsdl.org/show_bug.cgi?id=2077>
// <https://stackoverflow.com/a/40693139/1987178>
static int
event_watcher(void *data, SDL_Event *event) {
    (void) data;
    if (event->type == SDL_WINDOWEVENT
            && event->window.event == SDL_WINDOWEVENT_RESIZED) {
        // In practice, it seems to always be called from the same thread in
        // that specific case. Anyway, it's just a workaround.
        screen_render(&screen, true);
    }
    return 0;
}
#endif
```

另外就是处理真正的用户操作scrcpy窗口而发生的事件：

```c
static enum event_result
handle_event(SDL_Event *event, bool control) {
    switch (event->type) {
        case EVENT_STREAM_STOPPED:
            LOGD("Video stream stopped");
            return EVENT_RESULT_STOPPED_BY_EOS;
        case SDL_QUIT:
            LOGD("User requested to quit");
            return EVENT_RESULT_STOPPED_BY_USER;
        case EVENT_NEW_FRAME:
            if (!screen.has_frame) {
                screen.has_frame = true;
                // this is the very first frame, show the window
                screen_show_window(&screen);
            }
            if (!screen_update_frame(&screen, &video_buffer)) {
                return EVENT_RESULT_CONTINUE;
            }
            break;
        case SDL_WINDOWEVENT:
            screen_handle_window_event(&screen, &event->window);
            break;
        case SDL_TEXTINPUT:
            if (!control) {
                break;
            }
            input_manager_process_text_input(&input_manager, &event->text);
            break;
        case SDL_KEYDOWN:
        case SDL_KEYUP:
            // some key events do not interact with the device, so process the
            // event even if control is disabled
            input_manager_process_key(&input_manager, &event->key, control);
            break;
        case SDL_MOUSEMOTION:
            if (!control) {
                break;
            }
            input_manager_process_mouse_motion(&input_manager, &event->motion);
            break;
        case SDL_MOUSEWHEEL:
            if (!control) {
                break;
            }
            input_manager_process_mouse_wheel(&input_manager, &event->wheel);
            break;
        case SDL_MOUSEBUTTONDOWN:
        case SDL_MOUSEBUTTONUP:
            // some mouse events do not interact with the device, so process
            // the event even if control is disabled
            input_manager_process_mouse_button(&input_manager, &event->button,
                                               control);
            break;
        case SDL_FINGERMOTION:
        case SDL_FINGERDOWN:
        case SDL_FINGERUP:
            input_manager_process_touch(&input_manager, &event->tfinger);
            break;
        case SDL_DROPFILE: {
            if (!control) {
                break;
            }
            file_handler_action_t action;
            if (is_apk(event->drop.file)) {
                action = ACTION_INSTALL_APK;
            } else {
                action = ACTION_PUSH_FILE;
            }
            file_handler_request(&file_handler, action, event->drop.file);
            break;
        }
    }
    return EVENT_RESULT_CONTINUE;
}

```

### 13. 应用退出前收尾

最后应用退出前的收尾其实没啥可圈可点的地方。先关注应用的主线程如何从事件循环中退出：

```c
static bool
event_loop(bool display, bool control) {
    (void) display;
#ifdef CONTINUOUS_RESIZING_WORKAROUND
    if (display) {
        SDL_AddEventWatch(event_watcher, NULL);
    }
#endif
    SDL_Event event;
    while (SDL_WaitEvent(&event)) {
        enum event_result result = handle_event(&event, control);
        switch (result) {
            case EVENT_RESULT_STOPPED_BY_USER:
                return true;
            case EVENT_RESULT_STOPPED_BY_EOS:
                LOGW("Device disconnected");
                return false;
            case EVENT_RESULT_CONTINUE:
                break;
        }
    }
    return false;
}
```

当`SDL_WaitEvent`为false，或者是`handle_event`方法返回了`EVENT_RESULT_STOPPED_BY_USER`，`EVENT_RESULT_STOPPED_BY_USER`的话，就会从事件循环中退出，走到最后的资源销毁逻辑。就是下面这一段和创建的时候对等的逻辑。

```c
    screen_destroy(&screen);

end:
    // stop stream and controller so that they don't continue once their socket
    // is shutdown
    if (stream_started) {
        stream_stop(&stream);
    }
    if (controller_started) {
        controller_stop(&controller);
    }
    if (file_handler_initialized) {
        file_handler_stop(&file_handler);
    }
    if (fps_counter_initialized) {
        fps_counter_interrupt(&fps_counter);
    }

    // shutdown the sockets and kill the server
    server_stop(&server);

    // now that the sockets are shutdown, the stream and controller are
    // interrupted, we can join them
    if (stream_started) {
        stream_join(&stream);
    }
    if (controller_started) {
        controller_join(&controller);
    }
    if (controller_initialized) {
        controller_destroy(&controller);
    }

    if (recorder_initialized) {
        recorder_destroy(&recorder);
    }

    if (file_handler_initialized) {
        file_handler_join(&file_handler);
        file_handler_destroy(&file_handler);
    }

    if (video_buffer_initialized) {
        video_buffer_destroy(&video_buffer);
    }

    if (fps_counter_initialized) {
        fps_counter_join(&fps_counter);
        fps_counter_destroy(&fps_counter);
    }

    server_destroy(&server);

    return ret;
}
```

这段代码中，读者点进去看的话其实大同小异，基本上都是等待mutex和condition完成，然后回收资源，结束任务。过程是相对的，只要前面创建的代码理解了，后面退出的代码也不难看懂。目前重点关注的是如何结束网络socket连接，优雅退出，可以查看以下令server退出的代码：

```c
void
server_stop(struct server *server) {
    if (server->server_socket != INVALID_SOCKET
            && !atomic_flag_test_and_set(&server->server_socket_closed)) {
        close_socket(server->server_socket);
    }
    if (server->video_socket != INVALID_SOCKET) {
        close_socket(server->video_socket);
    }
    if (server->control_socket != INVALID_SOCKET) {
        close_socket(server->control_socket);
    }

    assert(server->process != PROCESS_NONE);

    cmd_terminate(server->process);

    if (server->tunnel_enabled) {
        // ignore failure
        disable_tunnel(server);
    }

    SDL_WaitThread(server->wait_server_thread, NULL);
}
```

把没杀掉的welcome socket杀掉，然后是video socket和control socket。最后把维持着server启动的adb shell命令也给kill掉，adb reverse(或者是adb forward) 隧道关掉。最后等待那个守护welcome socket的进程退出，`server_stop`方法的任务也宣告结束了。本质上这就是一个网络应用和实时编解码应用。因此搞定网络连接的优雅断开以及编解码资源、绘图资源的优雅回收是显得非常重要的。video buffer（包含decoding buffer和rendering buffer）就是这样清除的

```c
void
video_buffer_destroy(struct video_buffer *vb) {
    if (vb->render_expired_frames) {
        SDL_DestroyCond(vb->rendering_frame_consumed_cond);
    }
    SDL_DestroyMutex(vb->mutex);
    av_frame_free(&vb->rendering_frame);
    av_frame_free(&vb->decoding_frame);
}
```

## 写在最后

感谢所有坚持阅读到这里的读者，我们的源码阅读之旅到目前为止就接近尾声了。相信通过阅读以上的源码，大家也对scrcpy的client侧工作原理有了一个初步的认识。虽然，在很多人的印象中，scrcpy本质上只是一个把手机屏幕显示到电脑上，方便我们在电脑上操作安卓手机的小程序。咋一听上去并不是很难，并且也肯定有不少安卓同行们脑海里就已经勾勒出这类工具软件的项目文件结构、采用的技术点等轮廓了。

但是为了做到尽善尽美，做到操作低延迟，画面高质量，方便易用（鼠标点击模拟操作，拖拽传文件，大量的启动参数可供配置，C语言程序也做到跨平台），我们可以从源码中看到开发软件的团队做出了非常多的努力，代码文件组织得井井有条，平台相关代码的细节处理也非常到位，甚至在描述视频流的数据包时，还在注释中画了一个数据包结构的字符画；这是一个开源软件十分难能可贵的品质。

在写下这篇笔记之前，笔者其实已经使用scrcpy很长一段时间了。因为笔者业余会进行一些安卓小程序的开发，每次为了调试功能，都需要把手从鼠标键盘上移开，拿起桌面上的手机一通点点点，然后又要按起键盘鼠标继续操作，笔者个人感觉这样的调试方式对于工作流的割裂感非常严重；并且笔者听一些在字节跳动工作的朋友说，字节的安卓开发调试机是不需要你去拿到真机的，反而是登陆内部的一个测试机管理平台，提交申请后就能够在类似于网页的地方，以scrcpy的形式操作放在远程的测试机。这样一来，公司既省去了发放、保管测试机的麻烦，也可以避免测试机给到员工手里之后产生的耗损，甚至服务器端统一管理这些测试机，管理平台也能随时随地监控测试机上运行的应用状态，偶尔也会给安卓开发的同学一些性能优化的建议。笔者一度十分好奇字节跳动究竟是怎么做好这一套测试机管理申请发放系统的；而目前来说笔者能接触到的、和字节测试机管理平台最像的软件就只是scrcpy了。因此，笔者强烈的好奇心也是这篇笔记能够完成的一个动力吧。

再次感谢读者们的阅读，希望这篇并不怎么规范的笔记能给您带来帮助，也希望能够给安卓开发同行们带来灵感。

