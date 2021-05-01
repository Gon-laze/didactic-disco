---
description: 系统调用生成
---

# user/usys.pl

`usys.pl`是一个Perl语言文件。也许你对Perl语言并没有深入了解，但根据语法规则可以大概了解它的作用：这个脚本文件将根据输入的字符名，通过`entry()`格式化生成文本文件。

实际上，`usys.pl`的输入为16个系统调用，输出内容则保存为`kernel/usys.S`。这个文件将通过`make`命令生成：

```text
....
```



