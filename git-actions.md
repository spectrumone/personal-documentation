#Git Actions
<i>collection of semi advance stuff you can do with git</i>

###Edit the dates of previously pushed commits
```bash
git rebase -i HEAD~2
```
this will open a text where you will decied what to do per commit. Upon saving, you will be redirected to the terminal given an option whether you want to change the commit or not.
```bash
git commit --amend --date "Thu May 28 18:21:46 2015 +0530"
```
or
```bash
git commit --amend --date=now
```

###Revert local to a previous commit
```bash
git reset --hard <commit hash>
```
this will change all the files back to how they use to be on specified commit. It won't delete files however.
```bash
git commit push -f origin <commit hash>:<branch-name>
```
reverts origin file to previous commit

###Change branch-A to become an exact replica of Branch-B (with history)
```bash
(at branch B)$ git merge --strategy=ours branch-A
(at branch A)$ git merge branch-B
```
This will change branch A to completely loook like B by adding just one merge commit but still with the history of both.


