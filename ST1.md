##  Windows shellcode 和 Linux shellcode 有什么不同？ ##

([http://www.vividmachines.com/shellcode/shellcode.html](http://www.vividmachines.com/shellcode/shellcode.html))

Linux提供了直接和内核交互的接口（通过 int 0x80），在[http://www.informatik.htw-dresden.de/~beck/ASM/syscall_list.html](http://www.informatik.htw-dresden.de/~beck/ASM/syscall_list.html)可以找到所有Linux系统调用的清单。然而windows并没有直接和内核打交道的接口，它必须通过加载 DLL（动态链接库）中要执行函数的地址的方式与之交互。

这两个操作系统关键的不同在于windows不同的版本上函数的地址是不同的，而Linux的系统调用地址却不会变。windows程序员可以随意修改内核不用担心带来麻烦，而Linux的内核级函数都有固定编号，如果它们改变的话，成千上万的程序员会发狂。（也会带来许多不能用的代码）


    
## 在Windows下呢，我如何定位DLL中的函数地址? 每次版本的升级这些地址改变了怎么破? ##

([http://www.vividmachines.com/shellcode/shellcode.html](http://www.vividmachines.com/shellcode/shellcode.html))

有多种定位在shellcode需要使用的函数的方法。本教程讨论两种函数定位方法，你可以在运行时定位函数地址或者使用硬编码地址。我们主要讨论的是硬编码的方式，唯一必须映射到shellcode地址空间的DLL是kernel32.dll。这个DLL包含LoadLibrary和GetProcAddress函数，这两个函数可以把任何函数映射到漏洞进程的空间。然而这种方法有个问题，每个windows的版本（包括升级包、补丁等）都会使地址的偏移改变，所以用这种方法你的shellcode只能运行在特定版本的windows上面。

动态定位函数地址的方法将会在后面的教程中介绍。

## 配置我们的开发环境！ ##

首先我们要关注的是创建windows的汇编开发环境；Linux用来写汇编和开发shellcode真的非常方便，正因此我们把Cygwin作为shellcode的开发平台，它亦能够访问windows DLL。

从[http://www.cygwin.com/setup.exe](http://www.cygwin.com/setup.exe)下载并运行安装程序。

安装过程会询问你需要选择的安装包，选择下面的：

        - Devel->binutils (contains ld, as, objdump)
        - Devel->gcc
        - Devel->make
        - Devel->nasm
        - Devel->gdb
        - Editors->hexedit
        - Editors->vim
        - Net->netcat
        - System->util-linux

Cygwin环境配置完以后，下载下面的工具。一些是为了方便开发我自己写的脚本，最后两个是外部的资源。你可能需要把它们保存进你的Cygwin环境，所以这样拷贝它们C:\cygwin\home\Administrator\shellcode\，Administrator是你的用户名，我已经创建好了一个“shellcode directory”作为我们的工作目录。

        - xxd-shellcode.sh

解析xxd输出来提取原始shellcode

[http://www.projectshellcode.com/downloads/xxd-shellcode.sh](http://www.projectshellcode.com/downloads/xxd-shellcode.sh)

        - shellcode-compiler.sh

自动编译汇编代码，提取原始shellcode，创建原始shellcode的UNICODE编码版本，把编码后的shellcode注入到一个“Templates Exploit”（ms07-004）进行测试，创建一个包含你shellcode的测试程序，编译成可执行程序，非常方便。

[http://www.projectshellcode.com/downloads/shellcode-compiler.sh](http://www.projectshellcode.com/downloads/shellcode-compiler.sh)

或者

[http://www.projectshellcode.com/downloads/shellcode-compiler.zip](http://www.projectshellcode.com/downloads/shellcode-compiler.zip)

        - findFunctionInDLL.sh

查看系统中哪个DLLs包含一个指定的windows函数。

[http://www.projectshellcode.com/downloads/findFunctionInDLL.sh](http://www.projectshellcode.com/downloads/findFunctionInDLL.sh)

        - arwin.c

Win32 DLL地址解析程序

[http://www.vividmachines.com/shellcode/arwin.c](http://www.vividmachines.com/shellcode/arwin.c)

        - shellcodetest.c

[http://www.vividmachines.com/shellcode/shellcodetest.c](http://www.vividmachines.com/shellcode/shellcodetest.c)

从开始菜单打开一个bash shell，进入你的“shellcode directory”例如：

        cd /home/Administrator/shellcode/

你现在需要用下面的命令编译arwin.c

        gcc -o arwin arwin.c

现在你已经可以运行./arwin来显示一些有用的信息。

这个阶段我们还不需要编译shellcodetest.c，当我们写好自己的shellcode后，那时候我们把shellcode放到shellcodetest.c中并编译。这让我们通过运行shellcodetest来执行我们的shellcode。

Metasploit Framework 对编写shellcode来说是非常优秀的资源，在写这个教程的时候，Meatsploit正要发布3.3版本的Framework ，在windows的Cygwin环境下运行。开发版本可以从下面的链接中下载（如果链接已经不能用，请告知我）：

        - Metasploit Framework 3.3-dev

[http://www.projectshellcode.com/downloads/framework-3.3-dev.exe](http://www.projectshellcode.com/downloads/framework-3.3-dev.exe)

这个版本的Framework安装在C:\msf3\并有自己专门的Cygwin环境，你可以通过shell.bat加载一个shell。

另外也有一些其它优秀的windows程序我们会在教程4中使用，下载下面的工具：

        - OllyDbg 1.10 (优秀的Windows调试工具)

[http://www.ollydbg.de/odbg110.zip](http://www.ollydbg.de/odbg110.zip)

        - lcc-win32 (免费的Windows C编译器)

[http://www.q-software-solutions.de/pub/lccwin32.exe](http://www.q-software-solutions.de/pub/lccwin32.exe)

现在你已经准备好编写你第一个shellcode了！
