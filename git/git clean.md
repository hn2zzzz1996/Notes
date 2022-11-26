# git clean

`git clean` is used to remove local untracked files from the current git branch.

If you want to see which files will be deleted, you can use `-n` option before you run the actual command:

```Shell
git clean -n
```

Then you can use `-f` option to delete these untracked file:

```Shell
git clean -f
```

Notice: The `-f` just delete untracked file, listed by `-n` option. It will not delete the directory.

* remove untracked directory and file: `git clean -f -d` or `git clean -fd`
* remove ignored files: `git clean -fX`
* remove ignored and non-ignored files: `git clean -fx`

