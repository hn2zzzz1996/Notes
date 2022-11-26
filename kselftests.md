# kselftests

Compile the kselftests:

```shell
$ make -C tools/testing/selftests
```

We can only compile a single subsystem:

```Shell
$ make -C tools/testing/selftests TARGETS="kvm"
```

And can specify multiple tests to build and run:

```Shell
$ make TARGETS="kvm size" kselftest
```



