# homework-shell

## shell的执行顺序

> main
>
> ​	--getcmd//将中断输入的字符全部放在buf缓存区
>
> ​		--cd_cmd?chdir():fork()//判断是否是cd命令，如果是的话就跳转到对应目录，如果不是的话调用fork开启子进程准备
>
> ​			--parsecmd(buf)//解析我们读取到的buf
>
> ​				--parseline(&s,es);
>
> ​					--parsepipe(ps,es);
>
> ​						--parseexec(ps,es);
>
> ​							--execcmd();初始化一个ececcmd的结构体,将type设置为' '(如果不是cd命令一定是一个exec命令), handle recursively
>
> ​								--parseredirs(ret,ps,es)
>
> ​									--peek(ps,es,"<>")//判断buf中是否有"<>"重定向符号

首先这个作业让我们实现了shell中的三个功能分别是执行普通命令、重定向、管道

## 执行普通命令

我们需要实现的仅仅是在使用`execv()`函数来执行OS已经提供的函数功能

因此我们就可以直到`ecmd`结构体的成员分别是什么

```c
struct execcmd {
  int type;              // ' '表明他是一个执行命令
  char *argv[MAXARGS];   // arguments to the command to be exec-ed，这个执行命令的参数列表
};
```

> `argv[0]`是文件的名称，我们需要只用这个在操作系统中寻找他的可执行文件
>
> `argv`用于提供之后的选项’

那么这样实现思路就特变简单了。我们需要在系统找找到argv[0]对应的可执行文件来执行就可以了

具体实现如下：

```c
case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)//如果没有提供函数名称
      _exit(0);
    if(execv(ecmd->argv[0], ecmd->argv) == -1) {//本文件中是否这个文件的可执行文件
      char* path_bin = "/bin/";
      strcat(path_bin,ecmd->argv[0]);
      if(execv(path_bin,ecmd->argv) == -1) {//bin/目录下是否有这个可执行文件
        char* path_usr = "/usr/bin";
        strcat(path_usr,cmd->argv);
        if(execv(path_usr, ecmd->argv) == -1){//usr/bin中是否有可执行文件
          fprintf(stderr, "Command %s can't find\n", ecmd->argv[0]);
          _exit(0);
        }
      }
    }
    break;
```

## 输入输出重定向

输入输出重定向的格式是

`cmd >/< file`

通过这个我们就可以理解`redircmd`中的成员变量

```c
struct redircmd {
  int type;          // < or > 表明cmd类型
  struct cmd *cmd;   // the command to be run (e.g., an execcmd)，我们将会执行的cmd命令是什么？
  char *file;        // the input/output file's name，重定向文件的名称
  int flags;         // flags for open() indicating read or write我们需要对这个文件读还是写？
  int fd;            // the file descriptor number to use for the file，最重要的一点，我们需要关闭的是标准输入还是标准输出fd，这样就可以让我们的file对应的fd来放在0/1上。来实现重定向
};
```

例如：`ls > x`这是一个输出重定向。ls本来的输出是到1上，但是我们希望把他输出到文件中。所以这里需要关闭的fd就是1

cmd -> ls

file -> "x"

fd -> 1

flags -> 读对应的flag

具体实现：

```c
	case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    fprintf(stderr, "redir not implemented\n");
    close(rcmd->fd);//关闭1/0
    if(open(rcmd->file,rcmd->flags) < 0) {//将文件放到1/0
      fprintf(stderr, "file %s can't find\n", rcmd->file);
      _exit(0);
    }
    runcmd(rcmd->cmd);
    break;
```

## 管道

### 管道和重定向有什么区别？

管道的命令格式如下`cmd | cmd`

重定向的命令格式如下`cmd >/< file`

我们可以看到管道对应的是两条命令，而重定向对应的是一个命令和一个文件。

因此管道是两个命令之间的通信，将左边的输出作为右边的输入





那么这样我们就可以通过命令了解管道命令结构的成员是什么了

```c
struct pipecmd {
  int type;          // |
  struct cmd *left;  // left side of pipe
  struct cmd *right; // right side of pipe
};
```

我们需要知道的一些重要特性

- 我们使用`fork()`后，虽然父子变量有着相同的内存内容，但是他们在不同的内存和文件系统中执行。所以我们改变父子进程中的变量是不会相互影响的。子进程是父进程的一个拷贝。
- 当`exec()`函数调用成功后，并不会返回到调用`exec()`的程序中，而是会加载到`exec`对应的程序中去执行。
  - **虽然会替代处理器的内存，但是会保留文件描述附表**。所以这样我们就可以通过`fork()`, `exec()`这两个函数来实现文件的重定向和管道的事项。这个的意思就是caller和callee他们对应的fd_table是相同的。那么exec()的读写fd可以通过fork()出来的子进程来修改。这也就是为什么`fork()`和`exec()`两个函数要分开来实现。

### 那么什么是管道呢？

管道就是一个内核中的缓存区，并且暴露给进程两个端口(两个文件描述符)，他的两端一端用于读，一端用于写，是单向的通道。

那么我们通过第一个子进程关闭标准输出1，然后将p[1]绑定到标准输出1，就能够实现左边的输出存放到管道的缓冲区中。

我们再创建一个自己成关闭标准读入0，然后将p[0]绑定到标准读入1。这样就能够实现，右边命令的输入是管道中读取到缓存中的数据。

### 具体实现

```c
case '|':
    pcmd = (struct pipecmd*)cmd;
    // fprintf(stderr, "pipe not implemented\n");
    // Your code here ...
    int p[2];
    if(pipe(p) < 0) {
      fprintf(stderr,"Fail to create a pipe\n");
      _exit(0);
    }
    if(fork1() == 0) {
      close(1);//关闭标准输出
      dup(p[1]);//将管道的输出端口定位到标准输出，那么接下来执行的文件的输出将存放早管道中
      close(p[0]);//关闭子进程中的两个管道，避免出现无限循环
      close(p[1]);//
      runcmd(cmd->left);
    }
    if(fork1() == 0) {
      close(0);//关闭标准读入
      dup(p[0]);//将管道的读出段定位到标准读取，那么就下来命令的输入就是pipe中的内容
      close(p[0]);//关闭防止无限玄幻
      close(p[1]);
      runcmd(cmd->right);
    }
    break;
  }    
//关闭两个管道，防止无限循环
  close(p[0]);
  close(p[1]);
//等待两个进程的执行完成
  wait(&r);
  wait(&r);
  _exit(0);
```

