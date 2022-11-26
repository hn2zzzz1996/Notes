# COMMIT PREFIX CONVENTIONS

* "UPSTREAM:" The commit comes from upstream from a later kernel version, and its original SHA is noted in a "cherry-picked from ..." line at the end of the body of the commit message, before the committer's Signed-off-by line.
* "BACKPORT:" The commit is and "UPSTREAM" commit, as noted above, just had conflicts that needed to be resolved. The conflicts are noted at the end of the body of the commit messages.
* "FROMLIST": The commit is from an upstream mailing list, and is likely to be accepted into upstream, but has not yet landed. The mailing list archive URL to the commit, or Message-ID, is noted at the end of the body of the commit message.
* "ANDROID:, BRILLO:, CHROME:, YOCTO:..." The commit originates from the respective OS specific kernel tree, and is not yet upstream.
* "VENDOR: vendor-name: " The commit comes from a vendor (where "vendor-name" is replaced with the vendor), and contains commits not yet upstream.
* "RFC: " A temporary state where comments are requested before attempting to upstream the commit (after which it would move to "FROMLIST.")