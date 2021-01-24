---
title: git-rebase
date: 2020-01-21 14:19:22
tags: git 
---

# git commit 整理小提示


### 问题
发现目前代码提交的时候，有同事其实可以把debug用的commit可以与实际的改动合成一个[虽然更加建议的是用单元测试和TDD来做开发]，但是并没有这样做，导致每次的合入多合入了不少没用的Commit信息。

### 引导作用
希望可以通过这篇小分享可以达到把Git commit log与历史操作的意图能映射起来，能够快速找出可能当时修改的需求是什么，或者方便找出大概是哪些commit修改过哪部分的模块，达到快速定位，而不用全部靠经验来去问，减少沟通成本的开销

### 使用工具
git rebase -i $branch_name

branch_name是需要rebase到的分支，可以理解为提交MR的Target分支即可

### 操作例子

下面会以简单的场景作为例子来展示
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579588371776-631d0113-f187-4d90-9e95-aa0fcd632b25.png#align=left&display=inline&height=151&name=image.png&originHeight=302&originWidth=685&size=28554&status=done&style=none&width=342.5) -->
{% asset_img pic1.png 图1 %}

上图中有三个commit，本质上是对同一个功能进行修改。但是实际上只有最后一个Commit是有意义的，因此我们的目的是通过改写commit把三个commit的commitlog合成一个，并且保存三个commit的改动的代码。

执行命令后会弹出下图界面
```
git rebase -i develop
```

<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579588597962-54bb7d78-3229-499d-9848-865a9834097b.png#align=left&display=inline&height=206&name=image.png&originHeight=412&originWidth=699&size=53435&status=done&style=none&width=349.5) -->
{% asset_img pic2.png 图2 %}

根据我们的目的，应该把前置的东西修改成如下状态，然后用Vim的方法进行保存。
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579588679733-7d55dc8b-0cf9-4a97-b3a6-3f67f8bfb0d0.png#align=left&display=inline&height=206&name=image.png&originHeight=412&originWidth=699&size=53435&status=done&style=none&uid=1579588678281-0&width=349.5) -->
{% asset_img pic3.png 图3 %}
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579588708290-1d2c2288-cadb-4238-b7ce-c021bd1ad1e3.png#align=left&display=inline&height=207&name=image.png&originHeight=415&originWidth=722&size=53502&status=done&style=none&width=361) -->
{% asset_img pic4.png 图4 %}
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579588913048-47254f00-cbd1-472c-b620-e2d3e2eff61b.png#align=left&display=inline&height=207&name=image.png&originHeight=413&originWidth=731&size=54933&status=done&style=none&width=365.5) -->
{% asset_img pic5.png 图5 %}
（之后会对下面的状态会进行补充说明。
此处描述一下操作的目的，由于git的限制，第一个commit必须不能为Fixup的状态，所以我们实际上的操作要变成把第一个commit的commit log 改写，然后把后面两个的commit 保留代码修改，丢弃后两个的commit log )

然后就会rebase中的状态
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579588773915-fa4b12b1-62aa-4276-bfac-af3d2b37b04e.png#align=left&display=inline&height=46&name=image.png&originHeight=92&originWidth=716&size=8185&status=done&style=none&width=358) -->
{% asset_img pic6.png 图6 %}

查看git status
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579589042041-2305da7f-cab6-4cc4-9046-44d792dc8b94.png#align=left&display=inline&height=142&name=image.png&originHeight=284&originWidth=739&size=35631&status=done&style=none&width=369.5) -->
{% asset_img pic7.png 图7 %}

我们应该使用git commit --amend来继续commit的修改
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579589129508-8831ba61-768e-4bee-9e29-c4e1dd3a52f2.png#align=left&display=inline&height=177&name=image.png&originHeight=354&originWidth=750&size=43570&status=done&style=none&width=375) -->
{% asset_img pic8.png 图8 %}

对其进行修改，我们把红色圈圈的部分修改为下图（实际上即把第三个commit log 的msg作为修改值），然后保存
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579589189389-93d13df5-93e6-4771-aefd-c185a388a6c8.png#align=left&display=inline&height=169&name=image.png&originHeight=338&originWidth=758&size=39336&status=done&style=none&width=379) -->
{% asset_img pic9.png 图9 %}

然后继续
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579589215414-8280d103-70eb-490d-b464-f11d7328bf68.png#align=left&display=inline&height=35&name=image.png&originHeight=70&originWidth=756&size=10327&status=done&style=none&width=378) -->
{% asset_img pic10.png 图10 %}

查看git log 
<!-- ![image.png](https://cdn.nlark.com/yuque/0/2020/png/554199/1579589254771-7c813ced-058b-4fb7-aa7e-1e6c26379426.png#align=left&display=inline&height=204&name=image.png&originHeight=407&originWidth=729&size=49500&status=done&style=none&width=364.5) -->
{% asset_img pic11.png 图11 %}

我们成功把三个commit合成了一个commit，并且把commit log改写成了我们想要的样子

### Git rebase 参数讲解
<!-- ![](https://cdn.nlark.com/yuque/0/2020/png/554199/1579588913048-47254f00-cbd1-472c-b620-e2d3e2eff61b.png#align=left&display=inline&height=413&originHeight=413&originWidth=731&status=done&style=none&width=731) -->
{% asset_img pic12.png 图12 %}

pick 代表的是直接复用commit
edit  使用commit，但是需要commiter手动修改commit msg 
reword  使用commit，并且使用在此处修改后的commit msg （ 可以理解为edit + git commit --amend的连击操作）
squash  使用commit，并且把commit msg 直接追加到上一个commit ,如果上图中第二个改写为s，出现的最终commitlog应该为(注意生成结果是一个commit）

```
add: translation in output
debug: add log run success
```

fixup  使用commit，并且丢弃其commit msg ，保留代码的变动
exec  执行其他的shell命令
break 可以理解为打断点，就如果对某个commit的改动不太确定可以通过break停在某个commit上，使用--continue即可继续rebase的流程
drop  丢弃commit
label  对此commit打标签
reset  把head重置到此，这里的reset 是软reset，即此commit后的修改都会保留到git 工作区中，不会消失
merge 直接merge其他的分支或者commit

### 操作流程整理

1. 拉取最新的TargetBranch
1. 切换到开发分支
1. 执行rebase -i操作 ，根据实际情况来改写commit

### PS

1. 对于是从最新的TargetBranch通过git flow 创建出来的分支也是可以使用rebase的命令来整理commit的


