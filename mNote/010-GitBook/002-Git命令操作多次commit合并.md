# Git命令操作多次commit合并

---

> 																					       —  2021-12-01      — 于文波

----

> 此处会遇见两种情况：
>
> 情况一：提交的合并请求只有自己的多次commit提交记录；
>
> 情况二：提交的合并请求中，除了自己的多次commit记录外，还有更新下来的merge记录；

### 一、git基本操作命令

#### 具体可以参考：[Git常用命令汇总](https://github.com/ywbo/mwiki/blob/main/mNote/library/010-GitBook/003-Git%20%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4%E6%B8%85%E5%8D%95.md)

我们在处理以上两种情况时，主要用到了以下命令：

```bash
#回滚
git reset
#变基
git rebase 
#拉取
git pull
#推送
git push
```

### 二、commit合并

#### 1. 情况一：提交的合并请求只有自己的多次commit提交记录

这种情况比较简单，处理起来也比较容易。前提保证自己的本地的代码是最新的，即和服务器主仓代码一致。仅自己修改的部分。

效果图，只有自己的提交记录。

![1639708905947](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\1639708905947.png)

> 【思路】
>
> 我们需要使用`git reabse`（变基）命令，将我们提交的多次commit合并为一条。我们有几条提交记录，那么就变基几次。将除第一条pick记录外的其他记录的`pick`修改为`fixup`。然后保存退出，git会自己进行rebase操作，等操作完后，我们需要推送到远程私仓。完成。

【注】这里需要了解下`fixup` 和 `squash` 的区别。`fixup`：会将我们的commit 提交信息进行合并，最终只剩一条信息，不需要自行修改；`squash`：会将我们的commit提交信息进行融合，最终会将我们`rebase`的提交信息记录融合在一起，需要自行修改。

![1639708939388](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\1639708939388.png)

见git官方解释：

```bash
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
```

##### 具体操作如下：

##### ① git rebase -i HEAD~[*]  **提供一个参数，指明你想要修改的提交的父提交（-i 是--interactive(互相配合)的缩写）**

```bash
# 修改最近2次提交，事实上所指的是3次前提交之前，即你想要修改提交的父体交
git rebase -i HEAD~2
```

弹出类似于这样的一个编辑框（图片暂时没有，后续补）

![1639708939388](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\1639708939388.png)

```bash
pick c137cb8 Update README.md
pick e357b54 update host

# Rebase 20e39bf..63936af onto 20e39bf (3 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
```

此时我们修改第二条`pick`记录：

![1639709378618](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\1639709378618.png)

```bash
pick c137cb8 Update README.md
f e357b54 update host

# Rebase 20e39bf..63936af onto 20e39bf (3 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
```

保存退出：`wq!`

##### ② git push -f origin  使用 -f 表示强制推送到自己远程私仓，因为我们的远程私仓仅自己使用，所以可以强制推送，但是如果多人协同的仓库，谨慎使用强制推送哦！

此时刷新远程流水线，只有一条commit记录。

#### 2. 情况二：提交的合并请求中，除了自己的多次commit记录外，还有更新下来的merge记录

如下图：

![img-eg2](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\img-eg2.png)

这种情况稍微复杂一点，但只要理清思路，处理也很容易。

> 【思路】
>
>    	  1. 软合并（--soft）：我们要知道，既然有了Merge记录，就说明我们的本地的版本或者本地的代码已经落后了服务上代码的最新版本。那么此时，我最常用的方式就是将我们本地的代码进行回滚，并且在回滚是时进行软合并，这样保证我们本地修改的代码不会发生回退。然后将将本地代码更新到服务器上的最新代码，并推送到自己的远程库（origin）。此时，本地上次提交的文件和提交记录已经被回退了，我们之前提交记录也一并被回退，这时再此提交自己本地修改的文件，再推送到自己远程仓库，此时就只有一条commit记录。
>    	  2. 硬合并（--head）：我们在进行硬回退时，需要先把本地修改的文件stash（隐藏），`--head`会将本地的已经修改的文件全部回退了，所以我们需要将本地文件stash掉，然后回退到合并人Merge的记录上，再将stash的记录pop（释放）出来，若有冲突，处理后，再提交，此时也会有一条commit记录。

<font color="red">【注】这里我们需要特别注意一个问题，在进行代码回退时，我们必须回退到合并人Merge的记录上。what？为什么要回退到合并人Merge的记录上？</font>

【解答】因为我们在使用`git log`	命令时，显示的所有提交记录，如果我们不回退到某次合并人Merge记录上，却回退到某人`commit`记录上，那么这时会有个问题，我们并不能保证此时我们回退到某人的提交记录已经被合入了，也许他也仅仅是一次提交而已，那么我们回退的代码本地就会携带别人的代码，后续再提交处理就很麻烦了，所以这里我建议还是退回到合并人Merge的记录上来，这样就能保证远程主仓库—本地代码—远程私仓三处的代码是一致的。避免出现其他问题。

具体操作参考如下：

##### ① git status：查看本地文件状态

##### ② git stash：将本地修改文件暂存

##### ③ git reset --soft [commitID]   使用 --soft 表示软合并的方式，保留本地所有更改记录。

```bash
git reset --soft 06b433df889a990aceb7b4d376f09c11334be6b2
```

![img-reset-eg2](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\img-reset-eg2.png)

##### ④ git pull upstream Br_feature_12.0.1

```bash
git pull upstream Br_feature_12.0.1
```

![1639018193618](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\1639018193618.png)

##### ⑤ git push -f origin  使用 -f 表示强制推送到自己远程私仓，因为我们的远程私仓仅自己使用，所以可以强制推送，但是如果多人协同的仓库，谨慎使用强制推送哦！（最好加上远程分支名称）

```bash
git push -f origin Br_feature_12.0.1
```

![1639018244780](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\1639018244780.png)

##### ⑥ git stash pop/ 或在idea中unstash相应的记录，如有冲突，merge处理下；

##### ⑦ git commit(提交本地自己修改的文件)

##### ⑧ git push -f origin  远程分支（master/Br_feature_12.0.1）(强推送到自己远程私仓)

#### ⑦、⑧ 这两条命令就是提交自己本地修改的代码，可以在IDEA中直接提交，推送即可。结果是一样的。

处理完成后，流水线结果：

![1639018482029](D:\ywbWork\StudyNote\三省吾身\mwiki-main\mNote\library\010-GitBook\1639018482029.png)

