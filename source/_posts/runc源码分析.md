---
title: 容器进程是怎么启动的？（runc源码分析）
categories:
    - 编程
        - docker
tags:
    - runc
---


为了探究标题上的问题，我们先来看看runc启动容器的示例。

## 一、runc启动容器示例

依据runc的文档，可以知道启动一个容器有下面的步骤。

### 1、创建OCI 容器启动包

使用下面的命令生成rootfs：

```bash
# create the top most bundle directory
$ mkdir /mycontainer
$ cd /mycontainer

# create the rootfs directory
$ mkdir rootfs

# export busybox via Docker into the rootfs directory
$ docker export $(docker create busybox) | tar -C rootfs -xvf -
```

roortfs生成之后，只需要在生成规格说明文件config.json，就完成了OCI容器启动包。使用下面的命令可以生成一个基本的config.json文件：

```bash
$ runc spec
```

<!-- more -->


### 2、运行容器

依据前面的步骤，你已经有了一个OCI容器包，现在你可以有两种方式执行容器。

第一种方式是你可以使用`runc`命令轻松的创建、开启、删除容器。下面的命令可以开启容器：

```bash
# run as root
cd /mycontainer
runc run mycontainerid
```

如果你使用的是没有修改的`runc spec`生成的config.json模板，那么你将会得到一个运行在容器中的sh会话。

下面是容器启动之后的截图：

![image-20220805090657093](images/runc启动容器.png)

第二种开启容器的方式是使用spec声明周期操作，具体的就是修改config.json文件。这种方式可以有效的管理容器的创建和运行。如果移除了config.json中的`terminal`设置，那么容器就会在后台运行，下面是一个简单的例子。在config.json文件`process`字段设置`"terminal": false` 和 `"args": ["sleep", "10"]`.

```json
        "process": {
                "terminal": false,
                "user": {
                        "uid": 0,
                        "gid": 0
                },
                "args": [
                        "sleep", "10"
                ],
                "env": [
                        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                        "TERM=xterm"
                ],
                "cwd": "/",
                "capabilities": {
                        "bounding": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "effective": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "inheritable": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "permitted": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "ambient": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ]
                },
                "rlimits": [
                        {
                                "type": "RLIMIT_NOFILE",
                                "hard": 1024,
                                "soft": 1024
                        }
                ],
                "noNewPrivileges": true
        },
```

现在我们就可以在shell中进行生命周期操作了：

```bash
# run as root
cd /mycontainer
runc create mycontainerid

# view the container is created and in the "created" state
runc list

# start the process inside the container
runc start mycontainerid

# after 5 seconds view that the container has exited and is now in the stopped state
runc list

# now delete the container
runc delete mycontainerid
```

下面是执行容器生命周期操作的截图：

![image-20220805100024378](images/runc容器生命周期实际操作.png)

这种方式启动容器，可以运行更高层级的系统配置容器创建逻辑。例如，容器的网络栈一般就是在`create`之后`start`之前设置的。

接下来我们就看看runc代码中是怎么完成容器生命周期的，下文的源码分析基于runc源码release-1.1版本

## 二、runc容器创建分析

runc的命令行是使用urfave/cli创建的。

首先在main函数中创建并注册了命令：

```go
// main.go 文件
func main() {
    	app := cli.NewApp()
	app.Name = "runc"
	app.Usage = usage
    ...... // 省略
    
    // 注册命令
    app.Commands = []cli.Command{
		checkpointCommand,
		createCommand, //  创建容器的命令  // ref-1
		deleteCommand,
		eventsCommand,
		execCommand,
		killCommand,
		listCommand,
		pauseCommand,
		psCommand,
		restoreCommand,
		resumeCommand,
		runCommand,
		specCommand,
		startCommand, // 开启容器的命令  // ref-2
		stateCommand,
		updateCommand,
		featuresCommand,
	}
    ...... // 省略
    
    // 执行命令
    if err := app.Run(os.Args); err != nil {
		fatal(err)
	}
}
```

我们接下来看看`ref-1`处的`createCommand`命令都干了些什么事情？

```go
// create.go 文件
var createCommand = cli.Command{
	Name:  "create",
	Usage: "create a container",
	ArgsUsage: `<container-id>

Where "<container-id>" is your name for the instance of the container that you
are starting. The name you provide for the container instance must be unique on
your host.`,
	Description: `The create command creates an instance of a container for a bundle. The bundle
