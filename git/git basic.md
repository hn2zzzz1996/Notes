## git add -A 和 git add .的区别

* `git add .` : 提交新文件、已修改的文件和删除的文件
* `git add -A` : 同上
* `git add -u` : 只提交修改和删除的文件，不提交新文件

## delete untracked files

### delete untracked files

```bash
git clean -f
```

### delete untracked files and directory

```bash
git clean -fd
```

### delete files that included in gitignore

```bash
git clean -xfd
```

### preview which files will be deleted

use the `-n` parameter.

## 提交被删除文件的改动

* 直接使用`git rm`命令删除文件，不仅会删除本地文件，还会自动添加改动。
* 当使用shell自身的rm命令删除文件时，执行下面的命令来添加改动：
  * git add -A (--all)
  * 不要执行`git add .` 命令

