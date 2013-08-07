## 简介 ##

这个教程教你一些编写shellcode时有用的技巧，比如如何加载库，动态定位windows函数，定义和定位字符串常量，调用windows函数。让你开始写自己的shellcode。

部分内容参考自这里：
[http://www.vividmachines.com/shellcode/shellcode.html.](http://www.vividmachines.com/shellcode/shellcode.html)

## 目标 ##

这个shellcode的目标是创建一个Windows的对话框，包含一定制的信息。

## 我们需要调用什么函数？ ##

通过编程经验，我们知道要在Windows产生这样的对话框需要调用MessageBoxA函数。通过Google我们可以确定MessageBoxA函数包含在User32.dll中。

如果用上面教程提到的技术，可以用“findFunctionInDLL.sh”来确认函数包含在User32.dll当中，同时提供给我们这个函数的地址。下面就是这样做的：

    # ./findFunctionInDLL.sh MessageBoxA
    user32.dll
    MessageBoxA is located at 0x7e45058a in user32.dll

现在知道了MessageBoxA函数的地址，它包含在User32.dll中。然而不幸的是，如果我们把这个内存地址硬编码进shellcode，它只能运行在你现在的电脑上。对于我来说是Windows XP SP2.

我们换个方法，用一种新技术来动态定位MessageBoxA函数，而不是对内存地址硬编码。

## User32.dll加载了吗？ ##

在Windows中我们唯一可以确定加载的动态链接库是Kernel32.dll，但是我们不需要知道User32.dll是否已经加载。我们通常都认为它没有加载，除非你针对特定的一个Exploit编写的shellcode.

为了加载这个库我们可以调用LoadLibraryA函数，它在Kernel32.dll中。所以我们需要定位LoadLibraryA地址。这可以通过使用“findFunctionInDLL.sh”或者用上面的方法；然而既然我们已经知道了要用的库和函数，一个更加快捷的方法是直接使用arwin，像这面这样：

    # ./arwin Kernel32.dll LoadLibraryA
    arwin - win32 address resolution program - by steve hanna - v.01
    LoadLibraryA is located at 0x7c801d77 in Kernel32.dll

要注意的是，我们只需要动态定位MessageBoxA的地址，我们并不会清除所有的硬编码内存地址。在下面的教程中，我们会让你的shellcode运行在各种Windows平台上。

## 如何定位MessageBoxA的地址？ ##

到这里我们知道如何加载User32.dll，它包含着我们要调用的MessageBoxA函数。但是我们依然不知道MessageBoxA在内存中的地址。

可以使用Kernel32.dll中的GetProcAddress函数。把函数名传递给GetProcAddress函数，它会返回函数的地址。所以最开始的，我们要定位GetProcAddress的地址：

    # ./arwin Kernel32.dll GetProcAddress
    arwin - win32 address resolution program - by steve hanna - v.01
    GetProcAddress is located at 0x7c80adc0 in Kernel32.dll

## 如何防止破坏父进程？ ##

在上一节教程中，进程崩溃引发了一个核心存储（core dump）错误。为正常退出程序需要调用ExitProcess函数。为简单起见，我们使用arwin来枚举它在Kernel32.dll中的地址。可以这样做：

    # ./arwin Kernel32.dll ExitProcess
    arwin - win32 address resolution program - by steve hanna - v.01
    ExitProcess is located at 0x7c81ca82 in Kernel32.dll

目前为止，我们有了要写shellcode所需要的所有信息，因此你可以很容易理解下面的汇编代码，我只想点出一点使用的技术。

## 定义和定位字符串常量 ##

从上面的信息中，可以看到在我们的shellcode中需要三个字符串常量。它们是：

    'user32.dll'
    'MessageBoxA'
    'Hey'

我们把字符串常量“User32.dll”作为一个参数传递给LoadLibraryA函数，把“MessageBoxA”作为一个参数传递给GetProcAddress函数，把“Hey”作为参数传递给MessageBoxA函数，对话框可以显示“Hey”的信息。

下面的代码片段，后面演示了如何定义字符串常量，前面演示了如何定位这些字符串常量；

    +--------------- [snip] ---------------+
    ;Retrieve the address of the library name string set below.*
    	jmp short GetLibrary ;Jump to where our library string is located ("GetLibrary" label below)
    GetLibraryReturn:;Create a label we can call to return here.
    	pop ecx	;the "call" operation has pushed the return address onto the stack, which we have designed to point to our string - so pop the address of the library name string off the stack and into ecx.
    ;At this point, ecx points to our string.
    
    +--------------- [snip] ---------------+
    
    GetLibrary: ;Create the "GetLibrary" label where our library name string is located
    	call GetLibraryReturn ;"call" is like jump, but also pushes the next instruction address onto the stack. Since our string is defined immediately after this instruction, this is the address of our string.
    	db 'user32.dll' ;Write the raw bytes into the shellcode that represent our string.
    	db 0x00 ;Terminate our string with a null character.
    
    +--------------- [snip] ---------------+

## 编写shellcode ##

下面是msgbox.asm的代码，原始出自[http://www.vividmachines.com/shellcode/shellcode.html](http://www.vividmachines.com/shellcode/shellcode.html)，有一些细微的变化及注释。

在你机子上Cygwin环境中创建msgbox.asm文件，阅读完代码的注释，它们解释了每行的功能并介绍一些在编写shellcode过程中有用的技巧。记住把每个函数的地址替换你上面用arwin生成的。

    +--------------- Start msgbox.asm --------------+
    
    ;msgbox.asm
    
    [SECTION .text]
    
    BITS 32
    
    global _start
    
    _start:
    
    ;zero out the registers
    xor eax,eax
    xor ebx,ebx
    xor ecx,ecx
    xor edx,edx
    
    ;Retrieve the address of the library name string set below.
    jmp short GetLibrary
    GetLibraryReturn:
    pop ecx	;pop address of the Library string
    
    ;Pass library string as parameter to LoadLibraryA, and call LoadLibraryA
    mov ebx, 0x7c801d77	 ;LoadLibraryA(libraryname)
    push ecx	 ;push parameter to LoadLibraryA
    call ebx	 ;call LoadLibraryA - eax holds return value
    
    ;Retrieve the address of the function name string set below.
    jmp short FunctionName
    FunctionReturn:
    pop ecx	 ;pop address of the function string
    
    ;Pass function string as parameter to LoadLibraryA, and call LoadLibraryA
    push ecx	 ;push string as the second parameter
    push eax	 ;pass first parameter
    mov ebx, 0x7c80adc0	 ;GetProcAddress(hmodule,functionname)
    call ebx	 ;eax now holds address of MessageBoxA
    
    jmp short Message
    MessageReturn:
    pop ecx	 ;get the message string
    xor edx,edx	 ;clear edx value
    ;Push the parameters onto the stack:
    push edx	 ;MB_OK
    push ecx	 ;title
    push ecx	 ;message
    push edx	 ;NULL window handle
    call eax	 ;MessageBoxA(windowhandle,msg,title,type)
    
    ender:
    xor edx,edx	 ;empty edx out
    push eax	 ;move address of MessageBoxA onto stack
    mov eax, 0x7c81ca82 ;ExitProcess(exitcode);
    call eax	 ;exit cleanly so we don't crash parent
    
    GetLibrary: ;Define location and string constant "user32.dll"
    call GetLibraryReturn ;push address of next byte onto stack, and return to GetLibraryReturn
    db 'user32.dll';string constant
    db 0x00;terminate string with null
    
    FunctionName:;Define location and string constant "MessageBoxA"
    call FunctionReturn;push address of next byte onto stack, and return to FunctionReturn
    db 'MessageBoxA';string constant
    db 0x00;terminate string with null
    
    Message:;Define location and string constant "Hey"
    call MessageReturn;push address of next byte onto stack, and return to MessageReturn
    db 'Hey';string constant
    db 0x00;terminate string with null
    
    +--------------- End msgbox.asm --------------+

## 编译汇编代码 ##

现在用汇编写好了我们的shellcode，我们需要编译它。可以使用nsam汇编编译器，用下面的命令，这里msgbox.asm是源文件，msgbox.bin是编译生成的二进制文件：

	# nasm -f bin -o msgbox.bin msgbox.asm

## 提取shellcode ##

我们已经有了编译生成的二进制文件，接下来使用xxd工具生成shellcode.使用下面的xxd命令，输出如下：

    # xxd -i msgbox.bin
    unsigned char msgbox_bin[] = {
     0x31, 0xc0, 0x31, 0xdb, 0x31, 0xc9, 0x31, 0xd2, 0xeb, 0x2a, 0x59, 0xbb,
     0x77, 0x1d, 0x80, 0x7c, 0x51, 0xff, 0xd3, 0xeb, 0x2f, 0x59, 0x51, 0x50,
     0xbb, 0xc0, 0xad, 0x80, 0x7c, 0xff, 0xd3, 0xeb, 0x34, 0x59, 0x31, 0xd2,
     0x52, 0x51, 0x51, 0x52, 0xff, 0xd0, 0x31, 0xd2, 0x50, 0xb8, 0x82, 0xca,
     0x81, 0x7c, 0xff, 0xd0, 0xe8, 0xd1, 0xff, 0xff, 0xff, 0x75, 0x73, 0x65,
     0x72, 0x33, 0x32, 0x2e, 0x64, 0x6c, 0x6c, 0x00, 0xe8, 0xcc, 0xff, 0xff,
     0xff, 0x4d, 0x65, 0x73, 0x73, 0x61, 0x67, 0x65, 0x42, 0x6f, 0x78, 0x41,
     0x00, 0xe8, 0xc7, 0xff, 0xff, 0xff, 0x48, 0x65, 0x79, 0x00
    };
    unsigned int msgbox_bin_len = 94;

产生了一个可以在C程序中使用的字符数组。每个十六进制数字代码了shellcode中的一个字节。

接下来使用xxd-shellcode.sh脚本提取出原始shellcode，以便我们直接在shellcodetest.c程序中使用。

这个程序把"msgbox.bin"作为输入文件，通过下面的命令产生下面的输出。输出会自动被保存到“msgbox.shellcode”文件中。根据你使用的系统，这段shellcode代码会有不同。

	# ./xxd-shellcode.sh msgbox.bin
    \x31\xc0\x31\xdb\x31\xc9\x31\xd2\xeb\x2a\x59\xbb\x77\x1d\x80\x7c\x51\xff\xd3\xeb\x2f\x59\x51\x50\xbb\xc0\xad\x80\x7c\xff\xd3\xeb\x34\x59\x31\xd2\x52\x51\x51\x52\xff\xd0\x31\xd2\x50\xb8\x82\xca\x81\x7c\xff\xd0\xe8\xd1\xff\xff\xff\x75\x73\x65\x72\x33\x32\x2e\x64\x6c\x6c\x00\xe8\xcc\xff\xff\xff\x4d\x65\x73\x73\x61\x67\x65\x42\x6f\x78\x41\x00\xe8\xc7\xff\xff\xff\x48\x65\x79\x00

## 测试shellcode ##

我们将使用“shellcodetest.c”程序测试我们的shellcode.这一步的目标是把我们的shellcode插入到C程序当中，接着编译运行。这个程序被设计用来执行我们的shellcode.

在做这之前，我们把上面最后一步生成的shellcode插入到程序当中。把它放在“code[]”数组的引号之间，最后是下面这样：

    +----------------- Start updated shellcodetest.c -----------------+
    
    /*shellcodetest.c*/
    char code[] = "\x31\xc0\x31\xdb\x31\xc9\x31\xd2\xeb\x2a\x59\xbb\x77\x1d\x80\x7c\x51\xff\xd3\xeb\x2f\x59\x51\x50\xbb\xc0\xad\x80\x7c\xff\xd3\xeb\x34\x59\x31\xd2\x52\x51\x51\x52\xff\xd0\x31\xd2\x50\xb8\x82\xca\x81\x7c\xff\xd0\xe8\xd1\xff\xff\xff\x75\x73\x65\x72\x33\x32\x2e\x64\x6c\x6c\x00\xe8\xcc\xff\xff\xff\x4d\x65\x73\x73\x61\x67\x65\x42\x6f\x78\x41\x00\xe8\xc7\xff\xff\xff\x48\x65\x79\x00";
    int main(int argc, char **argv)
    {
    int (*func)();
    func = (int (*)()) code;
    (int)(*func)();
    }
    
    +----------------- End updated shellcodetest.c -----------------+

我们现在需要编译更新后的shellcodetest.c程序，让它来执行我们的shellcode.

可以使用下面的命令：

	# gcc -o shellcodetest shellcodetest.c

编译生成可执行文件“shellcodetest.exe”.

下面你可以执行包含着shellcode的程序，它显示一个包含“Hey”信息的对话框，接着正常退出而不会引发一个核心存储错误。如果发生了核心存储错误，可能是你硬编码地址不正确，检查你上面每个函数的arwin输出。

    # ./shellcodetest.exe
    (message box produced saying "Hey")
    (click "Ok" and it should exit cleanly)

## 祝贺你！ ##

你刚刚已经编写了一个shellcode，它加载Windows库，动态定位Windows函数，定义及定位字符串常量，调用Windows函数。

在编写你自己shellcode的道路上，你已经处在了有利的位置。然而，在上面的代码中我们依然有一些硬编码地址。在下面的教程中，我们将向你展示如何通过动态定位Kernel32.dll和GetProcAddress函数来去除这些硬编码。同时介绍如何用汇编语言编写函数以便我们重用自己的代码。