is a directory with a specification file named "` + specConfig + `" and a root
filesystem.

The specification file includes an args parameter. The args parameter is used
to specify command(s) that get run when the container is started. To change the
command(s) that get executed on start, edit the args parameter of the spec. See
"runc spec --help" for more explanation.`,
	Flags: []cli.Flag{
		...... // 省略flag绑定
	},
	Action: func(context *cli.Context) error {
		if err := checkArgs(context, 1, exactArgs); err != nil {
			return err
		}
 		// 执行创建容器命令的函数
		status, err := startContainer(context, CT_ACT_CREATE, nil) // ref-3
		if err == nil {
			// exit with the container's exit status so any external supervisor
			// is notified of the exit with the correct exit status.
			os.Exit(status)
		}
		return fmt.Errorf("runc create failed: %w", err)
	},
}
```

注意`create`命令的`Description`部分，它指出了`runc`会依据由`rootfs`和`config.json`组成的bundle来创建容器，如果有什么定制化，那么就去修改`config.json`文件。

我们在`createCommand`的`Action`字段看见了处理容器创建命令的真实函数`startContainer(context, CT_ACT_CREATE, nil)`，也就是代码`ref-3`处。我们接着往下看。

```go
// utils_linux.go 文件
func startContainer(context *cli.Context, action CtAct, criuOpts *libcontainer.CriuOpts) (int, error) {
    ...... // 省略
    // 获取规格说明
	spec, err := setupSpec(context)
	// 获取传入的容器id
	id := context.Args().First()
    ...... // 省略
    // 加载factory，创建libcontainer.Container对象，实际上就是创建容器需要用到的目录
	container, err := createContainer(context, id, spec)
    ...... // 省略
	r := &runner{
		enableSubreaper: !context.Bool("no-subreaper"),
		shouldDestroy:   !context.Bool("keep"),
		container:       container,
		listenFDs:       listenFDs,
		notifySocket:    notifySocket,
		consoleSocket:   context.String("console-socket"),
		detach:          context.Bool("detach"),
		pidFile:         context.String("pid-file"),
		preserveFDs:     context.Int("preserve-fds"),
		action:          action,
		criuOpts:        criuOpts,
		init:            true,
	}
    // 执行容器指定的命令
	return r.run(spec.Process)  // ref-4
}
```

接下来我们看看`ref-4`处的`run`函数内容：

```go
// utils_linux.go 文件
func (r *runner) run(config *specs.Process) (int, error) {
	...... // 省略
    // 创建容器进程结构体libcontainer.Process对象
	process, err := newProcess(*config) // ref-5
	process.LogLevel = strconv.Itoa(int(logrus.GetLevel()))
	// Populate the fields that come from runner.
	process.Init = r.init
	process.SubCgroupPaths = r.subCgroupPaths
	if len(r.listenFDs) > 0 {
		process.Env = append(process.Env, "LISTEN_FDS="+strconv.Itoa(len(r.listenFDs)), "LISTEN_PID=1")
		process.ExtraFiles = append(process.ExtraFiles, r.listenFDs...)
	}
	...... // 省略
	switch r.action {
	case CT_ACT_CREATE:
		err = r.container.Start(process)  // ref-6
	case CT_ACT_RESTORE:
		err = r.container.Restore(process, r.criuOpts)
	case CT_ACT_RUN:
		err = r.container.Run(process)
	default:
		panic("Unknown action")
	}
	...... // 省略
    // 处理信号
	status, err := handler.forward(process, tty, detach)
	...... // 省略
	return status, err
}
```

在`ref-5`处创建了一个代表容器进程的`process`对象，然后再`ref-6`处执行了启动，我们看看`r.container.Start(process) `又干了什么事情？

```go
// libcontainer/container_linux.go 文件
func (c *linuxContainer) Start(process *Process) error {
	...... // 省略
	if process.Init { // ref-7
        // 创建用于和子进程通信的fifo文件
		if err := c.createExecFifo(); err != nil {
			return err
		}
	}
	if err := c.start(process); err != nil { // ref-8
		if process.Init {
			c.deleteExecFifo()
		}
		return err
	}
	return nil
}
```

在`ref-7`处会判断`process`是否为容器中的第一个进程，如果是的话，就会调用`c.createExecFifo()`创建后续与子进程通信的管道。

在`ref-8`处就会开启容器中的进程process，我们看看runc是怎么开启进程的？

