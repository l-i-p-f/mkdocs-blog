# Git合并两个代码仓库的commit记录

<br>

## 背景

本地git代码开发完成，想提到远程仓库保存，但事先没在远程仓库内开发，因此存在两份git记录。

如果直接把本地代码**复制粘贴**到拉下来的远程仓库内，只能新增一条全新的git commit记录，导致原来本地代码开发的git commit记录丢失。

网上找了蛮多资料，但该需求材料较少，特写此记录。

<br>

## 步骤

1.对本地代码，添加远程仓库

```
git remote add -f ner [远程仓ssh路径]
```

`git remote add` 添加远程仓库

- `-f` 参数的作用是在添加后立刻fetch

- `ner` 是远程仓库在本地的名称，可任意

![img](images\git_image002.png)

```
# 可以查看当前有哪些远程仓
git remote -v
```

![img](images\git_image003.png)

2.合并代码

```
git merge --strategy ours --no-commit --allow-unrelated-histories ner/master
```

`git merge`为合并分支

- `–-strategy ours` 解析合并，在合并时，如果遇到冲突，以当前库的为准。（在本例中，即以本地库为准）。结果就是：
  - 远程仓库`ner`的`master`分支中的历史记录被合并到本地的历史记录中
  - 远程仓库的文件树被读取并和本地库的文件树比对进行冲突解析

- `–-no-commit` 合并解析完成后中断，停留在最后的提交步骤之前。
  - 只要你还没 commit，那么 merge 的结果就暂时保存在缓存区中，只有完成提交步骤合并才算彻底完成（文件树被正式改变）

- `-–allow-unrelated-histories` 允许合并无关的历史记录。
  - 如果不添加此选项，可能会出现`fatal: refusing to merge unrelated histories`错误

![img](images\git_image004.png)

3.拉取代码

```
git read-tree -u ner/master -m
git status	# 查找当前代码状态
```

`git read-tree` 将树的信息读到索引中

- `-u` 读取后更新index，使working tree与index保持同步
- `-m` 执行合并，而不仅仅是读取。如果你的索引文件中有未合并的条目，该命令将拒绝运行，这表明你还没有完成之前开始的合并。

这个命令的意义在于，之前的`git merge`命令可能会在解决冲突的时候，把远程仓库的文件树弄得比较混乱，使用`read-tree`去修复一下。

![img](images\git_image005.png)

4.提交commit

```
git commit -m [commit信息]
```

可以查看当前commit记录已经合并

```
git log --oneline
```

![img](images\git_image007.png)

`ner/master` 是本地代码仓commit记录，`origin/master` 是远程仓代码记录，此处是特意新建仓库模拟故只有一条commit。



5.提交代码到远程仓

 ```
git push origin master
 ```

<br>

## 参考

1. [《Git/Gitlab进阶》十五：合并两个项目为一个并保留合并前所有历史记录](https://www.jianshu.com/p/f592691062c4)

