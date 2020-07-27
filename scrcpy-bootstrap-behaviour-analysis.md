## scrcpy 启动流程分析

### 写在前面

scrcpy 是一款可以把安卓手机的屏幕投到电脑上，并支持在电脑端对手机进行操作（包括但不限于模拟触控，拖拽传文件，安装Apk，录屏等）的跨平台工具软件。相比于市面上的其他软件，scrcpy做到了轻量，便携，且不需要显式地在手机端安装任何辅助软件。

笔者使用这款软件已经有一段时间了，对这款软件的实现比较感兴趣，但苦于一直没有时间对这款软件的具体实现进行剖析总结，一直感到有点遗憾。趁着最近手头上的工作闲了下来，特意花了点时间下载了scrcpy的源码进行阅读。并总结了一些有关于scrcpy实现的一些心得体会，不敢独享，故写下了这篇文章。

**注意，在读源码之前，建议先按照scrcpy的文档配置好编译环境并且自己动手编译一次scrcpy的client和server。这样有助于配置IDE的代码高亮功能。**

**注意，本文的源码阅读环境为macOS 10.15，并不会去分析不属于本平台的代码段。**

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



