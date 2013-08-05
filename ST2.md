## 简介 ##
这个教程介绍编写shellcode的基础知识，包括在windows DLLs中定位函数地址，简单的汇编编写，如何编译ASM代码以及如何运行你的shellcode来观察它是否工作。我们会写一个最简单的shellcode：sleep5秒钟后退出。

教程上的一些参考信息来自这里

[http://www.vividmachines.com/shellcode/shellcode.html](http://www.vividmachines.com/shellcode/shellcode.html)

如何在Windows DLLs中定位函数地址

在windows中用作程序休眠的函数是“Sleep”。那么要使用这个函数我们要怎么做？我汇编里我们使用“call 0xXXXXXXXX”的结构，XXXXXXXX就是函数在内在中的地址，所以我们要知道这个函数加载在哪里。

这个可以利用我们在第1个教程中下载编译的“arwin”程序完成，如果在你的“shellcode directory”目录中运行./arwin，可以得到下面的信息：

    $ ./arwin.exe
    arwin - win32 address resolution program - by steve hanna - v.01
    ./arwin <Library Name> <Function Name>

所以要使用arwin首先我们要知道要用的函数存在于哪个DLL，这通常在Kernel32.dll或User32.dll或ws2_32.dll中；然后这取决于你的shellcode要完成什么样的功能，我写了下面的脚本“ findFunctionInDLL.sh”，是对arwin的一个封装，它会搜索你系统上的DLLs并找出函数包含在哪个DLL中。

    +----------------- Start findFunctionInDLL.sh -----------------+
    
    #!/bin/bash
    
    if [ $# -ne 1 ]
    then
    printf "\n\tUsage: $0 functionname\n\n"
    exit
    fi
    
    functionname=$1
    searchDir="/cygdrive/c/WINDOWS/system32"
    
    arwin_exe="`pwd`/arwin.exe"
    cd $searchDir
    
    ls -1d *.dll | grep -v gui | while read dll
    do
    printf "\r ";
    printf "\r$dll";
    count=0
    count=`$arwin_exe $dll $functionname | grep -c "is located at"`
    if [ $count -ne 0 ]
    then
    printf "\n";
    $arwin_exe $dll $functionname | grep "is located at"
    printf "\n";
    fi
    done
    printf "\r ";
    
    +----------------- End findFunctionInDLL.sh -----------------+

把 findFunctionInDLL.sh拷贝到你的系统，把它改为可执行权限“chmod 755 findFunctionInDLL.sh”。你要保证在Cygwin环境中自己的windows系统目录挂载到了/cygdrive/c/WINDOWS/system32。

我们现在可以用这个脚本找出我们需要把哪个DLL传递给arwin。你会发现脚本遍历系统上的DLLs，如果找到匹配的函数和地址它会通知你：
    
    # ./findFunctionInDLL.sh Sleep
    
    kernel32.dll
    Sleep is located at 0x7c802442 in kernel32.dll

你得到的地址可能跟我的不同，这取决于你的操作系统和版本。我使用的是Windows XP SP2.这意味着你的shellcode只能运行在特定操作系统的特定版本之上，因为我们用的是函数在内存中地址的硬编码。有更多高级的技术可以用来动态加载kernel32.dll和函数地址；但这会在后面的教程中讲到。 

这会花费一些时间，如果你已经知道在函数在哪个DLL中，你可以直接使用arwin：

    # ./arwin.exe Kernel32.dll Sleep
    arwin - win32 address resolution program - by steve hanna - v.01
    Sleep is located at 0x7c802442 in Kernel32.dll

## 汇编代码 ##
下面是sleep.asm的代码，原始出处是http://www.vividmachines.com/shellcode/shellcode.html加了少量的修改和注释。

在你Cygwin中“shellcode directory”中创建sleep.asm文件，写入下面代码。一定要阅读代码的注释，它解释了每一行语句做什么，为后面shellcode的开发提供了有用的小技巧。记住把“Sleep”的地址替换成你自己系统的。

    +----------------- Start sleep.asm -----------------+
    
    ;sleep.asm
    [SECTION .text]
    
    ; set the code to be 32-bit
    ; Tip: If you don't have this line in more complex shellcode,
    ;the resulting instructions may end up being different to
    ;what you were expecting.
    BITS 32
    
    global _start
    
    _start:
    ; clear the eax register
    ; Tip: xor is great for zeroing out registers to clear previous values.
    xor eax,eax
    
    ; move address of Sleep to ebx that we gained from "./arwin.exe Kernel32.dll Sleep"
    mov ebx, 0x7c802442
    
    ; pause for 5000ms by putting 5000 into ax (8 bit eax register)
    ; Tip: ax is the lower half of eax. Using ax when possible reduces
    ;the instruction size, and therefore the shellcode size.
    mov ax, 5000
    
    ; push eax onto the stack as the first parameter to the Sleep function.
    ; Tip: When functions are called, the parameters are pulled from the stack.
    push eax
    
    ; call the address of Sleep(ms) located in ebx
    ; Tip: Sleep has one parameter and will pull this from the stack.
    call ebx
    
    +----------------- End sleep.asm -----------------+

## 编译汇编代码 ##
现在已经有了汇编写的shellcode代码，我们需要编译它。这可以用nasm汇编编译器，使用下面的命令编译（sleep.asm是你的汇编源文件，sleep.bin是编译输出的二进制文件 ）：

    # nasm -f bin -o sleep.bin sleep.asm
## 
得到shellcode ##

现在你已经有了一个编译好的二进制文件，可以用xxd工具生成我们的shellcode。这可以用下面的xxd命令完成，得到下面的输出。

    # xxd -i sleep.bin
    unsigned char sleep_bin[] = {
     0x31, 0xc0, 0xbb, 0x42, 0x24, 0x80, 0x7c, 0x66, 0xb8, 0x88, 0x13, 0x50,
     0xff, 0xd3
    };
    unsigned int sleep_bin_len = 14;

它创建了一个我们可以在c程序中使用的字符数组。输出中的每个十六进制数字（0xXX）代表了shellcode中的一个字节。 

我们是用这些输出来创建我们的shellcode吗？我们会用下面的脚本来分离出原始shellcode，供我们在“shellcodetest.c”程序中直接使用。把下面的代码拷贝到你机器的“shellcode directory”文件夹，修改它的权限"chmod 755 xxd-shellcode.sh"。

    +----------------- Start xxd-shellcode.sh -----------------+
    
    #!/bin/bash
    if [ $# -ne 1 ]
    then
    printf "\n\tUsage: $0 filename.bin\n\n"
    exit
    fi
    
    filename=`echo $1 | sed s/"\.bin$"//`
    rm -f $filename.shellcode
    
    for i in `xxd -i $filename.bin | grep , | sed s/" "/" "/ | sed s/","/""/g | sed s/"0x"/"\\\\x"/g`
    do
    echo -n "\\$i" >> $filename.shellcode
    echo -n "\\$i"
    done
    echo
    
    +----------------- End xxd-shellcode.sh -----------------+

这个程序使用输入文件 “sleep.bin”，所以运行下面命令，会产生下面的输出。这个输出会被自动保存到“sleep.shellcode”文件中。在每个系统或版本上这会有轻微的差别，因为“Sleep”地址存在差异。
    
    # ./xxd-shellcode.sh sleep.bin
    \x31\xc0\xbb\x42\x24\x80\x7c\x66\xb8\x88\x13\x50\xff\xd3

## 测试shellcode ##

我们将使用“shellcodetest.c”程序来测试我们的shellcode。这一步的目的是把我们的shellcode插入到一个可编译和运行的c程序当中。这个程序被设计用来执行我们的shellcode。

在这之前，你需要把最后一步生成的shellcode插入到这个程序中。把它放到code[]数组的引号中间。最后差不多是这样子的：

    +----------------- Start updated shellcodetest.c -----------------+
    
    /*shellcodetest.c*/
    char code[] = "\x31\xc0\xbb\x42\x24\x80\x7c\x66\xb8\x88\x13\x50\xff\xd3";
    int main(int argc, char **argv)
    {
    int (*func)();
    func = (int (*)()) code;
    (int)(*func)();
    }
    
    +----------------- End updated shellcodetest.c -----------------+

我们现在需要编译更新后的shellcodetest.c程序来执行我们的shellcode。用下面的命令： 

	`# gcc -o shellcodetest shellcodetest.c`

编译生成可执行程序“shellcodetest.exe”。

现在你可以运行这个程序启动你的shellcode。这个shellcode被设计成简单的睡眠5秒钟就像你看到的那样，接着它会退出。——可能会产生核心存储错误（core dump），但在此阶段我们不关心这个。

    # ./shellcodetest.exe
    (sleeps for 5 seconds)
    (then exits - and may core dump)

祝贺你！你刚刚已经编写、编译、提取、格式化和测试了你的第一个shellcode。

接下来让我们做一些我们能真正看的见的东西！