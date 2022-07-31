---
title: Docker源码整体分析
categories:
    - 编程
        - docker
tags:
    - docker源码分析
---

## 一、Docker总体架构

Docker对于我们使用者来说，就是一个C/S模式的架构，但是Docker后端是一个非常松耦合的架构，每个模块都有它自己的用处，这些模块组合起来共同支撑着Docker的运行，而且这些模块可以很容易的被其他软件嵌入使用。

先来看一张Docker架构图：

<img src="images/Docker总体架构图.jpg" alt="img" style="zoom: 33%;" />

<!-- more -->

Docker Client就是我们使用的命令行工具`docker`，它与Docker Daemon之间使用Http进行联系，当我们使用`docker`下发容器相关的命令时，Docker Client就会给Docker Daemon发送Post请求。

Docker Daemon以后台进程dockerd驻留在系统后台，它负责接收Docker Client传递过来的请求，然后封装为一个Job，在代码里面实际上一个`Task`，Task中会调度下层的网络、镜像、命令空间、进程创建等模块。

```bash
mucao@mucao-vm:~$ ps aux | grep docker
root        1089  0.8  3.9 1456804 79428 ?       Ssl  12:31   0:01 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

在Driver模块就是具体负责管理容器资源和容器生命周期，我理解它是一个小的调度系统，在里面又分为负责镜像管理的graphdriver、负责网络的networkdriver和负责进程创建的execdriver。

最后真正负责容器运行和生命周期的是libcontainer，它里面包含cgroups、namespace、虚拟网卡等，负责和主机Linux系统打交道。



## 二、Docker各模块源码介绍

目前Docker各个模块都是在自己独立的源码仓库，并且在Docker运行时各个模块也是分块，从这个角度讲Docker不再是一个单独的软件，而是一个虚拟化系统。

在Docker架构图上标注出来了模块与源码仓库对应的关系，如下图所示：

![docker模块与源码仓库对应关系](images/Docker模块与源码仓库对应关系.jpg)



下面我们逐一来介绍：

### 1、cli

docker client在cli仓库，地址为https://github.com/docker/cli，cli主要依赖于`cobra`完成主命令和子命令的构建。

下面这个是`docker`命令的入口函数代码：

```go
// cmd/docker/docker.go文件
func main() {
	dockerCli, err := command.NewDockerCli()
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	logrus.SetOutput(dockerCli.Err())

	if err := runDocker(dockerCli); err != nil {
		if sterr, ok := err.(cli.StatusError); ok {
			if sterr.Status != "" {
				fmt.Fprintln(dockerCli.Err(), sterr.Status)
			}
			// StatusError should only be used for errors, and all errors should
			// have a non-zero exit status, so never exit with 0
			if sterr.StatusCode == 0 {
				os.Exit(1)
			}
			os.Exit(sterr.StatusCode)
		}
		fmt.Fprintln(dockerCli.Err(), err)
		os.Exit(1)
	}
}
...... 省略其他
```

重点在`rundocker`函数:

```go
// cmd/docker/docker.go文件
func runDocker(dockerCli *command.DockerCli) error {
	tcmd := newDockerCommand(dockerCli)
	
    ...... // 省略
    
	// We've parsed global args already, so reset args to those
	// which remain.
	cmd.SetArgs(args)
	return cmd.Execute()
}
...... 省略其他
```

接下来我们看看newDockerCommand函数中做了什么事情：

```go
// cmd/docker/docker.go文件
func newDockerCommand(dockerCli *command.DockerCli) *cli.TopLevelCommand {
	var (
		opts    *cliflags.ClientOptions
		flags   *pflag.FlagSet
		helpCmd *cobra.Command
	)

	cmd := &cobra.Command{
		...... // 省略
	}
	opts, flags, helpCmd = cli.SetupRootCommand(cmd)
	flags.BoolP("version", "v", false, "Print version information and quit")

	setFlagErrorFunc(dockerCli, cmd)

	setupHelpCommand(dockerCli, cmd, helpCmd)
	setHelpFunc(dockerCli, cmd)

	cmd.SetOut(dockerCli.Out())
	commands.AddCommands(cmd, dockerCli)

	cli.DisableFlagsInUseLine(cmd)
	setValidateArgs(dockerCli, cmd)

	// flags must be the top-level command flags, not cmd.Flags()
	return cli.NewTopLevelCommand(cmd, dockerCli, opts, flags)
}
...... 省略其他
```

在这里面主要干了两件事情，一是创建cobra.Command结构体，而是注册子命令，这里面的子命令就包括`docker run`。

```go
main()
|--> command.NewDockerCli()
|--> runDocker(dockerCli)
     |--> newDockerCommand(dockerCli)
     	  |--> cmd := &cobra.Command{...}
          |--> commands.AddCommands(cmd, dockerCli)
          |    |--> 注册其他命令         
          |    |--> container.NewRunCommand(dockerCli)
          |        |--> runRun(dockerCli, cmd.Flags(), &opts, copts) // 实际执行该函数
          |             |--> runContainer(...)
          |                  |--> 向daemon发送post /containers/create
          |                  |--> 向daemon发送post /containers/{id}/start
          |--> tcmd.Initialize()
          |--> cmd.Execute()
      
