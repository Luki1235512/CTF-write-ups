# [Committed](https://tryhackme.com/room/committed)

## One of our developers accidentally committed some sensitive code to our GitHub repository. Well, at least, that is what they told us...

Oh no, not again! One of our developers accidentally committed some sensitive code to our GitHub repository. Well, at least, that is what they told us... the problem is, we don't remember what or where! Can you track down what we accidentally committed?

### Discover the flag in the repository!

1. Enumerate all commits in the repository's history, including branches:

```bash
git log --all
```

**Results**

```
commit 28c36211be8187d4be04530e340206b856198a84 (HEAD -> master)
Author: fumenoid <fumenoid@gmail.com>
Date:   Sun Feb 13 00:49:32 2022 -0800

    Finished

commit 4e16af9349ed8eaa4a29decd82a7f1f9886a32db (dbint)
Author: fumenoid <fumenoid@gmail.com>
Date:   Sun Feb 13 00:48:08 2022 -0800

    Reminder Added.

commit c56c470a2a9dfb5cfbd54cd614a9fdb1644412b5
Author: fumenoid <fumenoid@gmail.com>
Date:   Sun Feb 13 00:46:39 2022 -0800

    Oops

commit 3a8cc16f919b8ac43651d68dceacbb28ebb9b625
Author: fumenoid <fumenoid@gmail.com>
Date:   Sun Feb 13 00:45:14 2022 -0800

    DB check
```

2. Inspect the "DB check" commit, which is the earliest one and likely where credentials were first added:

```bash
git checkout 3a8cc16f919b8ac43651d68dceacbb28ebb9b625
cat main.py
```

<img width="1402" height="862" alt="SCREEN01" src="https://github.com/user-attachments/assets/9d70a77f-813a-4027-a9cf-d554f5375b15" />
