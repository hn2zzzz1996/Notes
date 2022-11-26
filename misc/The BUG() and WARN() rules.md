# The BUG() and WARN() rules

There is a brain stone in the mm mailing list that when David send a patch where it's message-id is 20220808073232.8808-1-david@redhat.com. The Linus replied don't use the BUG_ON()  macro, due to it's not means to DEBUG, it means a BUG happen, and will panic the system. In a production kernel, that's Linus don't want to see.

So after several days later, the David write these discussion into a document, and send it to the mailing list, the message-id is 20220824163100.224449-1-david@redhat.com. And the patch name is:

> coding-style.rst: document BUG() and WARN() rules
>
> https://lore.kernel.org/lkml/20220824163100.224449-1-david@redhat.com/

And in this patch, David said:

We should remove most BUG_ON() instances (except for those who actually want panic happen, and want to dump the kernel information) and replaced them by WARN_ON_ONCE().

And at the same time, we also want to WARN_ON_ONCE() panic in some cases to capture a proper system dump instead of continuing, and with a completely broken system where is hard to extract any useful debug information. In this case, We'd have to enable `panic_on_warn`. And only do that in case kdump is actually armed after boot.

We not have `crash+kdump` even on most harmless WARN_ON(), that's we don't want to see. So, we need some kind of levels of warning:

1. This is harmless and we can easily recover, but please tell the developers.
2. This is real bad and unexpected, capture a dump immediately instead of trying to recover and eventually failing miserably (非常不幸的).

```diff
This resulted in a more generic discussion about usage of BUG() and
friends. While there might be corner cases that still deserve a BUG_ON(),
most BUG_ON() cases should simply use WARN_ON_ONCE() and implement a
recovery path if reasonable:

    The only possible case where BUG_ON can validly be used is "I have
    some fundamental data corruption and cannot possibly return an
    error". [2]
```

```diff
Now, that said, there is one very valid sub-form of BUG_ON():
    BUILD_BUG_ON() is absolutely 100% fine.
```

```diff
While WARN will also crash the machine with panic_on_warn set, that's
exactly to be expected:

    So we have two very different cases: the "virtual machine with good
    logging where a dead machine is fine" - use 'panic_on_warn'. And
    the actual real hardware with real drivers, running real loads by
    users. [3]
```

[1] https://lkml.kernel.org/r/CAHk-=wiEAH+ojSpAgx_Ep=NKPWHU8AdO3V56BXcCsU97oYJ1EA@mail.gmail.com
[2] https://lore.kernel.org/r/CAHk-=wg40EAZofO16Eviaj7mfqDhZ2gVEbvfsMf6gYzspRjYvw@mail.gmail.com
[3] https://lore.kernel.org/r/CAHk-=wgF7K2gSSpy=m_=K3Nov4zaceUX9puQf1TjkTJLA2XC_g@mail.gmail.com



### Why try to get rid of BUG_ON()

The Linus explained why we want to get rid of the `BUG_ON()`, the original message is 

> CAHk-=wit-DmhMfQErY29JSPjFgebx_Ld+pnerc4J2Ag990WwAA@mail.gmail.com

The linus said only the people who have million machines will want to kill a machine and have people to look and analysis the logs. And for other who only have one machine, they will never want to kill the machine. 

So `WARN_ON_ONCE()` is the thing to aim for. `BUG_ON()` is the thing for "oops, I really don't know what to do, and I physically **cannot** continue" (and that is not I'm too lazy to do error handling).

And for the people who really want the machine to be `panic-and-reboot`, they can turn the `WARN_ON()` becomes `PANIC()`.

In summary, For a kernel developer, we should not say "panic" or "kill the machine" in the code. **It's the choice who maintain the machine**. It depends on the maintainer if he wants to kill the machine.