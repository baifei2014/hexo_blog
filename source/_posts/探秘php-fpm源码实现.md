---
layout: blog
title: 探秘php-fpm源码实现(一)
date: 2020-04-22 21:06:50
tags: [php, php-fpm, socket, 多进程, IO, 服务器编程]
---

## 前言

php-fpm是php内置的一个fastcgi进程管理器，由c语言实现，主要用于管理进程。在dnmp应用里，通常用于接受nginx转发来的请求，响应请求处理结果。它可以支持动态/静态子进程伸缩，对高负载应用还是很有用的。


## 简介

这篇文章主要想介绍下php启动初始化的流程，还有处理请求的流程，主要涉及到多进程编程，socket编程，还有IO复用函数等信息。由于篇幅限制，所以没有面面俱到，关于代码分析仅限于主要功能流程的介绍。


## fpm资源初始化

fpm资源主要包括配置资源读取还有信号，子进程，socket等。代码如下：

```c
int fpm_init(int argc, char **argv, char *config, char *prefix, char *pid, int test_conf, int run_as_root, int force_daemon, int force_stderr)
{
	fpm_globals.argc = argc;
	fpm_globals.argv = argv;
	if (config && *config) {
		fpm_globals.config = strdup(config);
	}
	fpm_globals.prefix = prefix;
	fpm_globals.pid = pid;
	fpm_globals.run_as_root = run_as_root;
	fpm_globals.force_stderr = force_stderr;

	if (0 > fpm_php_init_main()           ||
	    0 > fpm_stdio_init_main()         ||
	    0 > fpm_conf_init_main(test_conf, force_daemon) ||
	    0 > fpm_unix_init_main()          ||
	    0 > fpm_scoreboard_init_main()    ||
	    0 > fpm_pctl_init_main()          ||
	    0 > fpm_env_init_main()           ||
	    0 > fpm_signals_init_main()       ||
	    0 > fpm_children_init_main()      ||
	    0 > fpm_sockets_init_main()       ||
	    0 > fpm_worker_pool_init_main()   ||
	    0 > fpm_event_init_main()) {

		if (fpm_globals.test_successful) {
			exit(FPM_EXIT_OK);
		} else {
			zlog(ZLOG_ERROR, "FPM initialization failed");
			return -1;
		}
	}

	if (0 > fpm_conf_write_pid()) {
		zlog(ZLOG_ERROR, "FPM initialization failed");
		return -1;
	}

	fpm_stdio_init_final();
	zlog(ZLOG_NOTICE, "fpm is running, pid %d", (int) fpm_globals.parent_pid);

	return 0;
}
```

从代码可以看出，初始化包括设置一些全局配置，还有读取配置文件信息。输入输出重定向，信号，子进程，事件，进程池等的初始化。具体我们可以看下输入输出重定向，代码如下：

```c
int fpm_stdio_init_main() /* {{{ */
{
	int fd = open("/dev/null", O_RDWR);

	if (0 > fd) {
		zlog(ZLOG_SYSERROR, "failed to init stdio: open(\"/dev/null\")");
		return -1;
	}

	if (0 > dup2(fd, STDIN_FILENO) || 0 > dup2(fd, STDOUT_FILENO)) {
		zlog(ZLOG_SYSERROR, "failed to init stdio: dup2()");
		close(fd);
		return -1;
	}
	close(fd);
	return 0;
}
```

cgi编程的原理就是将服务器输出到标准输出的内容发送到对应socket文件描述符中，所以这里master进程输入输出重定向做的首先就是将标准输入文件描述符(STDIN_FILENO)和标准输出文件描述符(STDOUT_FILENO)重定向到/dev/null，也就是输出到空设备文件。在后面的worker子进程中，设置新的文件描述符也就是socket连接。这样就完成了由输入输出由系统标准输入输出到socket连接的转换。

在fpm初始化中，还有socket连接初始化，master进程会创建socket文件描述符，并绑定地址端口执行监听，但是master进程并不处理socket连接，创建的socket文件描述符会复制给子进程，由子进程去处理socket连接。

## fpm运行

fpm运行主要内容是根据worker进程池配置，初始化创建子进程。首先我们看下worker进程池数据结构：

```c
struct fpm_worker_pool_s {
	struct fpm_worker_pool_s *next;
	struct fpm_worker_pool_config_s *config;
	char *user, *home;									/* for setting env USER and HOME */
	enum fpm_address_domain listen_address_domain;
	int listening_socket;
	int set_uid, set_gid;								/* config uid and gid */
	int socket_uid, socket_gid, socket_mode;

	/* runtime */
	struct fpm_child_s *children;
	int running_children;
	int idle_spawn_rate;
	int warn_max_children;
#if 0
	int warn_lq;
#endif
	struct fpm_scoreboard_s *scoreboard;
	int log_fd;
	char **limit_extensions;

	/* for ondemand PM */
	struct fpm_event_s *ondemand_event;
	int socket_event_set;

#ifdef HAVE_FPM_ACL
	void *socket_acl;
#endif
};
```

有代码可以看出，进程池是一个链表结构，有后向指针，能够配置多个进程池，支持不同的配置。在进程池中，含有当前池运行的子进程数，还有空闲进程等信息，包括监听的对应文件描述符。

fpm运行信息在php-fun函数里面，具体我们看代码：

```c
int fpm_run(int *max_requests) /* {{{ */
{
	struct fpm_worker_pool_s *wp;

	/* create initial children in all pools */
	for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
		int is_parent;

		is_parent = fpm_children_create_initial(wp);

		if (!is_parent) {
			goto run_child;
		}

		/* handle error */
		if (is_parent == 2) {
			fpm_pctl(FPM_PCTL_STATE_TERMINATING, FPM_PCTL_ACTION_SET);
			fpm_event_loop(1);
		}
	}

	/* run event loop forever */
	fpm_event_loop(0);

run_child: /* only workers reach this point */

	fpm_cleanups_run(FPM_CLEANUP_CHILD);

	*max_requests = fpm_globals.max_requests;
	return fpm_globals.listening_socket;
}
```

