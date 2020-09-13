# GIT

#### [镜像](github.com.cnpmjs.org)

```csharp
git config --global credential.helper store
```

[廖雪峰git](https://www.liaoxuefeng.com/wiki/896043488029600/900002180232448#0)

> 使用git init命令创建仓库，执行git add . 和git commit（提交成功后）,再使用git branch命令，才显示出本地分支。
>
> **git branch ：查看本地分支**
> **git branch -a :查看本地及远程仓库的分支**
> **git branch --all ：查看本地及远程仓库的分支**
>
> **“因为git的分支必须指向一个commit，没有任何commit就没有任何分支**
>
> **提交第一个commit后git自动创建master分支”** -------廖雪峰

staged untracked

patch 修补

各种参数

git log

git cat-file -t/-p commitId

checkout xxx 可能会 分离头指针（detached HEAD）指不处于任意分支状态，此时commit会被丢掉（可用于实验性修改）

.git 下目录

新建Git仓库，有且仅有一个commit 对象

tree（类似 目录对象）

blob 保存内容（git 相同内容只保存一个blob对象）

git status

#### 重要概念

工作区：电脑里能看到的目录文件夹

版本库：.git

暂存区：git add把文件从工作区>>>>暂存区

**HEAD** 指向分支引用

“指针”  -> commitId

`HEAD` 文件、（尚待创建的）`index` 文件，和 `objects` 目录、`refs` 目录。 它们都是 Git 的核心组成部分。 `objects` 目录存储所有数据内容；`refs` 目录存储指向数据（分支、远程仓库和标签等）的提交对象的指针； `HEAD` 文件指向目前被检出的分支；`index` 文件保存暂存区信息。



git 一次 add 多个文件（多个文件用 空格 隔开）

add 文件夹：git add catalog_name

```shell
#修改commit message

git commit --amend    修改最新的
git rebase -i [targetIds]^ 修改旧的 （reword） ，合并多个 commit message（squash压缩）
INCOMPATIBLE OPTIONS

git rebase --continue 



# 官网例子，branch commit 移植
# 和merge（在master分支）操作类似，但提交历史不同，rebase（当前分支，非master）操作的log更干净。
git rebase master
git rebase master topic


git rebase --onto master next topic
git rebase --onto topicA~5（end） topicA~3（new start index） topicA 删除分支topicA commit message 索引从 0开始 


git rebase处理过程,使用git rebase之后,只是返回冲突出现的提交处的commit,之后要在这个commit中进行解决冲突;再使用git add操作添加好要解决冲突后的文件,之后还要再执行一次git rebase --continu,到此git rebase衍合过程才真正结束;
```

### 文件差异

git diff ：查看变更内容 ; 比较内容 ; 当前工作目录和上次提交与本地索引间的差异

git diff --cached 暂存区与HEAD差异

git diff -- filePath...

git diff branch1（name/commitId） branch2 -- file 不同版本指定文件的差异



#### 回退/撤销修改

git reset Head \<file>... 暂存区指定文件的变更不保留，file为空则所有文件变更不保留 

git reset commit_id file.txt 回退指定文件到指定版本

git reset --hard commit_id 回退到 commit_id

> git-checkout - Switch branches or restore（还原） working tree files
>
> git checkout -- \<file>... 工作区恢复为暂存区一样
>
> 一种是`file`自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；
>
> 一种是`file`已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态

git reflog 

git revert





#### 保留工作现场

git stash

#### 远程

```
$ git remote add origin git@github.com:michaelliao/learngit.git
$ git push -u origin master


```



#### .gitignore

文件里每行可指定忽略规则

> \*.xxx 忽略指定后缀文件
>
> xxx/ 忽略指定目录
>
> xxx 忽略指定目录和文件

#### 分支管理

查看分支：`git branch`

创建分支：`git branch <name>`

切换分支：`git checkout <name>`或者`git switch <name>`

创建+切换分支：`git checkout -b <name>`或者`git switch -c <name>`

建立分支关联：git checkout -b local_branch remote_branch

合并某分支到当前分支：`git merge <name>`

删除分支：`git branch -d <name>`

> 跟踪远程分支
> 1）如果远程新建了一个分支，本地没有该分支，可以用`git checkout --track origin/branch_name`，这时候本地会新建一个分支名叫`branch_name`，会自动跟踪远程的同名分支`branch_name`。
> 2）用上面中方法，得到的分支名永远和远程的分支名一样，如果想新建一个本地分支不同名字，同时跟踪一个远程分支可以利用。
> `git checkout -b new_branch_name branch_name`，这条指令本来是根据一个`branch_name`分支分出一个本地分支`new_branch_name`，但是如果所根据的分支`branch_name`是一个远程分支名，那么本地的分支会自动的`track`远程分支。建议跟踪分支和被跟踪远程分支同名。
>
> 总结：一般我们就用git push --set-upstream origin branch_name来在远程创建一个与本地branch_name同名的分支并跟踪；利用git checkout --track origin/branch_name来在本地创建一个与branch_name同名分支跟踪远程分支。

Git会用`Fast forward`模式，但这种模式下，删除分支后，会丢掉分支信息。

`--no-ff`方式的`git merge`

```
$ git log --graph --pretty=oneline --abbrev-commit
```

``` 
git cherry-pick commitid   指定commit提交“复制”到当前分支（对当前分支commit）
```



```
git branch --set-upstream-to=origin/dev dev
```



git push -f origin commitid:master 强退远程master（实验用，-f 不推荐使用）

commit merge

squash merge

rebase merge    （cherry-pick） rerere



```console
git branch recover-branch 恢复分支
git log --pretty=oneline recover-branch
```

#### 远程仓库

git remote add xxx(default origin) target_url   (xxx  代表了 target_url)

git fetch

git merge --allow-unrelated-merge

git push xxx/branch branch



`git revert` 用于记录一些新的提交，以逆转某些较早提交的效果（通常只有一个错误的提交）。

`git revert commitid` 新建一个commit 还原 commitid 中做的变更



#### 邮件

```console
git format-patch -M origin/master


~/.gitconfig
[imap]
  folder = "[Gmail]/Drafts"
  host = imaps://imap.gmail.com
  user = user@gmail.com
  pass = YX]8g76G_2^sFbd
  port = 993
  sslverify = false
# or
[sendemail]
  smtpencryption = tls
  smtpserver = smtp.gmail.com
  smtpuser = user@gmail.com
  smtpserverport = 587
  
  
  
$ git send-email *.patch
```

#### ssh key

```shell
$ cd ~/.ssh 或cd .ssh

$ cd ~  #保证当前路径在”~”下

$ ssh-keygen -t rsa -C "xxxxxx@yy.com"  #建议填写自己真实有效的邮箱地址
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/xxxx_000/.ssh/id_rsa):   #不填直接回车
Enter passphrase (empty for no passphrase):   #输入密码（可以为空）
Enter same passphrase again:   #再次确认密码（可以为空）
Your identification has been saved in /c/Users/xxxx_000/.ssh/id_rsa.   #生成的密钥
Your public key has been saved in /c/Users/xxxx_000/.ssh/id_rsa.pub.  #生成的公钥
The key fingerprint is:
e3:51:33:xx:xx:xx:xx:xxx:61:28:83:e2:81 xxxxxx@yy.com
*本机已完成ssh key设置，其存放路径为：c:/Users/xxxx_000/.ssh/下。
注释：可生成ssh key自定义名称的密钥，默认id_rsa。

$ ssh-keygen -t rsa -C "邮箱地址" -f ~/.ssh/githug_blog_keys #生成ssh key的名称为githug_blog_keys，慎用容易出现其它异常。
添加ssh key到GItHub
复制id_rsa.pub的公钥内容。

配置账户
$ git config --global user.name “your_username”  #设置用户名
$ git config --global user.email “your_registered_github_Email”  #设置邮箱地址(建议用注册giuhub的邮箱)

测试ssh keys是否设置成功。
$ ssh -T git@github.com
The authenticity of host 'github.com (192.30.252.129)' can't be established.
RSA key fingerprint is 16:27:xx:xx:xx:xx:xx:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)? yes #确认你是否继续联系，输入yes
Warning: Permanently added 'github.com,192.30.252.129' (RSA) to the list of known hosts.
Enter passphrase for key '/c/Users/xxxx_000/.ssh/id_rsa':  #生成ssh kye是密码为空则无此项，若设置有密码则有此项且，输入生成ssh key时设置的密码即可。



ssh -T git@github.com
  git@github.com: Permission denied (publickey).
# 一次性配置 ，重启会失效
ssh-agent bash
ssh-add ~/.ssh/id_rsa_name
```



配置config文件（永久多ssh管理）

- 假如曾与其他代码托管平台进行ssh关联，则需要配置`~/.ssh/config`文件进行**多ssh管理**，否则可以忽略。

- 或者其实也可以直接使用**和其他代码托管平台相同的密钥对一起管理**，就可以省去生成新的密钥对和写配置文件的麻烦，直接第四点开始操作即可，看个人选择。

- 为什么说是永久呢？因为有一次性配置的方法，当重启cmder或shell就会失效。就是`ssh-agent`，`ssh-add ~/.ssh/id_rsa`和`ssh-add -l`的方法，这是一次性的

- **重点敲黑板**：`config`文件有几点需要注意的：

  - **Host**：代码托管平台的别名。讲道理其实是可以取自己喜欢的，但是这个别名和后面要用到的ssh链接 `git@github.com:xxx/xxx.git` 中的 `@` 符号后面的内容要一致，而一般来说github默认提供的就是`git@github.com`，因此为了方便，**github的Host写github.com即可**，别取别名了

  - **HostName**：代码托管平台真正的IP地址或域名

  - **IdentityFile**：对应的密钥文件路径。**必须写绝对路径**，以下几种写法都是可以的，亲测有效：

    ```
    ~/.ssh/id_rsa_github
    C://Users//xxx//.ssh//id_rsa_github
    C:/Users/xxx/.ssh/id_rsa_github
    C:\\Users\\xxx\\.ssh\\id_rsa_github
    C:\Users\xxx\.ssh\id_rsa_github12345
    ```

  - **PreferredAuthentications**：配置登录时用什么权限认证。可设为`publickey`，`password publickey`，`keyboard-interactive`等

  - **User**：对应的用户名。亲测github_username写git也可以。

- 写好的config文件如下示例，不要出现单词拼写错误，可借助高亮判断：

  ```
  cd ~/.ssh
  touch config
  vi config
  
  Host exist_server
      HostName exist_server_IP/exist_server_domain
      IdentityFile ~/.ssh/id_rsa_exist_server
      PreferredAuthentications publickey
      User username
  
  Host github.com
      HostName github.com
      IdentityFile ~/.ssh/id_rsa_github
      PreferredAuthentications publickey
      User github_username
  
  :wq
  ```

# IDEA

<https://blog.csdn.net/weixin_40861707/article/details/83586087>

System setting：设置上次项目

Maven
Git、Github

keymap --> main menu --> code  --> completion

> Editor   --> general
> ​              --> code completion --> case sensitive
> ​              --> Auto Import
> ​              --> Appearance

常用组合键：

Ctrl + T
Ctrl + G

Ctrl + Alt + T
Ctrl + H



<https://github.com/getlantern/download>

插件下载问题

http-proxy 改为 auto_detect      http://127.0.0.1:1080

或 custom plugin repository https://plugins.jetbrans.com/

## Debug

1. Step Into    跳入 
2. Step Over   跳过 
3. Step Return 执行完当前method，然后return跳出此method
4. step Filter 逐步过滤 一直执行直到遇到未经过滤的位置或断点(设置Filter:window-preferences-java-Debug-step Filtering) 
5. resume 重新开始执行debug,一直运行直到遇到breakpoint。

## 常用快捷键

http://idea.pyjuan.com/idea_code.html#

**alt+enter**（相当于eclipse的ctrl+1）

双击Shift简直是神快捷键，可以直接到你想要的文件夹和文件



## 自动代码

常用的有fori/sout/psvm+Tab即可生成循环、System.out、main方法等boilerplate样板代码 。

例如要输入for(User user : users)只需输入user.for+Tab ；

再比如，要输入Date birthday = user.getBirthday()只需输入user.getBirthday().var+Tab即可。

代码标签输入完成后，按Tab，生成代码。

1. Ctrl+Alt+O 优化导入的类和包
2. Alt+Insert 生成代码(如get,set方法,构造函数等) 或者右键（Generate）
3. fori/sout/psvm + Tab
4. Ctrl+Alt+T 生成try catch 或者 Alt+enter
5. CTRL+ALT+T 把选中的代码放在 TRY{} IF{} ELSE{} 里
6. Ctrl + O 重写方法
7. Ctrl + I 实现方法
8. Ctr+shift+U 大小写转化
9. ALT+回车 导入包,自动修正
10. ALT+/ 代码提示
11. CTRL+J 自动代码
12. Ctrl+Shift+J，整合两行为一行
13. CTRL+空格 代码提示
14. CTRL+SHIFT+SPACE 自动补全代码
15. CTRL+ALT+L 格式化代码
16. CTRL+ALT+I 自动缩进
17. CTRL+ALT+O 优化导入的类和包
18. ALT+INSERT 生成代码(如GET,SET方法,构造函数等)
19. CTRL+E 最近更改的代码
20. CTRL+ALT+SPACE 类名或接口名提示
21. CTRL+P 方法参数提示
22. CTRL+Q，可以看到当前方法的声明
23. Shift+F6 重构-重命名 (包、类、方法、变量、甚至注释等)
24. Ctrl+Alt+V 提取变量

## 查询快捷键

1. Ctrl＋Shift＋Backspace可以跳转到上次编辑的地
2. CTRL+ALT+ left/right 前后导航编辑过的地方
3. ALT+7 靠左窗口显示当前文件的结构
4. Ctrl+F12 浮动显示当前文件的结构
5. ALT+F7 找到你的函数或者变量或者类的所有引用到的地方
6. CTRL+ALT+F7 找到你的函数或者变量或者类的所有引用到的地方
7. Ctrl+Shift+Alt+N 查找类中的方法或变量
8. 双击SHIFT  在项目的所有目录查找文件
9. Ctrl+N 查找类
10. Ctrl+Shift+N 查找文件
11. CTRL+G 定位行
12. CTRL+F 在当前窗口查找文本
13. CTRL+SHIFT+F 在指定窗口查找文本
14. CTRL+R 在 当前窗口替换文本
15. CTRL+SHIFT+R 在指定窗口替换文本
16. ALT+SHIFT+C 查找修改的文件
17. CTRL+E 最近打开的文件
18. F3 向下查找关键字出现位置
19. SHIFT+F3 向上一个关键字出现位置
20. 选中文本，按Alt+F3 ，高亮相同文本，F3逐个往下查找相同文本
21. F4 查找变量来源
22. CTRL+SHIFT+O 弹出显示查找内容
23. Ctrl+W 选中代码，连续按会有其他效果
24. F2 或Shift+F2 高亮错误或警告快速定位
25. Ctrl+Up/Down 光标跳转到第一行或最后一行下
26. Ctrl+B 快速打开光标处的类或方法
27. CTRL+ALT+B 找所有的子类
28. CTRL+SHIFT+B 找变量的类
29. Ctrl+Shift+上下键 上下移动代码
30. Ctrl+Alt+ left/right 返回至上次浏览的位置
31. Ctrl+X 删除行
32. Ctrl+D 复制行
33. Ctrl+/ 或 Ctrl+Shift+/ 注释（// 或者/…/ ）
34. Ctrl+H 显示类结构图
35. Ctrl+Q 显示注释文档
36. Alt+F1 查找代码所在位置
37. Alt+1 快速打开或隐藏工程面板
38. Alt+ left/right 切换代码视图
39. ALT+ ↑/↓ 在方法间快速移动定位
40. CTRL+ALT+ left/right 前后导航编辑过的地方
41. Ctrl＋Shift＋Backspace可以跳转到上次编辑的地
42. Alt+6 查找TODO

## 其他快捷键

1. SHIFT+ENTER 另起一行
2. CTRL+Z 倒退(撤销)
3. CTRL+SHIFT+Z 向前(取消撤销)
4. CTRL+ALT+F12 资源管理器打开文件夹
5. ALT+F1 查找文件所在目录位置
6. SHIFT+ALT+INSERT 竖编辑模式
7. CTRL+F4 关闭当前窗口
8. Ctrl+Alt+V，可以引入变量。例如：new String(); 自动导入变量定义
9. Ctrl+~，快速切换方案（界面外观、代码风格、快捷键映射等菜单）

## 调试快捷键

其实常用的 就是F8 F7 F9 最值得一提的就是Drop Frame 可以让运行过的代码从头再来。

1. alt+F8 debug时选中查看值
2. Alt+Shift+F9，选择 Debug
3. Alt+Shift+F10，选择 Run
4. Ctrl+Shift+F9，编译
5. Ctrl+Shift+F8，查看断点
6. F7，步入
7. Shift+F7，智能步入
8. Alt+Shift+F7，强制步入
9. F8，步过
10. Shift+F8，步出
11. Alt+Shift+F8，强制步过
12. Alt+F9，运行至光标处
13. Ctrl+Alt+F9，强制运行至光标处
14. F9，恢复程序
15. Alt+F10，定位到断点

## 重构

1. Ctrl+Alt+Shift+T，弹出重构菜单
2. Shift+F6，重命名
3. F6，移动
4. F5，复制
5. Alt+Delete，安全删除
6. Ctrl+Alt+N，内联

## 十大Intellij IDEA快捷键

Intellij IDEA中有很多快捷键让人爱不释手，stackoverflow上也有一些有趣的讨论。每个人都有自己的最爱，想排出个理想的榜单还真是困难。

以前也整理过Intellij的快捷键，这次就按照我日常开发时的使用频率，简单分类列一下我最喜欢的十大快捷-神-键吧。

### 1 智能提示

Intellij首当其冲的当然就是Intelligence智能！基本的代码提示用Ctrl+Space，还有更智能地按类型信息提示Ctrl+Shift+Space，但因为Intellij总是随着我们敲击而自动提示，所以很多时候都不会手动敲这两个快捷键(除非提示框消失了)。

用F2/ Shift+F2移动到有错误的代码，Alt+Enter快速修复(即Eclipse中的Quick Fix功能)。当智能提示为我们自动补全方法名时，我们通常要自己补上行尾的反括号和分号，当括号嵌套很多层时会很麻烦，这时我们只需敲Ctrl+Shift+Enter就能自动补全末尾的字符。而且不只是括号，例如敲完if/for时也可以自动补上{}花括号。

最后要说一点，Intellij能够智能感知Spring、Hibernate等主流框架的配置文件和类，以静制动，在看似“静态”的外表下，智能地扫描理解你的项目是如何构造和配置的。

### 2 重构

Intellij重构是另一完爆Eclipse的功能，其智能程度令人瞠目结舌，比如提取变量时自动检查到所有匹配同时提取成一个变量等。尤其看过《重构-改善既有代码设计》之后，有了Intellij的配合简直是令人大呼过瘾！也正是强大的智能和重构功能，使Intellij下的TDD开发非常顺畅。

切入正题，先说一个无敌的重构功能大汇总快捷键Ctrl+Shift+Alt+T，叫做Refactor This。按法有点复杂，但也符合Intellij的风格，很多快捷键都要双手完成，而不像Eclipse不少最有用的快捷键可以潇洒地单手完成(不知道算不算Eclipse的一大优点)，但各位用过Emacs的话就会觉得也没什么了(非Emacs黑)。

此外，还有些最常用的重构技巧，因为太常用了，若每次都在Refactor This菜单里选的话效率有些低。比如Shift+F6直接就是改名，Ctrl+Alt+V则是提取变量。关注Java技术栈微信公众号，在后台回复关键字：IDEA，可以获取一份栈长整理的 IDEA 最新技术干货。

### 3 代码生成

这一点类似Eclipse，虽不是独到之处，但因为日常使用频率极高，所以还是罗列在榜单前面。常用的有fori/sout/psvm+Tab即可生成循环、System.out、main方法等boilerplate样板代码，用Ctrl+J可以查看所有模板。

后面“辅助”一节中将会讲到Alt+Insert，在编辑窗口中点击可以生成构造函数、toString、getter/setter、重写父类方法等。这两个技巧实在太常用了，几乎每天都要生成一堆main、System.out和getter/setter。

另外，Intellij IDEA 13中加入了后缀自动补全功能(Postfix Completion)，比模板生成更加灵活和强大。例如要输入for(User user : users)只需输入user.for+Tab。再比如，要输入Date birthday = user.getBirthday();只需输入user.getBirthday().var+Tab即可。

### 4 编辑

编辑中不得不说的一大神键就是能够自动按语法选中代码的Ctrl+W以及反向的Ctrl+Shift+W了。此外，Ctrl+Left/Right移动光标到前/后单词，Ctrl+[/]移动到前/后代码块，这些类Vim风格的光标移动也是一大亮点。以上Ctrl+Left/Right/[]加上Shift的话就能选中跳跃范围内的代码。Alt+Forward/Backward移动到前/后方法。还有些非常普通的像Ctrl+Y删除行、Ctrl+D复制行、Ctrl+折叠代码就不多说了。

关于光标移动再多扩展一点，除了Intellij本身已提供的功能外，我们还可以安装ideaVim或者emacsIDEAs享受到Vim的快速移动和Emacs的AceJump功能(超爽！)。

另外，Intellij的书签功能也是不错的，用Ctrl+Shift+Num定义1-10书签(再次按这组快捷键则是删除书签)，然后通过Ctrl+Num跳转。这避免了多次使用前/下一编辑位置Ctrl+Left/Right来回跳转的麻烦，而且此快捷键默认与Windows热键冲突(默认多了Alt，与Windows改变显示器显示方向冲突，一不小心显示器就变成倒着显式的了，冏啊)。

### 5 查找打开

类似Eclipse，Intellij的Ctrl+N/Ctrl+Shift+N可以打开类或资源，但Intellij更加智能一些，我们输入的任何字符都将看作模糊匹配，省却了Eclipse中还有输入*的麻烦。最新版本的IDEA还加入了Search Everywhere功能，只需按Shift+Shift即可在一个弹出框中搜索任何东西，包括类、资源、配置项、方法等等。

类的继承关系则可用Ctrl+H打开类层次窗口，在继承层次上跳转则用Ctrl+B/Ctrl+Alt+B分别对应父类或父方法定义和子类或子方法实现，查看当前类的所有方法用Ctrl+F12。

要找类或方法的使用也很简单，Alt+F7。要查找文本的出现位置就用Ctrl+F/Ctrl+Shift+F在当前窗口或全工程中查找，再配合F3/Shift+F3前后移动到下一匹配处。

Intellij更加智能的又一佐证是在任意菜单或显示窗口，都可以直接输入你要找的单词，Intellij就会自动为你过滤。关注Java技术栈微信公众号，在后台回复关键字：IDEA，可以获取一份栈长整理的 IDEA 最新技术干货。

### 6 其他辅助

以上这些神键配上一些辅助快捷键，即可让你的双手90%以上的时间摆脱鼠标，专注于键盘仿佛在进行钢琴表演。这些不起眼却是至关重要的最后一块拼图有：

Ø 命令：Ctrl+Shift+A可以查找所有Intellij的命令，并且每个命令后面还有其快捷键。所以它不仅是一大神键，也是查找学习快捷键的工具。

Ø 新建：Alt+Insert可以新建类、方法等任何东西。

Ø 格式化代码：格式化import列表Ctrl+Alt+O，格式化代码Ctrl+Alt+L。

Ø 切换窗口：Alt+Num，常用的有1-项目结构，3-搜索结果，4/5-运行调试。Ctrl+Tab切换标签页，Ctrl+E/Ctrl+Shift+E打开最近打开过的或编辑过的文件。

Ø 单元测试：Ctrl+Alt+T创建单元测试用例。

Ø 运行：Alt+Shift+F10运行程序，Shift+F9启动调试，Ctrl+F2停止。

Ø 调试：F7/F8/F9分别对应Step into，Step over，Continue。

此外还有些我自定义的，例如水平分屏Ctrl+|等，和一些神奇的小功能Ctrl+Shift+V粘贴很早以前拷贝过的，Alt+Shift+Insert进入到列模式进行按列选中。

Ø Top #10切来切去：Ctrl+Tab

Ø Top #9选你所想：Ctrl+W

Ø Top #8代码生成：Template/Postfix +Tab

Ø Top #7发号施令：Ctrl+Shift+A

Ø Top #6无处藏身：Shift+Shift

Ø Top #5自动完成：Ctrl+Shift+Enter

Ø Top #4创造万物：Alt+Insert

### 太难割舍，前三名并列吧！

Ø Top #1智能补全：Ctrl+Shift+Space

Ø Top #1自我修复：Alt+Enter

Ø Top #1重构一切：Ctrl+Shift+Alt+T

CTRL+ALT+ left/right 前后导航编辑过的地方 Ctrl＋Shift＋Backspace可以跳转到上次编辑的地方



-javaagent:D:/Program Files (x86)/IntelliJ IDEA 2018.1.6/bin/JetbrainsCrack-3.1-release-enc.jar

## 激活步骤

#### 1.1 修改配置

1.1.1 把下载的激活补丁拷贝到安装目录的bin目录

1.1.2 再bin目录下修改 **idea.exe.vmoptions** 和 **idea.exe.vmoptions 文件**，添加以下内容



```
-javaagent:安装目录/bin/JetbrainsCrack-2.6.2.jar
```



**1.1.3上面两个文件都要添加这一行，-javaagent:后面的就是下载补丁的位置**

1.1.4 添加完成之后，启动IDEA，如果点启动图标没有反应，那就是配置写错了。注意Windows Mac或者是Linux目录分割线都可以用 ‘/’

#### 1.2 填写Activation Code

重启IDEA 修改Activation Code

粘贴下面的代码，修改邮箱和用户名



```
ThisCrackLicenseId-{ 
"licenseId":"ThisCrackLicenseId", 
"licenseeName":"你想要的用户名",
"assigneeName":"",
"assigneeEmail":"随便填一个邮箱(我填的:idea@163.com)",
"licenseRestriction":"For This Crack, Only Test! Please support genuine!!!", 
"checkConcurrentUse":false, 
"products":[
    {"code":"II","paidUpTo":"2099-12-31"}, 
    {"code":"DM","paidUpTo":"2099-12-31"}, 
    {"code":"AC","paidUpTo":"2099-12-31"},
    {"code":"RS0","paidUpTo":"2099-12-31"},
    {"code":"WS","paidUpTo":"2099-12-31"}, 
    {"code":"DPN","paidUpTo":"2099-12-31"},
    {"code":"RC","paidUpTo":"2099-12-31"}, 
    {"code":"PS","paidUpTo":"2099-12-31"}, 
    {"code":"DC","paidUpTo":"2099-12-31"},
    {"code":"RM","paidUpTo":"2099-12-31"}, 
    {"code":"CL","paidUpTo":"2099-12-31"}, 
    {"code":"PC","paidUpTo":"2099-12-31"}
], 
"hash":"2911276/0", 
"gracePeriodDays":7, 
"autoProlongated":false}
```

激活成功，到2099年过期。

```
ThisCrackLicenseId-{
“licenseId”:”11011”,
“licenseeName”:”Wechat”,
“assigneeName”:”IT--Pig”,
“assigneeEmail”:”1113449881@qq.com”,
“licenseRestriction”:””,
“checkConcurrentUse”:false,
“products”:[
{“code”:”II”,”paidUpTo”:”2099-12-31”},
{“code”:”DM”,”paidUpTo”:”2099-12-31”},
{“code”:”AC”,”paidUpTo”:”2099-12-31”},
{“code”:”RS0”,”paidUpTo”:”2099-12-31”},
{“code”:”WS”,”paidUpTo”:”2099-12-31”},
{“code”:”DPN”,”paidUpTo”:”2099-12-31”},
{“code”:”RC”,”paidUpTo”:”2099-12-31”},
{“code”:”PS”,”paidUpTo”:”2099-12-31”},
{“code”:”DC”,”paidUpTo”:”2099-12-31”},
{“code”:”RM”,”paidUpTo”:”2099-12-31”},
{“code”:”CL”,”paidUpTo”:”2099-12-31”},
{“code”:”PC”,”paidUpTo”:”2099-12-31”}
],
“hash”:”2911276/0”,
“gracePeriodDays”:7,
“autoProlongated”:false}
```



## 附带我的idea.exe.vmoptions

```
-server
-Xms128m
-Xmx512m
-XX:ReservedCodeCacheSize=240m
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-Djdk.http.auth.tunneling.disabledSchemes=""
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-javaagent:E:/Program Files/JetBrains/IntelliJ IDEA 2018.2.5/bin/JetbrainsCrack-3.1-release-enc.jar
```





```txt
YZVR7WDLV8-eyJsaWNlbnNlSWQiOiJZWlZSN1dETFY4IiwibGljZW5zZWVOYW1lIjoiamV0YnJhaW5zIGpzIiwiYXNzaWduZWVOYW1lIjoiIiwiYXNzaWduZWVFbWFpbCI6IiIsImxpY2Vuc2VSZXN0cmljdGlvbiI6IkZvciBlZHVjYXRpb25hbCB1c2Ugb25seSIsImNoZWNrQ29uY3VycmVudFVzZSI6ZmFsc2UsInByb2R1Y3RzIjpbeyJjb2RlIjoiSUkiLCJwYWlkVXBUbyI6IjIwMTktMTEtMjYifSx7ImNvZGUiOiJBQyIsInBhaWRVcFRvIjoiMjAxOS0xMS0yNiJ9LHsiY29kZSI6IkRQTiIsInBhaWRVcFRvIjoiMjAxOS0xMS0yNiJ9LHsiY29kZSI6IlBTIiwicGFpZFVwVG8iOiIyMDE5LTExLTI2In0seyJjb2RlIjoiR08iLCJwYWlkVXBUbyI6IjIwMTktMTEtMjYifSx7ImNvZGUiOiJETSIsInBhaWRVcFRvIjoiMjAxOS0xMS0yNiJ9LHsiY29kZSI6IkNMIiwicGFpZFVwVG8iOiIyMDE5LTExLTI2In0seyJjb2RlIjoiUlMwIiwicGFpZFVwVG8iOiIyMDE5LTExLTI2In0seyJjb2RlIjoiUkMiLCJwYWlkVXBUbyI6IjIwMTktMTEtMjYifSx7ImNvZGUiOiJSRCIsInBhaWRVcFRvIjoiMjAxOS0xMS0yNiJ9LHsiY29kZSI6IlBDIiwicGFpZFVwVG8iOiIyMDE5LTExLTI2In0seyJjb2RlIjoiUk0iLCJwYWlkVXBUbyI6IjIwMTktMTEtMjYifSx7ImNvZGUiOiJXUyIsInBhaWRVcFRvIjoiMjAxOS0xMS0yNiJ9LHsiY29kZSI6IkRCIiwicGFpZFVwVG8iOiIyMDE5LTExLTI2In0seyJjb2RlIjoiREMiLCJwYWlkVXBUbyI6IjIwMTktMTEtMjYifSx7ImNvZGUiOiJSU1UiLCJwYWlkVXBUbyI6IjIwMTktMTEtMjYifV0sImhhc2giOiIxMTA1NzI3NC8wIiwiZ3JhY2VQZXJpb2REYXlzIjowLCJhdXRvUHJvbG9uZ2F0ZWQiOmZhbHNlLCJpc0F1dG9Qcm9sb25nYXRlZCI6ZmFsc2V9-rsJR5mlJcjibqRu1gQAMUCngMe8i+AOWIi+JZkNFYPET2G1ONcLPcIzoATTRi6ofkDm5l+3Y4HXjBPjVU6bHDdMBAzCnUqpXKsCknwSYyPSU0Y5pzuLvw6O9aPlQ46UBoTEC2BL5W6f11S7NlAq7tTbDuvFUynqSGAmTEfuZtKmzRmp20ejTPuMlSO7UqSkZvkg6YvSTrax1d2K+P9SAmVGZ9iC7AzBs4AwTf84QB9qHvE/Nh0oELSHWGG9hsZZ7sVghI/39/jPQFTp8GLFsl36ZPybPhGDam721zxS9H++/eJk23Jz3nxaRluE4dWmpHrDg1qBHp8qVpSFejg2QYw==-MIIElTCCAn2gAwIBAgIBCTANBgkqhkiG9w0BAQsFADAYMRYwFAYDVQQDDA1KZXRQcm9maWxlIENBMB4XDTE4MTEwMTEyMjk0NloXDTIwMTEwMjEyMjk0NlowaDELMAkGA1UEBhMCQ1oxDjAMBgNVBAgMBU51c2xlMQ8wDQYDVQQHDAZQcmFndWUxGTAXBgNVBAoMEEpldEJyYWlucyBzLnIuby4xHTAbBgNVBAMMFHByb2QzeS1mcm9tLTIwMTgxMTAxMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxcQkq+zdxlR2mmRYBPzGbUNdMN6OaXiXzxIWtMEkrJMO/5oUfQJbLLuMSMK0QHFmaI37WShyxZcfRCidwXjot4zmNBKnlyHodDij/78TmVqFl8nOeD5+07B8VEaIu7c3E1N+e1doC6wht4I4+IEmtsPAdoaj5WCQVQbrI8KeT8M9VcBIWX7fD0fhexfg3ZRt0xqwMcXGNp3DdJHiO0rCdU+Itv7EmtnSVq9jBG1usMSFvMowR25mju2JcPFp1+I4ZI+FqgR8gyG8oiNDyNEoAbsR3lOpI7grUYSvkB/xVy/VoklPCK2h0f0GJxFjnye8NT1PAywoyl7RmiAVRE/EKwIDAQABo4GZMIGWMAkGA1UdEwQCMAAwHQYDVR0OBBYEFGEpG9oZGcfLMGNBkY7SgHiMGgTcMEgGA1UdIwRBMD+AFKOetkhnQhI2Qb1t4Lm0oFKLl/GzoRykGjAYMRYwFAYDVQQDDA1KZXRQcm9maWxlIENBggkA0myxg7KDeeEwEwYDVR0lBAwwCgYIKwYBBQUHAwEwCwYDVR0PBAQDAgWgMA0GCSqGSIb3DQEBCwUAA4ICAQAF8uc+YJOHHwOFcPzmbjcxNDuGoOUIP+2h1R75Lecswb7ru2LWWSUMtXVKQzChLNPn/72W0k+oI056tgiwuG7M49LXp4zQVlQnFmWU1wwGvVhq5R63Rpjx1zjGUhcXgayu7+9zMUW596Lbomsg8qVve6euqsrFicYkIIuUu4zYPndJwfe0YkS5nY72SHnNdbPhEnN8wcB2Kz+OIG0lih3yz5EqFhld03bGp222ZQCIghCTVL6QBNadGsiN/lWLl4JdR3lJkZzlpFdiHijoVRdWeSWqM4y0t23c92HXKrgppoSV18XMxrWVdoSM3nuMHwxGhFyde05OdDtLpCv+jlWf5REAHHA201pAU6bJSZINyHDUTB+Beo28rRXSwSh3OUIvYwKNVeoBY+KwOJ7WnuTCUq1meE6GkKc4D/cXmgpOyW/1SmBz3XjVIi/zprZ0zf3qH5mkphtg6ksjKgKjmx1cXfZAAX6wcDBNaCL+Ortep1Dh8xDUbqbBVNBL4jbiL3i3xsfNiyJgaZ5sX7i8tmStEpLbPwvHcByuf59qJhV/bZOl8KqJBETCDJcY6O2aqhTUy+9x93ThKs1GKrRPePrWPluud7ttlgtRveit/pcBrnQcXOl1rHq7ByB8CFAxNotRUYL9IF5n3wJOgkPojMy6jetQA5Ogc8Sm7RG6vg1yow==
```

