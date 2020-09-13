[转载于](https://zhuanlan.zhihu.com/p/39237625)

![](https://ws1.sinaimg.cn/large/006xzusPly1g5ni7671hjj30eg0boaak.jpg)

## 1. 在linux上安装Git

*\*$ sudo apt-get install git***

## 2.创建版本库

- 创建新目录 **$ mkdir learingit**
- 进入新目录 **$ cd learingit**
- 通过git init命令将目录变成Git可以管理的仓库 **$ git init(当前目录下面会多一个.git目录，是Git来跟踪管理版本库的，一般来说不需要修改)**

## 3.上传文件

- 编写readme.txt文件，内容不限，要求放在learingit的子目录下面
- git add把文件添加到仓库 **$ git add readme.txt**
- git commit命令提交到仓库 **$ git commit -m "wrote a readme file"(-m之后是对于本次提交的说明)**
- **add命令和commit命令需要配合起来使用，如果每次修改之后不add到暂存区，就不会加入到commit中。**

## 4.状态查看

- 修改readme.txt文件，可以运行 **\*$ git status(掌握仓库当前的状态，readme.txt被修改了但还没有准备提交）***
- 查看指定文件与上一版本的不同 **\*$ git diff readme.txt***

## 5.版本回退

当你把文件改乱了或者误删了文件，还可以最近一个commit恢复。HEAD参数指向的版本就是当前版本。没有"--"参数就会变成了"创建一个新分支"。

* 查看历史记录 **$ git log(显示从最近到最远的提交日志，如果想要精简信息可以加上 --pretty=oneline参数： $ git log --pretty=oneline**

- 回退到指定的版本 **$ git reset --hard HEAD^(当前版本的上一个版本就是HEAD^,上上一个版本就是HEAD^^,往上100个版本就是HEAD~100)**
- 撤销回退命令 当使用 **$ git reset --hard HEAD^**回退到之前的版本，再想恢复到原来的版本的时候使用 **$ git reflog(查看你每一次的命令，每个指令最前面为序列号，如果为3628164，那么恢复的命令：$ git reset --hard 3628164)**

## 6.管理修改

- 提交后可以用**$ git diff HEAD -- readme.txt(查看工作区和版本库里面最新版本的区别)**

## 7.撤销修改

- 直接丢弃工作取得修改 **\*$ git checkout -- readme.txt***
- 如果已经执行过add指令，将文件提交到暂存区 **\*$ git reset HEAD readme.txt(能将修改内容从暂存区返回工作区，而工作区的修改命令 $ git checkout -- readme.txt)***

## 8.删除文件

- 当你直接在文件管理器中把没用的文件删了，或者用rm命令删了，此时工作区和版本库不一致，执行**$ git status** 命令就会立刻告诉你哪些文件被删除了
- 在版本库中删除该文件，执行**\*$ git rm test.txt $ git commit -m "remove test.txt"***
- 通过版本库恢复该文件，执行**$ git checkout -- test.txt（只能恢复文件到最新版本并且会丢失最近一次提交后你修改的内容）**

## 9.远程仓库与本地的连接

- 自行注册Github账号
- 创建SSH Key， **\*$ ssh-keygen -t rsa -C "youremail"***(因为你的本地Git仓库和GitHub仓库之前的传输是通过SSH加密)
- 登陆Github，打开Account settings，“SSH key”点“Add SSH Key”，填上任意的Title，在Key文本框里面黏贴id_rsa.pub文件里的内容（在用户主目录的.ssh目录）

在Github上面创建空的仓库，在本地仓库下运行命令： **$ git remote add origin git @github.com:youGithubName/yourRepo.git**

- 初次将本地库所有内容推送到远程库上 **$ git push -u origin master(加入-u参数，Git不但会把本地得master分支内容推送的远程新的master分支，还会把本地master分支和远程master分支关联起来，在以后的推送或者拉取时就能够简化命令)**
- 下一次提交直接用 **$ git push origin master**
- 从零开发最好方式是先创建远程库，然后，从远程库克隆 **$ git clone git @github.com:youGithubName/yourRepo.git**
- **9.1(针对coding提出修改方式)**
- 在Github上面创建空的仓库，在本地仓库下运行命令： **$ git remote add origin**
- [https://git.coding.net/xxxxxx/xxxxx.git](https://link.zhihu.com/?target=https%3A//git.coding.net/xxxxxx/xxxxx.git) 可能还需要你输入用户名和密码

## 10.分支管理

- 创建dev分支,然后切换到dev分支 **$ git checkout -b dev**
- 查看当前分支 $ git branch **(会列出所有分支，在当前分支前面会标一个\*号)**
- 提交文件到新分支上面 **$ git add readme.txt $ git commit -m "branch test"**
- 切换到master分支 **$ git checkout master**
- 将dev分支合并到master分支上**：$ git merge dev(用于合并指定分支到当前分支)**
- 删除分支 **$ git branch -d dev**