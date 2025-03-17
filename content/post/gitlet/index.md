+++

date = '2025-03-10T23:21:00+08:00'
draft = false
title = 'gitlet'
description = "一个自制的VCS"
categories = [
    "VCS",
    "java",
]
image = "branching-illustration.png"

+++

## 介绍

本项目参考[cs61b](https://sp21.datastructur.es/index.html)的[proj2](https://sp21.datastructur.es/materials/proj/proj2/proj2)的设计思路

在本项目中，我实现了一个[版本控制系统](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control)，它模仿了流行系统Git的一些基本功能。然而，我的版本较小且更简单，因此将其命名为Gitlet.

我的实现放在[这里](https://github.com/Spiceses/gitlet), 这个项目是在大二下学期学习数据结构(cs61b)时完成的, 由于当时尚未系统学习过软件工程, 所以并没有特别注重代码的安全性, 可读性与可扩展性(甚至类与方法的spec都没有), 希望读者谅解.

技术栈: java, VCS(版本控制系统), 持久化存储

PS: 突然想到, 我实现gitlet时, 也使用了git维护代码版本, 这就是世界上只有一个人需要手写汇编代码实现编译器的原理吗

## 抽象

在开始之前, 我想先介绍一个重要的抽象, 我之后就是通过一步一步实现这些抽象来完成所需功能的

* **提交(commit)**: 保存整个文件目录的内容, 称为提交（committing），保存的内容本身称为提交记录（commits）
* **暂存区(staging area)**: 暂存区用于存储有关下次提交的内容的信息
* **已跟踪文件(tracked files)**: 已经被版本控制系统“记录”并纳入版本管理的文件
* **索引(index)**: 所有已跟踪文件的最新版本

## 功能

版本控制系统本质上是一个用于相关文件集合的备份系统。Gitlet支持的主要功能包括：

- 保存整个文件目录的内容, 即commit。
- 恢复一个或多个文件或整个提交记录的版本。在Gitlet中，这称为检出（checking out）这些文件或提交记录。
- 查看备份的历史记录。在Gitlet中，您可以通过日志（log）查看此历史记录。
- 维护相关的提交记录序列，称为分支（branches）。
- 将一个分支中所做的更改合并(merge)到另一个分支中。

## 数据结构

### Blob

#### 描述：

- `Blob` 是 Gitlet 中用于表示文件内容的数据结构。
- 它存储的是文件内容的快照，而不是文件本身。每个文件的内容通过 SHA-1 哈希生成唯一的标识符，用于追踪文件版本。

#### 数据成员：

- `CONTENT`: 文件的内容，以字符串形式存储。
- `BLOB_DIR`: 保存所有 Blob 对象的目录。

#### 方法：

- `name()`: 返回该 `Blob` 对象的唯一标识符（通过序列化对象计算 SHA-1）。
- `save()`: 将 `Blob` 对象序列化后存储到磁盘中。
- `find()`: 根据文件名在存储目录中查找对应的 `Blob`。

#### 功能：

Blob 是文件版本的核心构件，每次文件内容发生变化时，新的 `Blob` 会被创建并保存。这种设计使得同一文件的不同版本可以通过其唯一标识符区分。

### Commit

#### 描述：

- `Commit` 是 Gitlet 中的核心概念，用于表示项目的一个版本。
- 它记录了文件快照（`Blob`）、提交信息、时间戳、父提交等内容。

#### 数据成员：

- `message`: 提交消息，描述提交的目的或修改内容。
- `timestamp`: 提交的时间戳。
- `track`: 一个 `Map<File, String>`，用于追踪当前提交的文件和对应的 `Blob` 标识符。
- `parent`: 父提交的标识符（SHA-1）。
- `COMMIT_DIR`: 保存所有 Commit 对象的目录。

#### 方法：

- `name()`: 返回该提交对象的唯一标识符（通过序列化对象计算 SHA-1）。
- `save()`: 将 `Commit` 对象序列化后存储到磁盘中。
- `getFileContent()`: 获取某个文件在此提交版本中的内容。
- `modifiedFiles()`: 确定被修改的文件。
- `contain()`: 检查提交是否包含某个文件。

#### 特殊情况：

- **`MergeCommit`** 是 `Commit` 的一个子类，用于表示合并操作产生的提交。它包含两个父提交（`parent` 和 `parent2`），并记录合并的分支。

#### 功能：

`Commit` 是 Gitlet 中的版本单位，每次提交都会创建一个新的 `Commit` 对象。通过链式记录（`parent`），构建了提交历史的链表结构。

PS: 这里的设计逻辑实际上有些问题, client第一次看到modifiedFiles()时, 估计会摸不着头脑, 这是因为这个method默认将commit的文件内容与所有已跟踪文件的内容相比较来确定暂存区内容, 所以它本不应放到Commit的内部, 而应放在Repository中或其它地方实现, 当时的我有点傻乎乎的, 由于这些当时看不出来的设计问题而瞎忙活了挺久, 不过回过头来看到这些, 才更明白学习软件工程意义.

### `Stage` (暂存区)

#### 描述：

- `Stage` 是 Gitlet 中用于管理文件变更的暂存区域。
- 它包含了所有被暂存（`add`）或标记为删除（`remove`）的文件。

#### 数据成员：

- `status`: 一个 `Map<File, String>`，用于记录文件路径和其对应的 `Blob` 标识符。值为 `null` 的文件表示被标记为删除。

#### 方法：

- `add(File f)`: 将文件添加到暂存区，如果文件未改变或已被标记为删除，则取消操作。
- `remove(File f)`: 将文件从暂存区中移除，或者标记为删除。
- `save()`: 将暂存区对象序列化并保存到磁盘。
- `isEmpty()`: 检查暂存区是否为空。
- `printStatus()`: 打印暂存区的状态，包括已暂存文件、已删除文件等。

#### 功能：

`Stage` 是连接工作区和版本库的桥梁，用户在提交之前需要先将文件添加到暂存区。

PS: 同样, 这里的printStatus()也让人感到困惑, 因为Status不止包含暂存区内容, 还包含未暂存的更改, 以及未跟踪的文件信息, 然后这里还有一个非常难以理解的设计, 就是status以值null来表示文件已删除, 也就是说, file->null, 代表我们不再版本控制文件file, 我看了半天才看懂, 更不用说其他人来看了. 不得不说, 这个决定让我的代码更加难以理解, 而且让我的这个类随时可能会产生空指针异常, 真是一个糟糕的设计.

### Graph

#### 描述：

- `Graph` 管理分支和提交之间的关系。
- 它维护了项目的分支结构以及分支之间的提交图。

#### 数据成员：

- `HEAD`: 当前分支的指针。
- `BRANCH_DIR`: 保存分支信息的目录。

#### 方法：

- `init()`: 初始化分支图，创建一个默认分支（`master`）。
- `moveHead(Commit commit)`: 将当前分支的指针移动到新的提交。
- `createBranch(String branchName)`: 创建一个新的分支。
- `rmBranch(String branchName)`: 删除指定分支。
- `getHeadCommit()`: 获取当前分支的最新提交。
- `searchSplitPoint(String branch1, String branch2)`: 查找两个分支的共同祖先（用于合并操作）。

#### 功能：

Graph 是控制分支和版本关系的核心。它通过分支名与提交的映射，支持分支操作（切换、合并等）。

### Repository

#### 描述：

- `Repository` 是 Gitlet 的核心运行环境，负责管理所有文件、提交、分支以及暂存区。
- 它提供了高层次的 API，用于处理用户命令。

#### 核心成员：

- `CWD`: 当前工作目录。
- `GITLET_DIR`: `.gitlet` 主目录，存储所有版本控制相关的数据。

#### 方法：

- `init()`: 初始化一个新的 Gitlet 存储库。
- `add(String name)`: 将文件添加到暂存区。
- `commit(String message)`: 创建新的提交。
- `checkoutBranch(String branchName)`: 切换到指定分支。
- `merge(String otherBranch)`: 合并两个分支。

#### 功能：

`Repository` 是 Gitlet 的控制中心，所有用户命令都通过它进行处理。

## 实现

首先我想先介绍一些如何利用我设计的数据结构实现之前设计的抽象

* 提交(commit): 直接利用Commit类的方法即可实现
* 暂存区(staging area): 直接利用Stage类实现
* 索引(index): 读取HEAD commit和staging area的内容, 就可以得到所有已跟踪的文件的最新版本(staging area的文件版本更新)

接下来我将用我构建的抽象实现下面的关键指令

### init

用法: `java gitlet.Main init`

这个命令会初始化仓库, 我的仓库目录结构设计如下

```
.gitlet/
│── Commit/        # 存储所有提交对象（Commit）
│── Blob/          # 存储所有文件快照（Blob）
│── Branch/        # 存储各个分支的 HEAD 指针
│── HEAD           # 存储当前分支的名称
│── stage          # 存储暂存区内容
```

这里有一些和git的重要区别

* 我的本地分支引用全部存储在.gitlet/Branch中, 而git则存储在.git/refs/heads/中
* 我的暂存区存储在stage中, git的暂存区是通过比较index与HEAD的差异(我猜测)来确定的, 所以这里实际上对应了, 同一个抽象可以有多种实现方式, 让我想到了软件工程中说的只要确定了spec, 实现者就有很高的自由度按照自己的想法实现.

### add

用法: `java gitlet.Main add [file name]`

这个命令用来跟踪新文件, 或者更新已跟踪的文件, 我们只需更改stage类即可

### rm

用法: `java gitlet.Main rm [file name]`

这个命令用来移除已跟踪的文件, 不再对其进行版本控制, 同样只需更改stage类

### commmit

用法: `java gitlet.Main commit [message]`

这个命令用来保存当前目录状态, 需要给索引拍一个快照保存. 而如何获得索引刚刚已经说过了

### log

用法: `java gitlet.Main log`

这个比较简单, 只需要我们沿着HEAD commit的父提交遍历回去输出信息就可以了

### status

用法: `java gitlet.Main status`

显示当前存在的分支，并用`*`标记当前分支。此外，还显示哪些文件已被暂存以供添加或移除。以下是其应遵循的确切格式示例：

```bash
=== Branches ===  
*master  
other-branch  

=== Staged Files ===  
wug.txt  
wug2.txt  

=== Removed Files ===  
goodbye.txt  

=== Modifications Not Staged For Commit ===  
junk.txt (deleted)  
wug3.txt (modified)  

=== Untracked Files ===  
random.stuff  
```

* Staged Files & Removed Files: 输出暂存区内容即可

* Modifications Not Staged For Commit & Untracked Files: 对于这两个内容我有些疑问, 所谓已更改未暂存的文件到底是跟HEAD commit比较, 还是跟暂存区中的文件比较, 结果是, 对于暂存区中没有的文件, 是跟HEAD commit比较, 对于暂存区中有的文件, 则是跟stage比较, 那这不就是跟索引比较吗, 这也是我设计索引这个抽象的缘由. 所以有了索引这个抽象后, 这里我们就只需要把CWD跟索引比较就好了.

### find

用法: `java gitlet.Main find [commit message]`

遍历所有提交, 找到对应commit message的提交

### checkout

用法：

1. `java gitlet.Main checkout -- [file name]`
2. `java gitlet.Main checkout [commit id] -- [file name]`
3. `java gitlet.Main checkout [branch name]`

描述：

1. 获取文件在当前分支最新提交（head commit）中的版本，并将其放入工作目录中。如果工作目录中已经存在该文件，则覆盖该文件的当前版本。新版本的文件不会被标记为已暂存。
2. 获取文件在指定id的提交（commit）中的版本，并将其放入工作目录中。如果工作目录中已经存在该文件，则覆盖该文件的当前版本。新版本的文件不会被标记为已暂存。
3. 获取指定分支最新提交（head commit）中的所有文件，将其放入工作目录中，覆盖已经存在的文件版本（如果存在）。此外，在执行此命令的末尾，指定的分支将被视为当前分支（HEAD）。任何在当前分支中被跟踪但在被检出的分支中不存在的文件将被删除。暂存区将被清空，除非被检出的分支是当前分支（参见失败案例部分）。

实现方式:

1. 读取HEAD, 找到对应文件
2. 读取commit id, 找到对应文件
3. 读取.gitlet/branch, 找到对应的branch name引用的commit id, 加载所有文件到CWD, 移动HEAD, 清空缓冲区

为了更好的用户体验, gitlet还提供缩写操作哦.

一个[提交id]，如前所述，是一个十六进制数字。真实的Git的一个方便功能是可以使用唯一的前缀来缩写提交。例如，可以将

```
a0da1ea5a15ab613bf9961fd86f010cf74c7ee48
```

缩写为

```
a0da1e
```

前提是（很可能）没有其他对象的SHA-1标识符也以相同的六位数字开头。

观察了git的.git/objects目录, 发现Git将哈希值分为两部分来组织目录：

- **前2位字符**：作为子目录名称。
- **后38位字符**：作为文件名称。

例如，一个对象的哈希值为 `6f1e3e5fa74c5d1f7c53c1f99b94a1a3c3b1e5d5`，它会存储在：

```
.git/objects/6f/1e3e5fa74c5d1f7c53c1f99b94a1a3c3b1e5d5
```

示例：

```
.git/objects/
├── 00/
├── 6f/
│   └── 1e3e5fa74c5d1f7c53c1f99b94a1a3c3b1e5d5
├── 9a/
│   └── 2b8e23a5f8d4e6c4ea1f21a3c3b5d8f9ef3e4c
└── info/
└── pack/
```

发现这实际上是一种hashmap, 只不过是通过文件系统来实现的, 每个子目录可以看作一个bucket, 我们的gitlet也仿照此实现即可

### branch

使用方式：`java gitlet.Main branch [分支名]`

描述：创建一个具有指定名称的新分支，并将其指向当前的最新提交（head commit）。分支仅仅是对提交节点的一个引用（SHA-1标识符）的名称。此命令不会立即切换到新创建的分支上（与真实的Git一样）。在第一次调用`branch`之前，代码应该在一个名为“master”的默认分支上运行。

实现方式: 在.gitlet/branch新建一个ref, 以分支名命名, 内容是head commit id

### rm-branch

使用方式： `java gitlet.Main rm-branch [branch name]`

描述： Deletes the branch with the given name. This only means to delete the pointer associated with the branch; it does not mean to delete all commits that were created under the branch, or anything like that.

实现方式: 删除.gitlet/branch下的对应文件即可

### reset

用法：`java gitlet.Main reset [commit id]`

描述：检出给定提交所追踪的所有文件。移除当前追踪但在该提交中不存在的文件。同时将当前分支的头指针移动到该提交节点。请参阅介绍部分了解使用复位后头指针的变化示例。[commit id] 可以像检出命令一样简写。暂存区将被清空。该命令本质上是对任意提交的检出，同时更改了当前分支的头指针。

实现方式: 把当前分支的头部更改到commit id, CWD也恢复到该commit的状态, 清空暂存区

### merge

用法 `java gitlet.Main merge [branch name]`

最后的难题了, 想象以下情况:

![split_point](split_point.png)

我们在split point时被分配了一个branch, 要求实现某个功能, 当我们实现时, 往往会执行以下指令:

```bash
java gitlet.Main merge master
```

这时候, 我们就可以把自己的代码和master上其他人写的代码合并咯.

但是会有一些边界条件出现, 我们的文件可能会冲突. 总结来说, 我们的文件一共有三个版本, master版, branch版, split poing版, 其中master版和branch版优先级一致, merge优先保存它们的状态, split poing版优先级最低, 只有master版或branch版没有覆盖它(三个版本一致)时才会保存它的状态.

感觉这是所有指令中最繁琐的一个了, 但是如果理解了使用场景, 对于[不同情况](https://sp21.datastructur.es/materials/proj/proj2/proj2#merge)的处理可能会更好理解一些.

## Testing

第一次写这种需要验证持久性的项目, 由于我们需要测试本地文件状态, 而不是程序状态, 以往的JUnit测试都不再可行了, 不过还好项目提供了一个python测试脚本, 可供我们断言执行gitlet指令之后的本地文件状态, 使用方法在[这里](https://sp21.datastructur.es/materials/proj/proj2/proj2#testing).

接下来就根据在软件工程中学到的[分区选择测试用例策略](https://web.mit.edu/6.102/www/sp24/classes/02-testing/)编写测试. 

## 未来改进方向

首先是在实现中已经说过的不合理的设计可以改进, 其次在抽象中讲到的index实际上我的代码并没有实现(或者说当时并没有想到这个抽象), 导致之后的status实现手忙脚乱.

然后我觉得还可以添加以下功能:

### 添加commit id的缩写功能

一个[提交id]，如前所述，是一个十六进制数字。真实的Git的一个方便功能是可以使用唯一的前缀来缩写提交。例如，可以将

```
a0da1ea5a15ab613bf9961fd86f010cf74c7ee48
```

缩写为

```
a0da1e
```

前提是（很可能）没有其他对象的SHA-1标识符也以相同的六位数字开头。

观察了git的.git/objects目录, 发现Git将哈希值分为两部分来组织目录：

- **前2位字符**：作为子目录名称。
- **后38位字符**：作为文件名称。

例如，一个对象的哈希值为 `6f1e3e5fa74c5d1f7c53c1f99b94a1a3c3b1e5d5`，它会存储在：

```
.git/objects/6f/1e3e5fa74c5d1f7c53c1f99b94a1a3c3b1e5d5
```

示例：

```
.git/objects/
├── 00/
├── 6f/
│   └── 1e3e5fa74c5d1f7c53c1f99b94a1a3c3b1e5d5
├── 9a/
│   └── 2b8e23a5f8d4e6c4ea1f21a3c3b5d8f9ef3e4c
└── info/
└── pack/
```

发现这实际上是一种hashmap, 只不过是通过文件系统来实现的, 每个子目录可以看作一个bucket, 我们的gitlet也仿照此实现即可

### **支持子目录的版本控制**：

当前的实现似乎没有对子目录进行专门处理，虽然可以通过存储文件路径作为字符串实现子目录支持，但这并不优雅。可以参考 Git 的树结构设计，为子目录创建专属的 `Tree` 数据结构，从而更高效地管理目录结构。

### 支持更多 Git 的功能

- 实现 `stash` 功能，允许用户将当前工作区的更改保存到一个临时区域，而不需要提交。
- 实现 `rebase` 功能，用于重写提交历史。
- 实现 `cherry-pick` 功能，允许用户从其他分支中选择性地应用特定提交。

### 添加日志功能

想到我这里的Graph实现, 好像也可以增加一个类似 `git log --graph` 的功能，显示提交历史的分支结构图，帮助用户更直观地理解分支关系。但是可视化我还不是很会, 先挖个坑.

## Summary

总的来说, 这是一个耗费了我许多心血的项目, 也是我造的第一个轮子, 大约1500行代码, 一个星期的努力, 在所有测试都通过的那一刻, 心中有难以抑制的喜悦情绪, 原来软件工程是在做这些东西, 原来我也有能力写出一个能用的项目, 这些美好的成就感激励着我继续探索CS, 乃至1年后我能在这里写下这篇博客. 不过话说回来软件工程确实挺掉头发的, 特别是还不熟悉软件工程中的一些经典设计方式之前, 还是要深深感谢这些名校们无私的开源[课程](https://web.mit.edu/6.102/www/sp24/), 也感谢我的大学能给我转专业的机会.

## Questions

**Q1**: 突然想到, 我写这个项目的记录, 应该用第一人称好, 还是用第二人称呢?

用第二人称感觉像指导手册, 但是我只是记录一下我的实现过程, 希望看到的人更了解我的经历, 最终决定用第一人称吧

**Q2**: git中在分离的状态下提交更改会怎么样?

这个提交会变成一个孤立的提交, 只被head引用, 虽然有父提交, 但是没有分支名称, 在切换分支后无法找回, 除非通过git reflog, 会在git中存储30天后被删除

**Q3**: gitlet与真正的Git有哪些区别:

* 内部结构中, git会包含一个树, 类似一个目录结构, 每个树只储存当前目录下的文件版本, 子目录的文件由子目录的树来管理, 而gitlet的所有文件都由树的根节点管理.
* gielet的commits的元数据仅包含时间戳和日志信息
* 在真实的Git中，可以一次添加多个文件。而在Gitlet中，一次只能添加一个文件。
* 合并操作中, gitlet仅允许一次合并两个分支, git允许合并多个, 所以gielet的父提交最多有2个, 而git没有限制
* gitlet有global-log和find命令, git没有.(或许这两个命令是为了调试?)
* 重大缺陷(仅存在于我当初实现的gitlet中): 已暂存的文件中只有自上次提交以来有更改的文件, 实际上只版本控制了这些文件, 那些稳定的文件并没有被控制, 啧啧啧

**Q4**: gitlet不处理子目录, 是指gitlet只能跟踪当前文件夹下的文件, 而不能跟踪子目录下的文件吗?

并不是, 实际上gitlet可以直接把文件路径当作字符串存储, 而不使用树结构.

**Q5**: 为什么哈希算法不使用SHA-2

我们可以基本上假设，任何两个具有不同内容的对象具有相同SHA-1哈希值的概率是2^-160，约为10^-48。基本上，我们可以忽略哈希冲突的可能性. 这里也没有安全性的需求(拿到.git文件的人有我们的源代码), 所以SHA-1已经足够.

**Q6**: 为什么git要更改默认分支名称(master -> main)?

google了一下, 说是master暗含奴隶主的意思, 政治敏感词

**Q7**: 暂存区和索引的区别是什么

索引: 用户在下一次提交要保存的所有文件

暂存区: 用户在更新版本时, 对比上一个版本要更改的文件
