



\1. vi .git/hooks/pre-commit

\2. edit like below:

*#!/bin/bash*

*exec git diff --cached | scripts/checkpatch.pl --no-signoff*

*# just show log from checkpatch, but not block commit*

*exit 0*

\3. chmod 755 .git/hooks/pre-commit