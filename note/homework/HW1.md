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