```go
// container_linux.go 文件
func (c *linuxContainer) start(process *Process) (retErr error) {
	// 创建父进程
    parent, err := c.newParentProcess(process) // ref-9
	// 设置同步子进程的日志
	logsDone := parent.forwardChildLogs()
	if logsDone != nil {
		defer func() {
			// Wait for log forwarder to finish. This depends on
			// runc init closing the _LIBCONTAINER_LOGPIPE log fd.
			err := <-logsDone
			if err != nil && retErr == nil {
				retErr = fmt.Errorf("unable to forward init logs: %w", err)
			}
		}()
	}

	if err := parent.start(); err != nil { // ref-10
		return fmt.Errorf("unable to start container process: %w", err)
	}
	...... // 省略
	return nil
}
```

在`ref-9`创建了一个父进程，这个可能后面需要好好看看，我们先看下`ref-10`处的父进程开启操作。

```go
// process_linux.go 文件
func (p *initProcess) start() (retErr error) {
	err := p.cmd.Start() // ref-11 使用系统调用创建了子进程，但是和当前进程并不是父子进程关系
	p.process.ops = p
	// close the write-side of the pipes (controlled by child)
	_ = p.messageSockPair.child.Close()
	_ = p.logFilePair.child.Close()
	if err != nil {
		p.process.ops = nil
		return fmt.Errorf("unable to start init: %w", err)
	}
	...... // 省略
    // 创建等待子进程完成任务的channel
	waitInit := initWaiter(p.messageSockPair.parent)
	...... // 省略
    // 将启动数据bootstrapData发送给子进程
	if _, err := io.Copy(p.messageSockPair.parent, p.bootstrapData); err != nil { // ref-16
		return fmt.Errorf("can't copy bootstrap data to pipe: %w", err)
	}
	err = <-waitInit // 等待子进程完成工作
	if err != nil {
		return err
	}

	childPid, err := p.getChildPid()
	if err != nil {
		return fmt.Errorf("can't get final child's PID from pipe: %w", err)
	}
	...... // 省略
    // 创建网卡
	if err := p.createNetworkInterfaces(); err != nil {
		return fmt.Errorf("error creating network interfaces: %w", err)
	}
    // 更新状态
	if err := p.updateSpecState(); err != nil {
		return fmt.Errorf("error updating spec state: %w", err)
	}
    // 向子进程发送配置
	if err := p.sendConfig(); err != nil {
		return fmt.Errorf("error sending config to init process: %w", err)
	}
	var (
		sentRun    bool
		sentResume bool
	)

	ierr := parseSync(p.messageSockPair.parent, func(sync *syncT) error {
		// 处理子进程消息的逻辑	
    }
	// 关闭和子进程通信的管道
	if err := unix.Shutdown(int(p.messageSockPair.parent.Fd()), unix.SHUT_WR); err != nil {
		return &os.PathError{Op: "shutdown", Path: "(init pipe)", Err: err}
	}
	...... // 省略
	return nil
}

```

`ref-11`处的`cmd`其实是由下面的代码`ref-12`中指定的：

```go
// libcontainer/factory_linux.go 文件
// New returns a linux based container factory based in the root directory and
// configures the factory with the provided option funcs.
func New(root string, options ...func(*LinuxFactory) error) (Factory, error) {
	...... // 省略
	l := &LinuxFactory{
		Root:      root,
		InitPath:  "/proc/self/exe", // ref-12 执行路径其实就是runc自己
		InitArgs:  []string{os.Args[0], "init"},
		Validator: validate.New(),
		CriuPath:  "criu",
	}
	...... // 省略
	return l, nil
}
```

现在的重点是这个父进程`parentProcess`里面具体都有些什么？我们来看看`ref-9`处的`c.newParentProcess(process)`实现：

