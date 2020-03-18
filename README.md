# CS:APP Labs

这个项目是《CS:APP》第三版的相关实验解答和笔记，实验的所有 lab 已经上传在 [labs](./labs/) 下，来源是 [Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html).

## 目录：

### [labs](./labs/)

 包含所有的 lab 文件，以及 CMU 给的参考文档，也包含我写的解答文件，我的实验环境是 Ubuntu 16.04 amd-64，其中 [source](./labs) 保存了所有 lab 的原文件；

### [notes](./notes/)

 是我写的笔记：

1. [Data Lab 笔记](./notes/datalab.md)

- 涉及了位运算，补码和浮点数的位级表示

- 都是 C 语言程序设计题，但是只能是straightline的代码（也就是，不能有循环和选择语句）。

- 只能使用有限数量、部分的运算符。具体来说就是 `!~&^|+<<>>` 这8种

2. [Bomb Lab 笔记](./notes/bomb.md)

- 拆除二进制炸弹，每个炸弹考察了机器级程序语言的不同方面

- 你需要理解汇编代码的的行为和作用，进而设法推断出拆除炸弹所需的目标字符串

- 你需要使用gdb调试器和objdump
  

3. [Attack Lab 笔记](./notes/attack.md)

 - 这个 lab 主要涉及了栈随机化，不可执行等栈保护的方法和使栈溢出、 ROP 攻击等内容。
 
 - 能更深入的理解x86-64机器代码的栈和参数传递机制
 
 - 能更熟悉的使用gdb和objdump这样的调试工具
 

4. [Shell Lab 笔记](./notes/shelllab.md)

- 实现一个简单的支持作业控制的Unix shell 程序

- 使你熟悉进程控制和信号处理的概念
