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

##### Git的工具

gitk

git自带的图形化工具，可以很方便的看见提交、分支、代码等信息。

tig

命令行式的工具。和git的命令配合使用，基本就满足所有操作了。我一般用来查看提交代码的修改

##### Git merge rebase cherry-pick

git 的merge操作会把两个分支进行合并成一个分支操作。这其实是一个"三方"合并。两个分支会找到
共同祖先的提交节点，再和两个分支现在的节点进行"三方"的代码合并。merge操作很方便，但是会带来分支连的混乱。

git 的rebase　操作就较好的解决了merge操作带来的分支混乱的问题。在从上游分支获取最新commit信息，并有机的将当前分支和上游分支进行合并。能使代码树保持一条线路，从而达到简介。

git rebase xxx　也会带来冲突，这时往往需要解决冲突,　git add .之后，进行git rebase --continue 操作，如果不想git rebase 了，就git rebase --abort进行中断。git rebase 操作后经常需要强push,也就是　git push -f 。这个操作会带来的后果是，会覆盖之前的代码。如果有人的分支和远端分支不一样，被另一个人强推操作到该分支后，很可能会导致其提交的丢失。而merge操作则可以不带来强推导致某个提交丢失的问题。merge的逻辑很简单，合并，有冲突解决冲突，提交代码，形成新的提交，不会把之前的提交丢失。(至少在我的使用中，没出现commit丢失的情况，而rebase出现过)

git cherry-pick 真的是个很好用，很灵活的合并方法。它不是要合并分支，而是合并某次commit的代码。这样就更不会担心某个分支之前代码的不同，而只选择某次commit，取其合并。合并的粒度更小，也许就避免了之前分支中代码的干扰。git cherry-pick　就是这样的作用。