```



### 2、moby

moby中主要实现了接受请求、创建任务，然后和containerd进行交互，是docker面向用户层面的api封装。地址为https://github.com/moby/moby

![img](images/docker_daemon.jpg)

moby中会启动一个http server，然后注册各种路由表。



### 3、containerd

containerd在整个Docker中占有非常重要的地位，我理解它是一个调度器作用的存在，地址为https://github.com/containerd/containerd。下面我们看一张官网给出的containerd的整体结构图：

![containerd architecture diagram](images/containerd结构图.png)

**containerd**可用作 Linux 和 Windows 的守护进程。它管理其主机系统的完整容器生命周期，从图像传输和存储到容器执行和监督，再到低级存储到网络附件等等。

containerd中有个重要的结构`Task`，这个结构内嵌了`Proess`，也就是说可以当进程一样使用。在containerd这一层看的出来已经逐渐的和操作系统的进程融合了。

```go
// task.go 文件
// Task is the executable object within containerd
type Task interface {
	Process

	// Pause suspends the execution of the task
	Pause(context.Context) error
	// Resume the execution of the task
	Resume(context.Context) error
	// Exec creates a new process inside the task
	Exec(context.Context, string, *specs.Process, cio.Creator) (Process, error)
	// Pids returns a list of system specific process ids inside the task
	Pids(context.Context) ([]ProcessInfo, error)
	// Checkpoint serializes the runtime and memory information of a task into an
	// OCI Index that can be pushed and pulled from a remote resource.
	//
	// Additional software like CRIU maybe required to checkpoint and restore tasks
	// NOTE: Checkpoint supports to dump task information to a directory, in this way,
	// an empty OCI Index will be returned.
	Checkpoint(context.Context, ...CheckpointTaskOpts) (Image, error)
	// Update modifies executing tasks with updated settings
	Update(context.Context, ...UpdateTaskOpts) error
	// LoadProcess loads a previously created exec'd process
	LoadProcess(context.Context, string, cio.Attach) (Process, error)
	// Metrics returns task metrics for runtime specific metrics
	//
	// The metric types are generic to containerd and change depending on the runtime
	// For the built in Linux runtime, github.com/containerd/cgroups.Metrics
	// are returned in protobuf format
	Metrics(context.Context) (*types.Metric, error)
	// Spec returns the current OCI specification for the task
	Spec(context.Context) (*oci.Spec, error)
}
```

Process接口就代表着Linux操作系统中的进程，如下所示：

```go
// process.go 文件
// Process represents a system process
type 
	Wait(context.Context) (<-chan ExitStatus, error)
	// CloseIO allows various pipes to be closed on the process
	CloseIO(context.Context, ...IOCloserOpts) error
	// Resize changes the width and height of the process's terminal
	Resize(ctx context.Context, w, h uint32) error
	// IO returns the io set for the process
	IO() cio.IO
	// Status returns the executing status of the process
	Status(context.Context) (Status, error)
}
```



### 4、runc

runc负责容器生命周期的管理，现在已经是作为一个独立的命令行工具发布。地址为https://github.com/opencontainers/runc。它是OCI(开发容器标准)的实现。

它的执行流程大致如下图：

![runc_init1](images/runc_init1.jpg)



## 三、总结

我们再来总体看一下这些模块源码之间的关系。

![docker模块源码之间关系](images/docker模块源码之间关系.jpeg)

各个模块的执行流程关系如下所示：

![这里写图片描述](images/docker各模块执行关系.png)



## 参考资料

1、[containerd官网](https://containerd.io/)

2、[Docker 源码分析（一）：Docker 架构](https://www.infoq.cn/news/docker-source-code-analysis-part1)



