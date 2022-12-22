If want to add the `Reviewed-by` tag for several patches, we can use below command.

```Shell
git rebase main -x "git commit --amend --no-edit --trailer \"Reviewed-by: xxx <Joey@abc.com>\""
```