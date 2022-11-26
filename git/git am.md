### How to apply multiple patches?

If there are multiple patches, we should better move it into one directory. And using the

```Shell
$ git am dir/*.patch
```

to apply all patches that in the directory.

Due to the patch in directory will in order by their patch sequence, so we can simply git am all of them.

### What if git am failed?

If git am failed, we should patch it by hand. But the git still can do a lots of work for us, use the following command:

```Shell
$ git am patch --reject
```

Add `--reject` could make git to merge these non-conflict code, if some code has conflict, git will generate a `.rej` file for that file should be patched. Then we should manually modify these file, and then `git add` them. Once all files has been `git add` ok, you can use `git am --continue` to commit this patch and go through.

Another way is using:

```C
git am --3way patch
```

This will try to merge the conflict into the object file, not to generate the `.rej` file, and it can make it easier to merge the conflict.
