# 模拟 Unix 文件系统

额，最近本人找到了自己高中时用的u盘，发现里面的数据全都变成乱码了，不知为何！

送去数据修复又甚感浪费钱，毕竟也是现在用不上的纪念性文件了。

为了眼前使用，只好格式化之。。。

但是本人突然有个歪门邪道的想法：整个盘数据一起乱码，有大概率是由于u盘文件系统损坏，这也是为什么格式化后能重新使用（格式化差不多是先全部归零，然后注入一个文件系统）。有些数据可能还是完好，只是操作系统无法通过文件系统读取而已。如果能掌握这个文件系统的原理，自己把文件系统修好，没损坏的数据不就回来了么！

本人深感绝妙，不由神魂飘飘。沾沾自喜时，不忘分享灵感，启发各位，有备无患！故备此项目，望与各位共同操练操练。

我们需要面对的是一整块 16MB 的二进制文件，通过空间管理大师的实力，实现一个真能用文件系统（的一部分）！

## 介绍环节

正如楔子所讲，操作系统依赖文件系统来区别自己存放的数据是什么、属于谁，文件系统便是数据管理者。

我们要实现的是 ext2 文件系统，其原型是 1984 年提出的 [BSD Fast File System](https://people.eecs.berkeley.edu/~brewer/cs262/FFS.pdf)。不过说实话这篇论文太原理了些，下面还是简要介绍一下。

![图](https://github.com/T-Eric/File-System-for-4-Decades-Ago/blob/main/pics/Snipaste_2024-09-16_20-53-07.png)

这张图的第二、三行展示了 ext2 的结构，第一行实际上说明， Linux 下每个分区的文件系统相互独立，即不在我们实现的范畴。

我们要实现的是一个分区，即引导块和一系列块组；而引导块 (Boot Sector) 存储的是硬盘分区信息和启动信息，属于第一行及以上层次的系统才会使用的范畴，因此在实现文件系统的任务中，不考虑也是可以的。结果发现最后我们最少只需要实现 16MB 的**包含两个块组**（Block Group）的数据结构就可以了。

------

一个块组中的成员及其作用如下：

- 超级块（SuperBlock）包含整个文件系统的重要信息，只在第0个块组有就行，但一般**选择让每个块组保留一份拷贝**。其保存的内容有点像在文件资源管理器看到的 C 盘属性。

  至少要保存：

  - 文件系统名
  - blocks 和 inode 总数量
  - 单 block 和单 inode 的大小
  - 最近一次修改的时间（`tips: time_t = long `）

- 组描述符（Group Descriptor）描述块组及其成员在整个磁盘中的位置信息，用于在读写块时精准定位、表明大小。同样，在每个块组中保留一份拷贝，操作系统会依赖它进行系统一致性检查。

  至少要保存：

  - 空闲的 blocks 和 inode 数量

  - 块组在整个磁盘中开始与结束位置的 block 编号
  - 每个成员（superblock，自己，bitmap，data blocks等，看你实现）的开始与结束位置

- 数据块位图（Block Bitmap）记录块组中各个块的使用情况。一个块被用到了就标1，否则就是0，十分简单吧？

- 索引节点位图（Inode Bitmap）与上面功能类似，记录的是使用与未使用的 inode (索引节点，index node) 编号。由于文件系统中 inode 

- 索引节点表（Inode Table）存放所有 inode。Inode 的职责是记录文件属性以及该文件的数据存放在哪些块内。

  根据 ext2 实际，inode 的大小为 128 bytes。但是我们可以随意设计。每个文件会且仅会占用一个 inode，系统读取文件时先找到 inode，再根据 inode 中的文件属性读写文件，最后修改 inode。

  别的花里胡哨的，权限之类，我们不记录，但至少要保存：

  - 文件的类型（在 baseline 只有 txt 和目录两种）

  - 文件在磁盘中的位置（所有保存了数据的块的位置）
  - 文件的实际大小（占用块数的整数倍即可）
  - 创建时间、最近读取时间、最近修改时间（ctime, atime, mtime）

- 数据块（Data Blocks），就是剩下的块的总称，真正存放文件内容的地方。建议每个块都拥有自己的编号，以便定位。

  一个文件大于块大小就会占用多个块，**一个块被占用后，即使有剩余容量也不能再被使用了**。

------

我们要管理的东西是文件和**目录（directory，即文件夹）**，目录也是一种文件，所以也存在对应的 inode。目录可以理解为存放了其他 inode 的地方，至少要保存：

- 所包含文件的文件名（并非路径）
- 文件对应的 inode 号码

推荐在目录中同时保存父目录（`..`）和本目录（`.`）的 inode 位置，十分方便，比如后面要实现的 `cd ..` 命令就需要返回父目录。

------

进一步需要解决的问题：

1. 一个 inode 如何存储大文件？不知大家怎么实现，但一定会用 inode 指向对应的数据块（即储存数据块编号）。在 ext2 的标准实现中用 15 个指针指向文件的实际数据块：

   - 前 12 个指针直接指向保存了文件数据的数据块
   - 单级间接指针指向一个“间接块”，这个特殊数据块里面存的全是指向真正数据块的指针
   - 双级间接指针指向的“间接块”里面存的全是单级间接指针
   - 三级则指向双级，后面不必多说

   假设你的设计是每个数据块（我的是 512 byte）用第一个 byte 存自己的属性（各个 bit 显示自己是数据块还是几级指针）（实际上完全可以不用），那么一个 inode 能管理 
   $$
   \begin{align}
   12*511+511^2+511^3+511^4&=65153{\ \rm GB}\\
   &=63{\ \rm TB}
   \end{align}
   $$
   当然，实际上文件大小用 byte 表示，保存在 inode 里是有位数限制的，尽管采取了特殊标识法，根据块大小，最多也只能支持 2TB 到 32 TB 的单个文件大小。

   **不过，我们的虚拟磁盘总大小就 16M，基本没有用二级甚至三级指针的机会，这种设计有点非必要**，看各位自己如何实现吧。

   说白了，用的是类似于多级数组的存法。你需要约定一套方法，判断读写文件时按照什么样的顺序去读写各个数据块。

2. 我们需要实现一些基于文件路径的操作。如何得知自己当前在哪个目录，并基于此做文件操作？

3. 如何尽量紧密地分配数据块、正确地释放数据块？Ext2 自带碎片整理功能，“紧密”的事放在 bonus 内容到时候再说，你现在只需思考如何做到正确。

------

鄙人不善表述。如果不甚明了，或者由于对具体实现方式不太明白而不敢下手、迟迟不愿开始，可以找一些博客或去搜索知乎增进了解。理论推荐[这个](https://jingxiang-zhang.github.io/cn/2019-12-30-OS-Ext2)，逐个剖析数据结构推荐[这个](https://blog.csdn.net/zyqash/article/details/142249316)，实践推荐[这个](https://www.jianshu.com/p/3355a35e7e0a)。对于新手，可能需要了解一下文件 IO 相关知识，能做到移动文件指针定点读写就足够完成任务了。

## 基本要求

下面是一些小要求。

我闲着无聊，自己写了一个项目框架，未经测试，欢迎大家借鉴。更欢迎大家在遵循下列基本要求的前提下，使用自己的框架。

要求：

- **请不要把东西都存到内存里！**Baseline 阶段，内存中不能存任何当前用不到的东西，用多少读写多少，尽管将来可能用到。我们会检查你的内存使用。

- **最小盘块大小 256 byte。最大盘块大小 4 KB。**我用的是 512 byte，但事实上 256 byte 更方便各位将盘内存的数据用数字打印出来进行调试（一行64个数字嘛）

  比较旧式的传统机械硬盘的最小物理储存单位一般是 512 byte，但是我们要管理的盘子很小，也没必要。**盘块（扇区）作为硬盘的基本单位，硬盘读写文件时必须整块读取或写入其所在的盘块。**因此，各种文件系统的最小操作单位一般是盘块大小的 $2^n$ 倍（因此在你自己的框架中，选用哪一种盘块大小都是可以的）。最小操作单位越大，读大文件需要的读取次数越少，但实现也相对复杂。

  因此，各位在实现时需要慎重规划每个结构的占用空间，精打细算是很必要的。比较简单的做法是，将一堆 inode 或 superblock 等同类事物塞进一个总大小为 512 byte 的类中，多余的空间填充好，然后以这个类为单位进行读写。

- 为了简便和直观，所有文件都是文本文件。当然也可以不是，只要求能做到按字节读写。

- 关于文件路径，只有一点需要注意：**以 `/` 开头表示绝对路径（即从根目录开始的路径），否则都表示相对路径（即从当前所在目录开始）。**你只需要在实现 `cp` 这个指令时注意这一点。

- 最终的交互界面应当是类似于命令行或 Shell 的无限循环等待输入、一有输入立刻执行的形式。需要支持以下基本命令（后接使用方法）：

  - `format`，对整个虚拟磁盘进行格式化，注入文件系统。也要支持在使用过程中格式化。

  - `touch/rm`，**在当前目录下**创建空文件 / 删除文件。如 `touch tsuyogaru.txt`。

    二者都不需要支持类似 `touch a/b/c.txt` 的多级子目录操作。

  - `ls`，输出当前目录下的所有目录名和文件名，按字典序排序。直接 `ls`。

  - `mkdir/rmdir`，**在当前目录下**创建空目录 / 删除目录。如 `mkdir xuexiziliao`。

  - `cd`，转换当前目录，只需支持到父目录和到任意下一级子目录的转换。如 `cd newdir`。

  - `read`，在当前目录下读取一个文件，将其内容转化为 ascii 标准文字输出。如 `read sodayo.txt`。

  - `write`，将当前目录下的一个文件全部修改为新输入的内容。使用方法为：

    - `> write`，进入输入模式。
    - 输入希望更改进去的文本。
    - 在某一次回车后的新一行，输入 `:exit` 并回车，退出文本输入模式，将之前输入的文本作为数据保存到文件中（当然要忽略 `:exit` 了）

  - `cp`，（有点难），复制**当前目录**中的**一个文件或一个目录及其下所有内容**到任意目录下。如 `cp a.txt b` 将本目录下的 `a.txt` 复制到本目录下的 `b` 文件夹里，而 `cp a /b` 则会将本目录下的 `a` 文件夹及其下所有内容复制到根目录。

- 关于报错信息，以下最基本的情况，请报错并跳过这条指令，怎样报错都可以：

  - 操作不存在的对象，如 `rm/rmdir/ls/cd/read/write/rename`。
  - 创建同名同种类对象，如 `touch/mkdir`。当然，**目录和文件可以同名**。

## 数据结构设计提示（或许待完善

提醒初次接触这类文件读写占比很重的项目的同学，`string`, `vector` 等数据结构是不能随意使用的！由于其是变大小的，写入文件时可能由于超长覆盖掉后面的数据。请用固定长度数组替代之。

## Bonus

Bonus 没有强制要求满足和测试什么，欢迎大家了解和尝试。唯一的评分标准是，你需要向我展示你做到了，然后我会来检查你的仓库代码，或者随便让你做一做。最重要的是，你要展示你做完了，**且有对应效果**。**我会一定程度根据你完成的任务量打分**。

- 其他文件系统。

  之前的文件系统是 Linux 文件系统典型，的树状结构，常见的还有旧 Windows 和u盘上常见的 FAT(16, 32) 和新 Windows 常用的 NTFS 文件系统，各自使用 FAT 表和 MFT 表管理文件，富有特色。**以全新的文件系统原理重新管理一块 20M 的盘子，并完成 baseline 中要求的全部任务**，即可得到满分。

  如果不想做新任务，也可以直接对你的系统进行版本升级！就目前而言，Linux 操作系统中常见的文件系统是 ext(2,3,4)，最常见是 ext4 (Fourth Extended Filesystem)，而我们当前实现的文件系统接近 ext2。简单说，三代支持记录所有文件层面更改的日志系统，要能支持查询和撤销，四代主要特色是支持更大的单文件和单卷大小。**实现到 ext4 特性（包括 ext3 特性）可以得到满分**，实现到 ext3 则能得到一半分数。

  当然还有很多其他的，随意选择吧！

  如果你选择实现 ext 科技树，**请（必须）做下面任务**：

  - **Link 功能。**实际上一个文件有唯一的索引节点号与之对应，对于一个索引节点号，却可以有多个文件名与之对应。因此，在磁盘上的同一个文件可以通过不同的路径去访问它。

    请了解并实现 Link 功能。

- 磁盘碎片整理。

  想象先创建 1000 个文件，每个占用 1 个数据块。然后删掉编号为偶数的 500 个文件。此时创建一个占用 500 个块的文件，为了分配的紧密性，就需要至少用掉零散的 500 个块，这些零散的块使用，便是磁盘碎片。

  我们知道，磁盘需要物理读写，磁头的移动距离（块的物理距离）影响磁盘读写性能，那么**大文件肯定是尽量把磁盘块放在一起比较好**。尽管涉及大量数据转移，但也为后面很长一段时间的使用效率保驾护航了，属于磨刀不误砍柴工。

  请你探索碎片整理的方法，做这样一个工作。需要你提供一个指令，输入之以手动碎片整理。如果能做到自动检测离谱的碎片情况并整理，更是不错。

- 文本编辑器。这其实不属于模拟文件系统范畴的任务，有点越界了哈。

  在命令行中的文件编辑器，这不是 vim 嘛！实现类似的东西即可，即把内容显示出来，用打字的方式修改，保存。
