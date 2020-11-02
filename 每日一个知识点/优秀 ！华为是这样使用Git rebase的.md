

使用git参与多人之间的合作开发大概有三年的时间，大多数场景下使用的git命令一只手多一点就能数的过来

- git add
- git commit
- git push
- git merge
- git pull
- git log



理论上来说，只要能合理管理项目分支，这几个命令已经足以应付所有的日常开发工作。但是如果我们偶尔看一下自己的git graph，我的天呐，为什么会这么乱。



鉴于分支管理的混乱（或者根本就没有进行过分支管理），我们经常遇到一些意想不到的问题，因此需要使用很多面生的git命令来解决我们的问题，比如说本文讲到的git rebase。



# git rebase 和 git merge 区别

Git rebase 的中文名是变基，就是改变一次提交记录的base。在这一环节，我们不妨带着这样一个假设：git rebase ≈ git merge，并用两种命令实现同一工作流来对比他们之间的异同。



回想我们日常的工作流，假设a和b两人合作开发，三个分支：develop, develop_a, develop_b。两个人分别在develop_a和develop_b分支上进行日常开发，阶段性地合入到develop。



那么从a的角度来看，可能的工作流是这样的：

（1）个人在develop_a分支上开发自己的功能

（2）在这期间其他人可能不断向develop合入新特性

（3）个人功能开发完毕后通过merge 的方式合入别人开发的功能



#### git merge

<img src="/Users/luqiang/Downloads/公众号图片/关于git/g1.png" alt="img" style="zoom:50%;" />

上图为日常merge 工作流，对应的git操作命令如下：

```
git checkout develop_a

// 本地功能开发...

git pull origin develop = git fetch origin develop + git merge develop复制代码
```





#### git rebase

同样走完这样一个工作流如果我们使用git rebase来实现，结果如下：



**git rebase 之前，如图：**

<img src="/Users/luqiang/Downloads/公众号图片/关于git/g2.png" alt="img" style="zoom:50%;" />

**git rebase 之中，如图：**

<img src="/Users/luqiang/Downloads/公众号图片/关于git/g3.png" alt="img" style="zoom:50%;" />



**git rebase 之后，如图：**

<img src="/Users/luqiang/Downloads/公众号图片/关于git/g4.png" alt="img" style="zoom:50%;" />

 

git rebase 操作对应命令如下：

```
git checkout develop_a

// 本地功能开发...

git fetch origin develop

git rebase develop

git checkout develop

git merge develop_a

git br -d develop_a 复制代码
```



由此可见，git rebase 和git merge的异同之处如下：

（1）两者都可以用于本地代码合并

（2）git merge 保留真实的用户提交记录，且**在merge时会生成一个新的提交**

（3）git rebase 会改写历史提交记录，这里的改写不仅限于树的结构，树上的节点的commit id也会别改写，因此图3和图4用e'代表图2的e'，收益是可以保证提交记录非常清爽



# 如何使用git rebase -i 修改历史提交记录



git rebase -i，中文名叫交互式变基。意思就是在变基的过程中是可以**掺入用户交互**的，通过交互过程我们可以主动改写历史提交记录，包括修改、合并和删除等。我们以上面使用rebase后得到的提交记录为例，来进行历史提交记录的修改，在修改之前，提交记录是这个样子的。



![img](/Users/luqiang/Downloads/公众号图片/关于git/g5.png)



使用git rebase -i 修改历史提交的过程主要包含三步：

（1）列出一个提交记录的范围，并指出你在这个范围内需要对哪些记录进行什么样的修改

（2）以次执行上述的修改，如果遇到冲突需要解决

（3）完成rebase 操作



以上面截图中的提交记录为例，来对历史提交的commit msg进行修改，操作步骤如下：

```
// 查看最近6次提交记录，选择对哪一条记录进行修改

git rebase -i HEAD~6复制代码
```

![img](/Users/luqiang/Downloads/公众号图片/关于git/g6.png)



执行完上述命令后，会以vim的方式打开一个文件，文件中显示了最近6次的提交信息，从上到下，由远到近。



从下面的注释可以看到，我们分别把每一行前面的pick修改成r, s, d的方式就可以实现对历史记录的修改，合并和删除。

首先我们尝试修改提交信息，把第二行前面的pick改成r，保存退出。当前页面关闭的同时会打开一个新的页面，让你对选中的提交信息进行编辑。



