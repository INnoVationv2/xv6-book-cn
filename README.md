# XV6:一个简单的类Unix教学用操作系统

## 前言和致谢词

这是一份为操作系统课程准备的文本。通过研究一个名为xv6的内核来解释操作系统的主要概念。xv6以Dennis Ritchie和Ken Thompson的Unix Version 6 (v6)为蓝本。大致遵循了v6的结构和风格。用ANSI C实现，运行在RISC-V多核处理器上。

本文建议与xv6的源代码一起阅读，这种学习方法的灵感来自John Lions的《UNIX第六版评论》。你可以在这里https://pdos.csail.mit.edu/6.S081获得v6和xv6的在线学习资源，包括几个使用xv6的实验作业。

这篇文档主要在MIT的6.828和6.S081的操作系统课上使用。我们感谢这些课程的教师、助教和学生，他们都直接或间接地对xv6的做出了贡献。特别的，我们要感谢Adam Belay, Austin Clements, 和Nickolai Zeldovich。最后，我们要感谢那些给我们发邮件指出文本中的错误或提出改进建议的人：Abutalib Aghayev, Sebastian Boehm, Anton Burtsev, Raphael Carvalho, Tej Chajed, Rasit Eskicioglu, Color Fuzzy, Giuseppe, Tao Guo, Naoki Hayama, Robert Hilder-man, Wolfgang Keller, Austinman, Wolfgang Keller, Austin Liew, Pavan Maddamsetti, Jacek Masiulaniec, Michael McConville, m3hm00d, miguelgvieira, Mark Morrissey, Harry Pan, Askar Safin, Salman Shah, Adeodato Simó, Ruslan Savchenko, Pawel Szczurko, Warren Toomey, tyfkda, tzerbib, Xi Wang, and Zou Chang wei。

如果您发现错误或有改进建议，请发邮件给Frans Kaashoek 或 Robert Morris(kaashoek@csail.mit.edu,rtm@csail.mit.edu)。
