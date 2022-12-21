# What is the meaning of '@' in Makefile

Like below code in a Makefile, without @, it can work too.

```C
all:
    @echo "For correctness test of basic get and put, run: make test;"
```

Without `@`:

```
$ make
echo "For correctness test of basic get and put, run: make test;"
For correctness test of basic get and put, run: make test;
```

With `@`:

```
$ make
For correctness test of basic get and put, run: make test;
```

The difference is clear: with `@`, make will not print out the command that it executes.