![img](/Users/luqiang/Downloads/公众号图片/关于git/g7.png)





编辑完信息之后保存退出，就完成了对历史提交记录的修改。通过观察下图可以发现，develop_a的提交记录中的commit msg 仍然是feat_c，但是develop 分支中对应的提交记录，commit msg 已经变成了feat: c-update.。



这里需要留意到的一个现象是**develop 和develop_a 分支上相同提交的commit id 已经发生了变化**，这个在后面会再次提到。



![img](/Users/luqiang/Downloads/公众号图片/关于git/g8.png)



除了修改提交的commit msg 之外，我们也可以通过把pick 改为e，结合git reset --soft HEAD^ 的方式对档次提交的改动内容进行修改。



合并与删除历史提交的操作步骤与编辑类似，只需要把pick分别改为s 和d 即可，各位看官可以自行尝试。如果在rebase的过程中遇到了冲突，需要手工解决，然后使用git rebase --continue 完成rebase 操作。

git rebase 的提示还是非常友好的，它会告诉你需要进行哪些操作解决当前的问题。

![img](/Users/luqiang/Downloads/公众号图片/关于git/g9.png)



# 使用git rebase -i 必须遵循的规则是什么？



从修改历史提交记录这个功能来看，交互式变基是一个非常强大的功能。但是使用这个功能必须要遵循一个铁则：**不要对线上分支的提交记录进行变基！**



引用git 官方指导文档的话来说大概是这样：

> 如果你遵循这条金科玉律，就不会出差错。 否则，人民群众会仇恨你，你的朋友和家人也会嘲笑你，唾弃你。



在说为什么不能对线上提交执行交互式变基之前，先说一下如果要对线上功能执行这个操作要怎么做。

首先，你需要在自己本地变基成功，然后使用git push -f 强行push 并覆盖远程对应分支，之所以需要执行覆盖式push 是因为如果你不覆盖，当前变基过后产生的新提交会与远程合并，导致你在本地的变基行为失去意义。

因为我们上面提到过，从变基那个节点开始往后的所有节点的commit id 都会发生变化。

同样的原因，即使你使用git push -f 使远程分支发生了变基，如果你的同事的开发分支中还存在你执行变基操作（不论是修改、合并还是删除）时针对的那些分支，那么当你的同事merge 你的提交之后，你所有想使用变基改变的东西都回来了！



# 如果打破了git rebase -i 的使用规则应该如何补救



此处我们尝试通过要点描述的方式，说明线上提交执行变基会导致什么结果以及如何避免这个结果：

（1）你在本地对部分线上提交进行了变基，这部分提交我们称之为a，a在变基之后commit id 发生了变化

（2）你在本地改变的这些提交有可能存在于你的同事的开发分支中，我们称之为b，他们与a的内容相同，commit id 不同

（3）如果你把变基结果强行push 到远程仓库后，你的同事在本地执行git pull 的时候会导致a 和b 发生融合，且都出现在了历史提交中，导致你的变基行为无效

（4）我们想要的是你的同事拉取线上代码时跳过对a 和b 的合并，只是把他本地分支上新增的修改合并进来

讲了这么多，最终的结论就是，使用变基解决变基带来的问题。即你的同事使用git rebase 的方式把他本地的修改rebase 到远程你执行过rebase 的分支上。



简言之，就是你的同事使用git pull --rebase 而不是git pull 来拉取远程分支。在这个操作的过程中，git 会对我们上面提到几个要点的信息进行检查并把真正属于同事本地的修改合入远程分支的最后。

文字描述可能有些乏力，更多详细信息可以参考这里：[git-scm.com/book/zh/v2/…](https://www.mdeditor.tw/jump/aHR0cHM6Ly9naXQtc2NtLmNvbS9ib29rL3poL3YyL0dpdC0lRTUlODglODYlRTYlOTQlQUYtJUU1JThGJTk4JUU1JTlGJUJB)



# 所以我们应该如何使用git rebase



鉴于上面描述的git rebase 可能带来的问题，最后要回答的一个问题是我们应该如何在日常工作中使用git rebase，同样借用git 官方文档中的一句话：



> 总的原则是，只对尚未推送或分享给别人的本地修改执行变基操作清理历史， 从不对已推送至别处的提交执行变基操作，这样，你才能享受到两种方式（rebase 和merge）带来的便利。

 

 

原文地址：https://www.mdeditor.tw/pl/pMHD

作者：DevUI 华为团队