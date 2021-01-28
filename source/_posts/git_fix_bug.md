---
title: 使用fixup与autosquash命令改变错误提交
date: 2021-01-23 11:28:54
tag: [教程, github, 奇技淫巧]
category: [工具使用]
excerpt: 由于文章引用的代码包含了一个错误，回到过去，修复该错误，并保证目录简洁。
---

## 起因
事情是这样的，为了方便读者能找到文章对应的代码，我每个章节提交后都会打一个tag，然后在文章给上这个tag的地址。有次我在提交前将log换成了断言，结果断言写反了，没有验证就提交发布了。后面经过了一些提交才发现这个问题。如果此时我切到这个tag修复并提交，就会导致和文章的链接对应不了， 所以有什么解决办法呢。
![](/img/blog/wrong_commit.png) 

目前的提交情况：我需要修改tag v0.3中的一个小错误
``` console
$ git log --oneline
aa278c1 (HEAD -> main, origin/main, origin/HEAD) java binder callback
9731e60 (tag: v0.3) cpp binder die
19bda12 cpp/java dumpsys server
77dbf97 cpp/java binder linkToDeath test
0ed8e1c update README and makefile
8cbed3f (tag: v0.2) cpp binder code optimization
de89247 Merge branch 'main' of https://github.com/hqw700/binderdemo into main
8ae91b8 Gen cpp binder file and test
86ee113 cpp binder gen
acd95a8 Create README.md
7e44f1c (tag: v0.1) Gen java binder file and test
b4f22b9 Init aidl file
```

## 使用fixup与autosquash命令改变错误提交
修改这个错误后, 需要 git commit --fixup=`对应commit_id`   
然后 git rebase -i --autosquash `对应commit_id的上一个id`
这样就会自动将fixup的commit合并到对应的  
**注意：**  
认真观察会发现，这之后的所有id都变了，tag也不见了，但commit的信息没变，所以这是一条很危险的命令，会改变历史。squash命令本来就是优化commit信息的。
``` console
$ git add cpp/main.cpp
$ git commit --fixup=9731e60
$ git log --oneline
ab4ca46 (HEAD -> main) fixup! cpp binder die
aa278c1 (origin/main, origin/HEAD) java binder callback
9731e60 (tag: v0.3) cpp binder die
19bda12 cpp/java dumpsys server
77dbf97 cpp/java binder linkToDeath test
0ed8e1c update README and makefile
8cbed3f (tag: v0.2) cpp binder code optimization
de89247 Merge branch 'main' of https://github.com/hqw700/binderdemo into main
8ae91b8 Gen cpp binder file and test
86ee113 cpp binder gen
acd95a8 Create README.md
7e44f1c (tag: v0.1) Gen java binder file and test
b4f22b9 Init aidl file

$ git rebase -i --autosquash 9731e60^
$ git log --oneline
efe605c (HEAD -> main) java binder callback
cd0d752 cpp binder die
19bda12 cpp/java dumpsys server
77dbf97 cpp/java binder linkToDeath test
0ed8e1c update README and makefile
8cbed3f (tag: v0.2) cpp binder code optimization
de89247 Merge branch 'main' of https://github.com/hqw700/binderdemo into main
8ae91b8 Gen cpp binder file and test
86ee113 cpp binder gen
acd95a8 Create README.md
7e44f1c (tag: v0.1) Gen java binder file and test
b4f22b9 Init aidl file
```

## push到远程
因为commit id都变了，push到远程也会提示先pull。pull之后会发现有冲突。
``` console
$ git push origin main
To https://github.com/hqw700/binderdemo
 ! [rejected]        main -> main (non-fast-forward)
error: failed to push some refs to 'https://github.com/hqw700/binderdemo'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.

$ git pull
Auto-merging cpp/main.cpp
CONFLICT (content): Merge conflict in cpp/main.cpp
Automatic merge failed; fix conflicts and then commit the result.
```
所以需要加上-f选项才可以push成功。`git push -f origin main`

## 参考文章
[如何借助fixup与autosquash让Git分支保持整洁](http://lazybios.com/2017/01/git-tip-keep-your-branch-clean-with-fixup-and-autosquash/)