```go
// libcontainer/container_linux.go 文件
func (c *linuxContainer) newParentProcess(p *Process) (parentProcess, error) {
	// 创建用于父子进程传递消息的文件对
    parentInitPipe, childInitPipe, err := utils.NewSockPair("init")
	if err != nil {
		return nil, fmt.Errorf("unable to create init pipe: %w", err)
	}
	messageSockPair := filePair{parentInitPipe, childInitPipe}

  	// 创建用于父子进程传输日志的文件对
	parentLogPipe, childLogPipe, err := os.Pipe()
	if err != nil {
		return nil, fmt.Errorf("unable to create log pipe: %w", err)
	}  
	logFilePair := filePair{parentLogPipe, childLogPipe}

    // 依据模板创建命令
	cmd := c.commandTemplate(p, childInitPipe, childLogPipe) // ref-13
	if !p.Init {
        // 如果不是容器的第一个进程就创建SetnsProcess
		return c.newSetnsProcess(p, cmd, messageSockPair, logFilePair) // ref-14
	}

	// We only set up fifoFd if we're not doing a `runc exec`. The historic
	// reason for this is that previously we would pass a dirfd that allowed
	// for container rootfs escape (and not doing it in `runc exec` avoided
	// that problem), but we no longer do that. However, there's no need to do
	// this for `runc exec` so we just keep it this way to be safe.
	if err := c.includeExecFifo(cmd); err != nil {
		return nil, fmt.Errorf("unable to setup exec fifo: %w", err)
	}
    // 如果是容器的第一个进程就创建InitProcess
	return c.newInitProcess(p, cmd, messageSockPair, logFilePair) // ref-15
}
```

我接下来看看`ref-13`处的命令模板情况：

```go
// libcontainer/container_linux.go 文件
func (c *linuxContainer) commandTemplate(p *Process, childInitPipe *os.File, childLogPipe *os.File) *exec.Cmd {
    // 创建操作系统支持的command
	cmd := exec.Command(c.initPath, c.initArgs[1:]...)
    // 下面就是设置命令的参数和环境变量，其实就是在向子进程传递信息
	cmd.Args[0] = c.initArgs[0]
	cmd.Stdin = p.Stdin
	cmd.Stdout = p.Stdout
	cmd.Stderr = p.Stderr
	cmd.Dir = c.config.Rootfs
	if cmd.SysProcAttr == nil {
		cmd.SysProcAttr = &unix.SysProcAttr{}
	}
	cmd.Env = append(cmd.Env, "GOMAXPROCS="+os.Getenv("GOMAXPROCS"))
	cmd.ExtraFiles = append(cmd.ExtraFiles, p.ExtraFiles...)
	if p.ConsoleSocket != nil {
		cmd.ExtraFiles = append(cmd.ExtraFiles, p.ConsoleSocket)
		cmd.Env = append(cmd.Env,
			"_LIBCONTAINER_CONSOLE="+strconv.Itoa(stdioFdCount+len(cmd.ExtraFiles)-1),
		)
	}
	cmd.ExtraFiles = append(cmd.ExtraFiles, childInitPipe)
	cmd.Env = append(cmd.Env,
		"_LIBCONTAINER_INITPIPE="+strconv.Itoa(stdioFdCount+len(cmd.ExtraFiles)-1),
		"_LIBCONTAINER_STATEDIR="+c.root,
	)

	cmd.ExtraFiles = append(cmd.ExtraFiles, childLogPipe)
	cmd.Env = append(cmd.Env,
		"_LIBCONTAINER_LOGPIPE="+strconv.Itoa(stdioFdCount+len(cmd.ExtraFiles)-1),
		"_LIBCONTAINER_LOGLEVEL="+p.LogLevel,
	)
	return cmd
}
```

接着我们看看`ref-15`处的`InitProcess`创建过程：

```go
// libcontainer/container_linux.go 文件
func (c *linuxContainer) newInitProcess(p *Process, cmd *exec.Cmd, messageSockPair, logFilePair filePair) (*initProcess, error) {
    // 追加环境变量
	cmd.Env = append(cmd.Env, "_LIBCONTAINER_INITTYPE="+string(initStandard))
    // 设置namespace信息
	nsMaps := make(map[configs.NamespaceType]string)
	for _, ns := range c.config.Namespaces { // ref-17
		if ns.Path != "" {
			nsMaps[ns.Type] = ns.Path
		}
	}
	_, sharePidns := nsMaps[configs.NEWPID]
    // 创建子进程的bootstrap数据
	data, err := c.bootstrapData(c.config.Namespaces.CloneFlags(), nsMaps, initStandard)// ref-23
	if err != nil {
		return nil, err
	}

	if c.shouldSendMountSources() {
		...... // 省略创建挂载信息的代码
		cmd.Env = append(cmd.Env,
			"_LIBCONTAINER_MOUNT_FDS="+string(mountFdsJson),
		)
	}

	init := &initProcess{
		cmd:             cmd, // 其实就是/proc/self/exe   runc命令自己的执行程序
		messageSockPair: messageSockPair, // 消息通信
		logFilePair:     logFilePair, // 日志通信
		manager:         c.cgroupManager, // cgroups管理
		intelRdtManager: c.intelRdtManager,
		config:          c.newInitConfig(p), // 初始化配置
		container:       c,  // 容器对象
		process:         p, // 进程对象
		bootstrapData:   data, // 子进程引导数据
		sharePidns:      sharePidns, // 子进程要完成的pid命名空间切换
	}
	c.initProcess = init
	return init, nil
}
```

