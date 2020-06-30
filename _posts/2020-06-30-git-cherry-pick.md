---
layout: post
title: "撿寶 git cherry-pick"
tags: version_control
---

情境
1. 新增 want
```
touch want
```
2. 修改 want
```
echo happy > want
```
3. 刪掉 want
```
git rm want
```
歷史如下
```
git log --oneline --name-status
```
![](../../../assets/git/cp_1.png)

撿 modify want
```
git cherry-pick -n 941ab3b
```
若撿到衝突 unmerged path
```
git status
```
![](../../../assets/git/cp_2.png)

後續有幾種選擇
1. 讓 want staged，這個例子```cat want```應該有```happy```
```
git add want
```
2. 把 want 變成 HEAD 狀態，在這個例子就是 D (刪除)
```
git reset want
```
3. 把 want 變成別的 commit 狀態，這個例子```cat want```應該要是空的
```
git add want
git checkout 2d8880e -- want
```
