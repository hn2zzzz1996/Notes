# git send-patch series

Before we send patch to the mailing list, we need to get the maintainer for each patch, and then send it out. But if we manually get maintainer for each patch and then send it. It will be very slow progress, and we want it to be automatic. Then we need the shell script.

The most different patch is the cover-letter, it doesn't have any maintainer, so who will we send the cover-letter to? Actually, we should send it to the mailing list, and not to send it to any maintainer, due to we don't know it should send to who. So we need to scan all of the patch and get it's mailing list and add these mailing list all to the cover-letter. And then send the cover-letter to all the mailing list.

```Shell
    #! /bin/bash
    #
    # cocci_cc - send cover letter to all mailing lists referenced in a patch series
    # intended to be used as 'git send-email --cc-cmd=cocci_cc ...'
    # done by Wolfram Sang in 2012-14, version 20140204 - WTFPLv2

    shopt -s extglob

    name=${1##*/}
    num=${name%%-*}

    if [ "$num" = "0000" ]; then
        dir=${1%/*}
        for f in $dir/!(0000*).patch; do
            scripts/get_maintainer.pl --no-m $f
        done | sort -u
    else
        scripts/get_maintainer.pl $1
    fi
```

Store the file to `~/.cc.sh`.

And then add a function into the `~/.bashrc`:

```C
function kpatch() {
    git send-email \
        --cc-cmd="~/.cc.sh" \
        $@
}
```

How to use?

```Shell
kpatch *.patch
```



## Reference

https://lwn.net/Articles/585782/