master进程会遍历链表，调用fpm_children_create_initial()函数初始化创建子进程，具体我们看下函数信息：

```c
int fpm_children_create_initial(struct fpm_worker_pool_s *wp) /* {{{ */
{
	if (wp->config->pm == PM_STYLE_ONDEMAND) {
		wp->ondemand_event = (struct fpm_event_s *)malloc(sizeof(struct fpm_event_s));

		if (!wp->ondemand_event) {
			zlog(ZLOG_ERROR, "[pool %s] unable to malloc the ondemand socket event", wp->config->name);
			// FIXME handle crash
			return 1;
		}

		memset(wp->ondemand_event, 0, sizeof(struct fpm_event_s));
		fpm_event_set(wp->ondemand_event, wp->listening_socket, FPM_EV_READ | FPM_EV_EDGE, fpm_pctl_on_socket_accept, wp);
		wp->socket_event_set = 1;
		fpm_event_add(wp->ondemand_event, 0);

		return 1;
	}
	return fpm_children_make(wp, 0 /* not in event loop yet */, 0, 1);
}
```

在了解这段代码之前，我们先来再重复下php-fpm运行模式。

php-fpm三种运行模式：
- pm = static 静态，始终保持一个固定数量的子进程

- pm = dynamic 启动时，会产生固定数量的子进程（由pm.start_servers控制）可以理解成最小子进程数，而最大子进程数则由pm.max_children去控制，OK，这样的话，子进程数会在最大和最小数范围中变化，还没有完，闲置的子进程数还可以由另2个配置控制，分别是pm.min_spare_servers和pm.max_spare_servers，也就是闲置的子进程也可以有最小和最大的数目，而如果闲置的子进程超出了pm.max_spare_servers，则会被杀掉

- pm = ondemand 按需启动，没有请求时只有主进程

所以这里首先判断php-fpm运行模式是不是ondemand，如果是，那就注册完事件，在有连接时启动，然后直接返回就行了，如果是static模式或dynamic模式，那就要创建worker进程了，创建完worker进程之后，主进程就完成资源初始化了，进入event_loop事件事件监听中，剩下的socket连接就由子进程进行了。

## worker进程

在php-fpm中，worker进程是具体处理socket连接的，它会阻塞等待连接，当有连接时，创建新的文件描述符，worker进程接受连接代码较多，这里只贴出主要逻辑：

```c
while (1) {
	if (req->fd < 0) {
		while (1) {
			if (in_shutdown) {
				return -1;
			}

			req->hook.on_accept();
			{
				int listen_socket = req->listen_socket;
				sa_t sa;
				socklen_t len = sizeof(sa);

				FCGI_LOCK(req->listen_socket);
				req->fd = accept(listen_socket, (struct sockaddr *)&sa, &len);
				FCGI_UNLOCK(req->listen_socket);

				client_sa = sa;
				if (req->fd >= 0 && !fcgi_is_allowed()) {
					fcgi_log(FCGI_ERROR, "Connection disallowed: IP address '%s' has been dropped.", fcgi_get_last_client_ip());
					closesocket(req->fd);
					req->fd = -1;
					continue;
				}
			}

			if (req->fd < 0 && (in_shutdown || (errno != EINTR && errno != ECONNABORTED))) {
				return -1;
			}
			if (req->fd >= 0) {
				struct pollfd fds;
				int ret;

				fds.fd = req->fd;
				fds.events = POLLIN;
				fds.revents = 0;
				do {
					errno = 0;
					ret = poll(&fds, 1, 5000);
				} while (ret < 0 && errno == EINTR);
				if (ret > 0 && (fds.revents & POLLIN)) {
					break;
				}
				fcgi_close(req, 1, 0);

			}
		}
	} else if (in_shutdown) {
		return -1;
	}
	req->hook.on_read();
	if (fcgi_read_request(req)) {
		return req->fd;
	} else {
		fcgi_close(req, 1, 1);
	}
}
```

由以上代码也可以看出，worker进程调用accept阻塞等待连接，当有连接来时，生成文件描述符，然后调用IO复用函数监听此socket连接可读就绪事件，如果在超时时间内事件触发，那就会执行到后面调用zend引擎解释脚本，处理对应请求。否则关闭连接，重新进入阻塞等待模式。

到这里，大致对php-fpm运行模式有个了解了，总的来说，master进程并不直接处理socket请求，而是管理进程资源，具体的请求处理，是由fork出来的worker进程去处理的。

php-fpm运行流程图如下：

{% asset_img php-fpm-start.jpg php-fpm流程图 %}

## php-fpm与redis请求处理比较

若阅读过redis源码，可能会看出与php-fpm调用IO复用函数有些区别。在redis中，master进程会监听服务端socket连接是否有可读可写事件，如果有，那就调用accept()接受连接请求。但是在php-fpm中，不同于redis处理逻辑，每个子进程是先调用accept()阻塞等待，有连接之后，然后调用poll()监听可读就绪事件。


## 其它

我这里创建了一个项目，是在阅读php-fpm源码时测试多进程还有socket用的，还都是php-fpm源码，删除了大量其它内容相关的代码，简化保留多进程，socket相关信息，感兴趣或想动手实践的话可以参考使用。

[mock_php-fpm](https://github.com/baifei2014/mock_php-fpm "mock_php-fpm")