`ref-17`在设置namespace信息组成的ma，,根据config.json中namespaces配置，生成一个namespace为key，path为value的map。这个namespace的map在构造bootstrapdata时，会判断每个namespace是否配置了path，如果配置了，则之后不再创建新的namespace，而是将这种类型的namespace  join 到这个path下。

我们再来看看`ref-23`处的代码都创建了哪些bootstrap参数：

```go
// mapping etc.
func (c *linuxContainer) bootstrapData(cloneFlags uintptr, nsMaps map[configs.NamespaceType]string, it initType) (_ io.Reader, Err error) {
	// create the netlink message
	r := nl.NewNetlinkRequest(int(InitMsg), 0)
	...... // 省略
	// write cloneFlags
	r.AddData(&Int32msg{
		Type:  CloneFlagsAttr,
		Value: uint32(cloneFlags),
	})

	// write custom namespace paths
	if len(nsMaps) > 0 {
		nsPaths, err := c.orderNamespacePaths(nsMaps)
		if err != nil {
			return nil, err
		}
		r.AddData(&Bytemsg{
			Type:  NsPathsAttr,
			Value: []byte(strings.Join(nsPaths, ",")),
		})
	}

	// write namespace paths only when we are not joining an existing user ns
	_, joinExistingUser := nsMaps[configs.NEWUSER]
	if !joinExistingUser {
		// write uid mappings
		if len(c.config.UidMappings) > 0 {
            ...... // 写入uid
		}

		// write gid mappings
		if len(c.config.GidMappings) > 0 {
			...... // 写入gid
		}
	}

	if c.config.OomScoreAdj != nil {
		// write oom_score_adj
		r.AddData(&Bytemsg{
			Type:  OomScoreAdjAttr,
			Value: []byte(strconv.Itoa(*c.config.OomScoreAdj)),
		})
	}

	// write rootless
	r.AddData(&Boolmsg{
		Type:  RootlessEUIDAttr,
		Value: c.config.RootlessEUID,
	})

	// Bind mount source to open.
	if it == initStandard && c.shouldSendMountSources() {
		...... // 构建挂载信息
		r.AddData(&Bytemsg{
			Type:  MountSourcesAttr,
			Value: mounts,
		})
	}
	// 通过netlink消息格式封装
	return bytes.NewReader(r.Serialize()), nil
}
```

bootstrapData: 构造出 run init 阶段需要用到的启动参数。包括clone(2)的clone flag，uid/gid映射, oom阈值，rootless相关的设置等等。这些配置都通过netlink消息格式封装。然后通过之前的那对unix socketpair将其发送给子进程。

现在就是有一个疑问了，子进程中是怎么处理这些信息的呢？子进程又给父进程发送了哪些信息呢？

这些疑问的答案都被runc隐藏在`init.go`文件中了。

```go
// init.go 文件
package main

import (
	"os"
	"runtime"
	"strconv"

	"github.com/opencontainers/runc/libcontainer"
	_ "github.com/opencontainers/runc/libcontainer/nsenter" // ref-22
	"github.com/sirupsen/logrus"
)

func init() {
	if len(os.Args) > 1 && os.Args[1] == "init" { // ref-18
		// This is the golang entry point for runc init, executed
		// before main() but after libcontainer/nsenter's nsexec().
		runtime.GOMAXPROCS(1)
		runtime.LockOSThread() // ref-19

		level, err := strconv.Atoi(os.Getenv("_LIBCONTAINER_LOGLEVEL"))
		if err != nil {
			panic(err)
		}

		logPipeFd, err := strconv.Atoi(os.Getenv("_LIBCONTAINER_LOGPIPE"))
		if err != nil {
			panic(err)
		}

		logrus.SetLevel(logrus.Level(level))
		logrus.SetOutput(os.NewFile(uintptr(logPipeFd), "logpipe"))
		logrus.SetFormatter(new(logrus.JSONFormatter))
		logrus.Debug("child process in init()")

		factory, _ := libcontainer.New("") // ref-20
		if err := factory.StartInitialization(); err != nil { // ref-21
			// as the error is sent back to the parent there is no need to log
			// or write it to stderr because the parent process will handle this
			os.Exit(1)
		}
		panic("libcontainer: container init failed to exec")
	}
}
```

