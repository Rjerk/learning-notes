---
title: "同步一个 fork"
tags: git
date: 2018-08-22
---

1. 配置 upstream

查看当前 upstream

```
git remote -v
```

指定新的 upstream

```
git remote add upstream https://github.com/ceph/ceph-iscsi-config.git
```

```
git remote -v
origin    git@github.com:Rjerk/ceph-iscsi-config.git (fetch)
origin    git@github.com:Rjerk/ceph-iscsi-config.git (push)
upstream    https://github.com/ceph/ceph-iscsi-config.git (fetch)
upstream    https://github.com/ceph/ceph-iscsi-config.git (push)
```

2. 从 upstream 取得分支和 commit

```
git fetch upstream
```

本地 checkout master 分支

```
git checkout master
```

3. merge upstream/master 到本地的 master

```
git merge upstream/master
```

4. push 更新

```
git push -u origin master
```
