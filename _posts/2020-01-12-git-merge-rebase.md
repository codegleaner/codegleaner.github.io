---
layout: post
title: "併回 master branch"
---

建立master branch，與a、b、c commit

![](../../../assets/git/master_br.png)

從b commit 切出debug branch，繼續建立C、D commit 

(debug branch後來增加的commit故意用大寫區別)

![](../../../assets/git/debug_br.png)

合併回master branch有兩種方式：Rebase 與 Merge

### Rebase
```
git checkout debug
git rebase
```
解決衝突
```
git add --all
git rebase --continue
```
解決衝突
```
git add --all
git rebase --continue
```
.
.
.
直到沒有衝突

debug branch 的C、D commit 接到master branch 最後的c commit 之後，不再是從b commit 開始接
![](../../../assets/git/rebased.png)

### Merge
```
git checkout master
git merge debug
```
解決衝突
```
git add --all
git commit -m "新增的合併節點"
```

debug branch 仍然是從 master branch的 b commit 開始接，並長出一個圈，歸結在新的節點
![](../../../assets/git/merged.png)