特别注意`ref-18`处的注释，这个`init()`方法会`在main()`函数之前，`libcontainer/nsenter`的`nsexec()`之后，执行。这个`libcontainer/nsenter`是在`import`中导入的，在`ref-22`处代码。

我们得先看看`nsexec()`函数干了些什么事情？

`nsexec()`是使用C代码实现的，具体的作用就是实现命名空间的切换。这段C代码我没有看懂，参考[mount 内核源码_runc源码分析](https://blog.csdn.net/weixin_39970689/article/details/112454983)进行了理解。

我们就下来再回过头来看看`initProcess`的`start()`函数：

```go
// libcontainer/process_linux.go 文件
func (p *initProcess) start() (retErr error) {
	defer p.messageSockPair.parent.Close() //nolint: errcheck
	err := p.cmd.Start() // 开始执行 runc init  // ref-25

	// 向runc init 子进程发送bootstrap数据
	if _, err := io.Copy(p.messageSockPair.parent, p.bootstrapData); err != nil {
		return fmt.Errorf("can't copy bootstrap data to pipe: %w", err)
	}
    ...... // 省略
	//  等待第一个子进程退出
	if err := p.waitForChildExit(childPid); err != nil { // ref-24
		return fmt.Errorf("error waiting for our first child to exit: %w", err)
	}
	//  设置网卡状态为up
	if err := p.createNetworkInterfaces(); err != nil {
		return fmt.Errorf("error creating network interfaces: %w", err)
	}
	//  更新当前状态到 state.json
	if err := p.updateSpecState(); err != nil {
		return fmt.Errorf("error updating spec state: %w", err)
	}
	//  向子进程runc init发送配置。子进程runc init在ref-24处已经退出了，这个其实是给子进程runc init的子进程发送数据, 也就是ref-25处start()子进程创建的子进程
	if err := p.sendConfig(); err != nil {
		return fmt.Errorf("error sending config to init process: %w", err)
	}	
	//  等待子进程发送回来的同步消息
	ierr := parseSync(p.messageSockPair.parent, func(sync *syncT) error {
		switch sync.Type {
		case procSeccomp:
			...... // 省略
		case procReady:
			...... // 省略
		case procHooks:
			...... // 省略
		default:
			return errors.New("invalid JSON payload from child")
		}

		return nil
	})
    ...... // 省略
	// 开始关闭自己
	if err := unix.Shutdown(int(p.messageSockPair.parent.Fd()), unix.SHUT_WR); err != nil {
		return &os.PathError{Op: "shutdown", Path: "(init pipe)", Err: err}
	}

	// Must be done after Shutdown so the child will exit and we can wait for it.
	if ierr != nil {
		_, _ = p.wait()
		return ierr
	}
	return nil
}
```

为了更清楚地理解目前几个进程之间的关系，按照进程出现的先后顺序，暂且将几个进程称为:

**runc:CREATE -> runc:[0:PARENT] -> runc:[1:CHILD] -> runc:[2:INIT]**

其中，**runc:[0:PARENT]** clone时指定了**CLONE_PARENT**，后面三个进程在进程父子视图上并不是 父子关系，而是 兄弟关系。如下图所示：

![b59f16d2db630309761b51d4c6fa6b47.png](images/runc三个进程关系图.png)

下面我们通过一张图来理解`nsexec()`函数：

![9acc46b0cbac77817c9aa450545a37a5.png](images/nsexec函数进程执行流程.png)

runc中实现切换新的namespace都是使用unshare()。

namespace切换发生在runc:[1:CHILD]。CLONE_NEWUSER首先需要单独做一次unshare()。CLONE_NEWUSER不能与CLONE_PARENT同时指定。具体细节读者可见：man 2 clone

runc:[1:CHILD]执行完unshare(cloneflags&~CLONE_NEWGROUP),主要的namespaces已经改变了，但CLONE_NEWPID这个namespace并未发生切换，需要再次clone()。

一旦clone_parent(JUMP_INIT)开始执行，runc:[2:INIT]进程产生，pid namespae切换。runc:[1:CHILD]进程一只脚在容器内，一只脚在容器外；而runc:[2:INIT]是完全进入到容器的进程。

runc:[1:CHILD]，runc:[0:PARENT] 相继退出，runc:[2:INIT]中执行完nsexec()函数后，后续流程开始进入runc init剩下的go runtime部分。

当`nsexec()`函数执行完之后，就该执行`init.go`中的初始化函数了，下面我们一起看看：

```go
// libcontainer/factory_linux.go 文件
// StartInitialization loads a container by opening the pipe fd from the parent to read the configuration and state
// This is a low level implementation detail of the reexec and should not be consumed externally
func (l *LinuxFactory) StartInitialization() (err error) {
	// Get the INITPIPE.
	envInitPipe := os.Getenv("_LIBCONTAINER_INITPIPE")
	pipefd, err := strconv.Atoi(envInitPipe)
	pipe := os.NewFile(uintptr(pipefd), "pipe")
	defer pipe.Close()
	...... // 省略
	// Only init processes have FIFOFD.
	fifofd := -1
	envInitType := os.Getenv("_LIBCONTAINER_INITTYPE")
	it := initType(envInitType)
	if it == initStandard {
		envFifoFd := os.Getenv("_LIBCONTAINER_FIFOFD")
		if fifofd, err = strconv.Atoi(envFifoFd); err != nil {
			return fmt.Errorf("unable to convert _LIBCONTAINER_FIFOFD: %w", err)
		}
	}
	...... // 省略
	// Get mount files (O_PATH).
	mountFds, err := parseMountFds()
	if err != nil {
		return err
	}

	// clear the current process's environment to clean any libcontainer
	// specific env vars.
	os.Clearenv()
	...... // 省略
    // 创建容器初始化器
	i, err := newContainerInit(it, pipe, consoleSocket, fifofd, logPipeFd, mountFds) // ref-26
	if err != nil {
		return err
	}

	// If Init succeeds, syscall.Exec will not return, hence none of the defers will be called.
	return i.Init() // ref-27
}
```

接下来最后来看看容器初始化器做了什么？

```go
// libcontainer/standard_init_linux.go 文件
func (l *linuxStandardInit) Init() error { // 这个函数在子进程runc init的子进程中执行。
	...... // 省略
    // 设置网卡为up
	if err := setupNetwork(l.config); err != nil {
		return err
	}
    // 设置路由表
	if err := setupRoute(l.config.Config); err != nil {
		return err
	}

	// initialises the labeling system
	selinux.GetEnabled()

	// We don't need the mountFds after prepareRootfs() nor if it fails.
    // 准备rootfs
	err := prepareRootfs(l.pipe, l.config, l.mountFds)
	for _, m := range l.mountFds {
		if m == -1 {
			continue
		}

		if err := unix.Close(m); err != nil {
			return fmt.Errorf("Unable to close mountFds fds: %w", err)
		}
	}
	...... // 省略
	// Tell our parent that we're ready to Execv. This must be done before the
	// Seccomp rules have been applied, because we need to be able to read and
	// write to a socket.
    // 告诉父进程，现在已经都准备好了，父进程(parentProcess)收到消息后就会退出
	if err := syncParentReady(l.pipe); err != nil {
		return fmt.Errorf("sync ready: %w", err)
	}
		...... // 省略
	// Wait for the FIFO to be opened on the other side before exec-ing the
	// user process. We open it through /proc/self/fd/$fd, because the fd that
	// was given to us was an O_PATH fd to the fifo itself. Linux allows us to
	// re-open an O_PATH fd through /proc.
	fifoPath := "/proc/self/fd/" + strconv.Itoa(l.fifoFd)
    // 这个open方法会导致阻塞，直到另外一端也打开
	fd, err := unix.Open(fifoPath, unix.O_WRONLY|unix.O_CLOEXEC, 0) // ref-27
    // 阻塞结束后，就真正的执行用户指定的容器中执行的命令了
	// 告诉父进程现在开始启动用户给容器指定的命令。
	if _, err := unix.Write(fd, []byte("0")); err != nil { // ref-28
		return &os.PathError{Op: "write exec fifo", Path: fifoPath, Err: err}
	}

	// Close the O_PATH fifofd fd before exec because the kernel resets
	// dumpable in the wrong order. This has been fixed in newer kernels, but
	// we keep this to ensure CVE-2016-9962 doesn't re-emerge on older kernels.
	// N.B. the core issue itself (passing dirfds to the host filesystem) has
	// since been resolved.
	// https://github.com/torvalds/linux/blob/v4.9/fs/exec.c#L1290-L1318
	_ = unix.Close(l.fifoFd)

	s := l.config.SpecState
	s.Pid = unix.Getpid()
	s.Status = specs.StateCreated
	if err := l.config.Config.Hooks[configs.StartContainer].RunHooks(s); err != nil {
		return err
	}
	// 使用系统调用执行用户指定的容器要执行的命令，伴随着的还有子进程runc init的子进程也就退出了。
	return system.Exec(name, l.config.Args[0:], os.Environ())
}
```

## 三、容器启动分析

上面已经分析到`runc create`命令最后一步就是在等待一个消息，然后就开启用户指定的容器命令了。这个消息就是`runc start`发出的。接下来我们具体看看。

首先看看`runc start`命令的执行函数：

```go
// start.go文件
var startCommand = cli.Command{
	Name:  "start",
	Usage: "executes the user defined process in a created container",
	ArgsUsage: `<container-id>

Where "<container-id>" is your name for the instance of the container that you
are starting. The name you provide for the container instance must be unique on
your host.`,
	Description: `The start command executes the user defined process in a created container.`,
	Action: func(context *cli.Context) error {
		if err := checkArgs(context, 1, exactArgs); err != nil {
			return err
		}
		container, err := getContainer(context)
		if err != nil {
			return err
		}
		status, err := container.Status()
		if err != nil {
			return err
		}
		switch status {
		case libcontainer.Created: // ref-29
            // ref-30
			notifySocket, err := notifySocketStart(context, os.Getenv("NOTIFY_SOCKET"), container.ID())
			if err != nil {
				return err
			}
			if err := container.Exec(); err != nil { // ref-31
				return err
			}
			if notifySocket != nil {
                // ref-32
				return notifySocket.waitForContainer(container)
			}
			return nil
		case libcontainer.Stopped:
			return errors.New("cannot start a container that has stopped")
		case libcontainer.Running:
			return errors.New("cannot start an already running container")
		default:
			return fmt.Errorf("cannot start a container in the %s state", status)
		}
	},
}
```

`ref-29`处在判断容器的当前状态，`ref-31`的`container.Exec()`，我们来看看里面具体干了啥：

```go
// libcontainer/container_linux.go 文件
func (c *linuxContainer) exec() error { // linux container 执行start任务的具体函数
	path := filepath.Join(c.root, execFifoFilename) // 获取用于子进程通信的文件路径
	pid := c.initProcess.pid()
    blockingFifoOpenCh := awaitFifoOpen(path) // 打开文件，子进程(runc init的子进程)阻塞结束
	for {
		select {
		case result := <-blockingFifoOpenCh: // 收到子进程发送过来的数据
			return handleFifoResult(result) // ref-32 读取数据就返回了，这儿就真的只是读取数据，啥也没干

		case <-time.After(time.Millisecond * 100): // 超时控制
			stat, err := system.Stat(pid)
			if err != nil || stat.State == system.Zombie {
				// could be because process started, ran, and completed between our 100ms timeout and our system.Stat() check.
				// see if the fifo exists and has data (with a non-blocking open, which will succeed if the writing process is complete).
				if err := handleFifoResult(fifoOpen(path, false)); err != nil {
					return errors.New("container process is already dead")
				}
				return nil
			}
		}
	}
}
```

我们最后来看看`ref-32`的数据读取都做了什么操作？

```go
// libcontainer/container_linux.go 文件
func handleFifoResult(result openResult) error {
	if result.err != nil {
		return result.err
	}
	f := result.file
	defer f.Close()
	if err := readFromExecFifo(f); err != nil {
		return err
	}
	return os.Remove(f.Name())
}

// 就是读取了一下文件管道中的数据，然后真的就啥也没干！
func readFromExecFifo(execFifo io.Reader) error {
	data, err := io.ReadAll(execFifo)
	if err != nil {
		return err
	}
	if len(data) <= 0 {
		return errors.New("cannot start an already running container")
	}
	return nil
}
```



到这儿，docker 容器进程启动分析就结束了。可以看到runc要想把容器中用户指定的命令进程完全与宿主机隔离，还是非常曲折的。



参考资料：

1、https://mkdev.me/posts/the-tool-that-really-runs-your-containers-deep-dive-into-runc-and-oci-specifications

2、runc仓库地址：https://github.com/opencontainers/runc

3、mount 内核源码_runc源码分析：https://blog.csdn.net/weixin_39970689/article/details/112454983