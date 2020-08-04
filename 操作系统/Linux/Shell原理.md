shell 的中文翻译为“外壳”，它的定义就是提供接口给用户与[操作系统](https://www.tomorrow.wiki/archives/tag/操作系统)内核交互的软件。简单来说就是一个可以分析并执行用户所输入的命令的软件。shell 的工作流程应该是这样的：

1. 打印命令提示符$或者#；

2. 读取并分析命令；

3. 执行命令；

4. 执行完命令后，重复 1-3；

例如下面的代码：

```c
while（true）
{
        printf("$");//打印命令提示符
        if('p'==gerchar())//读取命令；分析命令
                print("Hello ,I am a shell .\n");//执行命令
}
```

## 内建命令与外部命令

### 内部命令

内部命令实际上是[shell](https://www.tomorrow.wiki/archives/tag/shell)程序本身的一部分，通常都是一些比较简单的系统命令。这些命令所实现的功能与所做工作都是由[shell](https://www.tomorrow.wiki/archives/tag/shell)程序本身来完成的，也就是在[shell](https://www.tomorrow.wiki/archives/tag/shell)程序的源码里面实现的，其执行速度要比外部命令快很多，**因为执行内部命令时，[shell](https://www.tomorrow.wiki/archives/tag/shell)无需创建新的进程产生多余的开销**。

### 外部命令

外部命令区别于内建命令，通常是一些功能较为强大、复杂的命令。**它由[shell](https://www.tomorrow.wiki/archives/tag/shell)分析然后通过**[Linux](https://www.tomorrow.wiki/archives/tag/linux)**内核 API 创建新的进程，在新的进程中执行**，在新的进程中所执行的代码是不属于 shell 的，**所以在 shell 加载时并不随之一起被加载到内存中**，而是在外部命令执行时才将其调入内存中。

例如命令`ls`，实际上`ls`本身就是个可执行二进制程序文件，通常放在系统的`/bin`文件夹下。Shell在执行该命令时，把`ls`的可执行二进制程序文件加载到内存中执行。最常见的Linux下Shell外部命令有：

```shell
ls
cat
more
grep
```

### 外部命令的调用方法

外部命令被调用时通常是通过Linux提供的`exec()`函数族完成。`exec()`函数族根据指定的文件名和相关参数找到可执行文件，如果找到了可执行文件，则该函数执行后是不会返回的，它会用可执行文件来取代当前进程的内容。即在进程内部执行一个可执行文件。

### 外部命令执行举例

加入要通过Shell执行以下外部命令：

```shell
ls -l -a
```

`ls`就是一个在Linux下已经存在的二进制可执行文件，`-l`是执行该文件时要传入的参数，那么就可以通过`exec`族函数中的一个`execvp`来执行该外部命令：

```c
char *argv[] = {"ls", "-l", "-a", NULL};
execvp("ls", argv);
```

`execvp()`函数第一个参数时要执行的命令文件名，第二个是一个指针数组，该数组第一个元素指向命令字符串，中间的指向参数字符串，结尾以`NULL`空指针结尾。

调用`execvp()`函数并找到对那个的`ls`可执行文件后，**当前的进程内容就会被`ls`可执行文件的内容所取代**，即原来的`execvp()`的那个进程已经不存在了，即使`execvp()`下面还有其它代码也不会被执行。`ls`命令执行完成后，进程就结束了。

## 创建子进程执行外部命令

因为前面说到Shell进程本身会**被外部命令的可执行程序所取代**，从而导致调用外部命令之后程序无法执行。那么我们可以通过创建要给子进程，在子进程执行外部命令，而原来的父进程继续执行Shell的其它代码。

### 父进程与子进程

每个进程都是由其他进程创建产生的，创建的叫父进程，被创建的叫子进程。

### `fork()`创建子进程

Linux提供了`fork()`API函数来创建新的子进程：

```c
#include <unistd.h> 
#include <sys/types.h>
#include <stdio.h>  

int main()
{
	int pid;//process identification，进程的唯一标识号
	
	pid=fork();
	
	if(0==pid)
	{
		printf("I am child process!\n");
	}
	else
	{
		printf("I am parent process!\n");
	}
	
	return 0;
}
```

编译执行上面代码，结果如下：

```c
I am child process!
I am parent process!
```

为什么两个`printf()`都被执行了呢？

- `fork()`会被执行两次，先在父进程里创建子进程的pid，然后在子进程中执行并返回0
- 根据返回值的不同就可以判断出是父进程还是子进程，从而执行不同的代码，虽然它们使用相同的代码段，但却可执行不同的代码。
- 子进程会继承父进程在执行`fork()`之前的所有东西，包括变量的定义，变量的赋值和其它。

看下面这个例子：

```c
#include <unistd.h> 
#include <sys/types.h>
#include <stdio.h>  

int main()
{
	int pid;//process identification，进程的唯一标识号
	int var=66;
	
	pid=fork();
	
	if(0==pid)
	{
		printf("I am child process! pid=%d var=%d\n",pid,var);
	}
	else
	{
		printf("I am parent process! pid=%d var=%d\n",pid,var);
	}
	
	return 0;
}

/** output
I am parent process! pid=112575 var=66
I am child process! pid=0 var=66
*/
```

- 父进程执行`fork()`之前定义了一个变量var并赋值为66，那么**子进程中也会有一个变量var，且`fork()`之后子进程中的var也是66**。
- 如果子进程对var进行修改，父进程中的var是不会变；同样父进程改变var也不会影响子进程中的。虽然名字一样，但它们对应的内存空间不同。
- 父进程和子进程的内存空间完全独立，子进程只是把父进程的内存空间的内容复制一份，所以子进程内存空间的初始状态是和父进程执行`fork()`的时候完全一样。
- 所以在`fork()`后，父进程干的所有事都跟子进程无关，子进程干的所有事也和父进程无关。

![fork原理](https://www.tomorrow.wiki/wp-content/uploads/2018/06/9908ab342e8ace70706607d2ffb16ba2.png)

### 在子进程中执行外部命令

所以只需要把`exec`族函数的调用放到子进程要执行的代码里就可以了，子进程执行外部命令，父进程等待外部命令执行完后，接着等待下一个命令的输入：

```c
pid=fork();

if(0==pid)//execute command in child process
{
	if(0!=execvp(cmd[0],cmd))//execvp 如果找不到对应的可执行文件就会返回非 0 的错误代码
		printf("No such command!\n");
	exit(EXIT_SUCCESS);//结束当前进程
}
else//parent process 
{
	//其他代码，例如等待命令执行完，然后重新执行新的命令
}
```

这样就实现了重复执行内建命令和外部命令，但如果要执行含有管道和重定向的一些复杂命令则需要加更多东西。

## 管道与重定向

### 管道

#### 管道定义

管道就是一个进程与另一个进程之间通信的信道，通常是用来把一个进程的输出通过管道连接到另一个进程的输入。它是**半双工**运作的，想要同时双向传输需使用两个管道。管道又分为**匿名管道**和**命名管道**，而**shell中使用的是匿名管道**。

例如命令`ls | grep main.c`使用管道来连接两条命令执行，让我们快速知道当前目录下是否有main.c文件。**管道的本质是内存中的缓冲区，可看作是打开到内存中的文件**。所以需要使用两个文件描述符来索引它，一个表示读端，一个表示写端。且规定：**数据只能从读端读取，只能往写端写入**。

#### 创建管道

使用函数`pipe()`可创建匿名管道，需要包含头文件`unistd.h`：

```c
int fd[2];
pipi(fd);
```

首先创建一个2个元素的整型数组，然后将其作为`pipe()`的参数，`pipe()`执行成功后，**数组元素`fd[0]`就会变为创建管道的读端文件描述符，`fd[1]`就会变为写端的文件描述符**。管道创建成功。

#### 把管道作为标准输入输出

管道创建成功后就可以直接使用`read()`和`write()`函数对管道进行数据读写。而Shell中都是使用**标准输入输出**对管道进行读写：

```c
int fd[2];
pipe(fd);  // 创建管道
pid=fork();

if(0==pid)//execute next command in child process
{
	dup2(fd[0],0);//redirect standard input to pipe(read)
	close(fd[0]);
	close(fd[1]);

	if(0!=execvp(cmd0[0],cmd0))
		printf("No such command!\n");
	exit(EXIT_SUCCESS);
}
else//execute current command in current process 
{
	dup2(fd[1],1);//redirect standard output to pipe(write)
	close(fd[0]);
	close(fd[1]);

	if(0!=execvp(cmd1[0],cmd1))
		printf("No such command!\n");

	exit(EXIT_SUCCESS);
}
```

首先创建一个管道，然后创建子进程，子进程会继承这个管道，**且父进程和子进程操作的是同一个管道**（管道的继承与普通变量不同）。如果希望在子进程中执行管道的读端程序例如`ls | grep main.c`中的`grep main.c`；在父进程中执行管道的写端程序，例如`ls | grep main.c`中的`ls`，那么在子进程中，先调用`dup2(fd[0], 0)`；此函数就是将标准输入的文件描述符0指向了管道的读端。

### 教程

[教程](https://www.tomorrow.wiki/archives/956)