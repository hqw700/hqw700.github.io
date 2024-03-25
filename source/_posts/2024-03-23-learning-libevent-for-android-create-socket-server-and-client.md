---
title: Learning libevent for Android
date: 2024-03-23 18:40:12
tag: [Study]
category: [Android系统]
excerpt: Creating Socket Server and Client
---

Android默认包含libevent库，代码路径在$AOSP/external/libevent下，生成库在/system/lib{64}/libevent.so下， 以下是在Android Native下使用libevent的简单示例。

## 一，使用libevent库创建socket服务端
\$AOSP/frameworks/native/cmds/study/libevent_socket_server.cpp
``` cpp
#include <event2/event.h>
#include <event2/listener.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <iostream>
#include <cutils/sockets.h>

void echo_read_cb(struct bufferevent *bev, void *ctx) {
    struct evbuffer *input = bufferevent_get_input(bev);
    struct evbuffer *output = bufferevent_get_output(bev);
    size_t len = evbuffer_get_length(input);
    void *data = malloc(len);
    evbuffer_remove(input, data, len);
    evbuffer_add(output, data, len);
    std::cout << "RECE & SEND: " << (char *) data << std::endl;
    free(data);
}

void echo_event_cb(struct bufferevent *bev, short events, void *ctx) {
    if (events & BEV_EVENT_ERROR) {
        perror("Error from bufferevent");
    }
    if (events & (BEV_EVENT_EOF | BEV_EVENT_ERROR)) {
        bufferevent_free(bev);
    }
}


void accept_cb(evconnlistener *listener, evutil_socket_t fd, sockaddr *address, int socklen, void *ctx) {
    std::cout << "Accepted connection" << std::endl;
    event_base *base = evconnlistener_get_base(listener);
    bufferevent *bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
    bufferevent_setcb(bev, echo_read_cb, NULL, echo_event_cb, NULL);
    bufferevent_enable(bev, EV_READ | EV_WRITE);
}

int main() {
    event_base *base = event_base_new();
    if (!base) {
        std::cerr << "Error creating event base" << std::endl;
        return -1;
    }

    sockaddr_in sin;
    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(INADDR_ANY);
    sin.sin_port = htons(8080);
    evconnlistener *listener = evconnlistener_new_bind(base, accept_cb, NULL, LEV_OPT_CLOSE_ON_FREE | LEV_OPT_REUSEABLE,
                                                       -1, (sockaddr *) &sin, sizeof(sin));
    if (!listener) {
        std::cerr << "Could not create listener" << std::endl;
        return -1;
    }

    event_base_dispatch(base);

    evconnlistener_free(listener);
    event_base_free(base);

    return 0;
}
```


## 使用libevent库创建socket客户端
\$AOSP/frameworks/native/cmds/study/libevent_socket_client.cpp
``` cpp
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <iostream>
#include <cutils/sockets.h>
#include <arpa/inet.h>
#include <event2/buffer.h>

void read_cb(struct bufferevent *bev, void *ctx) {
    struct evbuffer *input = bufferevent_get_input(bev);
    size_t len = evbuffer_get_length(input);
    char *data = (char *) malloc(len + 1);
    evbuffer_remove(input, data, len);
    data[len] = '\0';
    printf("Received: %s\n", data);
    free(data);
}

void event_cb(struct bufferevent *bev, short events, void *ctx) {
    if (events & BEV_EVENT_ERROR) {
        perror("Error from bufferevent");
    }
    if (events & (BEV_EVENT_EOF | BEV_EVENT_ERROR)) {
        bufferevent_free(bev);
    }
}

void stdin_read_cb(evutil_socket_t fd, short events, void *arg) {
    struct bufferevent *bev = (struct bufferevent *) arg;
    char msg[1024];
    if (fgets(msg, sizeof(msg), stdin)) {
        struct evbuffer *output = bufferevent_get_output(bev);
        evbuffer_add(output, msg, strlen(msg));
    } else {
        // EOF or error
        event_base_loopbreak(bufferevent_get_base(bev));
    }
}


int main() {
    event_base *base = event_base_new();
    if (!base) {
        std::cerr << "Error creating event base" << std::endl;
        return -1;
    }

    sockaddr_in sin;
    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = inet_addr("127.0.0.1");
    sin.sin_port = htons(8080);

    bufferevent *bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);
    bufferevent_setcb(bev, read_cb, NULL, event_cb, NULL);
    bufferevent_enable(bev, EV_READ | EV_WRITE);
    if (bufferevent_socket_connect(bev, (sockaddr *) &sin, sizeof(sin)) < 0) {
        std::cerr << "Error connecting to server" << std::endl;
        return -1;
    }
    struct event *ev_stdin = event_new(base, fileno(stdin), EV_READ | EV_PERSIST, stdin_read_cb, bev);
    event_add(ev_stdin, NULL);

    event_base_dispatch(base);

    bufferevent_free(bev);
    event_base_free(base);

    return 0;
}
```

## Android.bp
\$AOSP/frameworks/native/cmds/study/libevent_socket_client.cpp 
``` java
cc_defaults {
    name: "study_defaults",

    cflags: [
        "-Wall",
        "-Wextra",
        "-Werror",
        "-Wno-unused-parameter",
        "-Wno-unused-function",
    ],
    static_libs: [
        "libbase",
        "libcutils",
        "libevent",
        "liblog",
    ],
}

cc_binary {
    name: "libevent_socket_server",
    defaults: ["study_defaults"],
    srcs: [
        "libevent_socket_server.cpp",
    ],
}

cc_binary {
    name: "libevent_socket_client",
    defaults: ["study_defaults"],
    srcs: [
        "libevent_socket_client.cpp",
    ],
}
```

## 三，测试
### 3.1 编译
``` shell
$ make libevent_socket_server && make libevent_socket_client
$ adb push $OUT/system/bin/libevent_socket* /data/local/tmp/
```

### 3.2 run
![](/img/blog/libevent_test.png)


## 四，libevent在android系统中的使用
在[tombstoned](https://cs.android.com/android/platform/superproject/main/+/main:system/core/debuggerd/tombstoned/tombstoned.cpp)程序中，采用libevent来处理socket连接


> 操作系统：Windows 11 
> 编译机：WSL2 Ubuntu-18.04  
> 手机：Google Pixel 3 Android 10
> AOSP: android-10.0.0_r41