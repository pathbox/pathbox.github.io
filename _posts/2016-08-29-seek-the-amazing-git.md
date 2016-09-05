---
layout: post
title: Seek the amazing Git
date:   2016-08-29 16:43:06
categories: git
image: /assets/images/top/20160822.jpg
---


Git is amazing! 本文是对git的进一步的学习和掌握总结。

Git是分布式的版本控制管理系统，出自linus主导的团队，它是开源的项目。Git已经得到了世界上大部分软件开发人员的认可，
几乎成为了软件开发团队必不可少的代码版本管理工具。起初对git的了解仅限于命令的使用。现在，对git更深入的学习，让我惊诧于git的设计思想和原理。

##### git的工具

gitk

git自带的图形化工具，可以很方便的看见提交、分支、代码等信息。

tig

命令行式的工具。和git的命令配合使用，基本就满足所有操作了。我一般用来查看提交代码的修改

##### git merge rebase cherry-pick

git 的merge操作会把两个分支进行合并成一个分支操作。这其实是一个"三方"合并。两个分支会找到
共同祖先的提交节点，再和两个分支现在的节点进行"三方"的代码合并。merge操作很方便，但是会带来分支连的混乱。

git 的rebase　操作就较好的解决了merge操作带来的分支混乱的问题。在从上游分支获取最新commit信息，并有机的将当前分支和上游分支进行合并。能使代码树保持一条线路，从而达到简介。

git rebase xxx　也会带来冲突，这时往往需要解决冲突,　git add .之后，进行git rebase --continue 操作，如果不想git rebase 了，就git rebase --abort进行中断。git rebase 操作后经常需要强push,也就是　git push -f 。这个操作会带来的后果是，会覆盖之前的代码。如果有人的分支和远端分支不一样，被另一个人强推操作到该分支后，很可能会导致其提交的丢失。而merge操作则可以不带来强推导致某个提交丢失的问题。merge的逻辑很简单，合并，有冲突解决冲突，提交代码，形成新的提交，不会把之前的提交丢失。(至少在我的使用中，没出现commit丢失的情况，而rebase出现过)

git cherry-pick 真的是个很好用，很灵活的合并方法。它不是要合并分支，而是合并某次commit的代码。这样就更不会担心某个分支之前代码的不同，而只选择某次commit，取其合并。合并的粒度更小，也许就避免了之前分支中代码的干扰。git cherry-pick　就是这样的作用。

##### git reset

git reset 回对commit 进行回滚。或者，你想回滚道某个commit。 接的参数为 --hard。 如果用了--hard参数，表示本地的代码修改都忽略了，这种情况要想想清楚，
会使你的代码修改丢失。如果不加--hard，本地代码修改还在，你需要进行 add 和 commit操作，提交commit。

举例：

git reset origin/master --hard  我放弃了本地分支的代码修改，将本地分支代码回滚为远程的 origin/master 分支代码，你可以认为就是将本地分支代码同步为origin/master的代码
去掉 --hard，则可能会和master的代码有冲突，需要你解决冲突然后进行一次commit操作的提交。

git reset xxx_commit_SHA1 回滚到具体的某次commit

git reset 是一种很方便的同步远程代码的方法，而不会产生merge或rebase这样的合并操作，对于多人开发的时候，想要从某个分支开始开发，可以这样便捷的reset到那个分支后，开始进行开发，不用管本地的分支
