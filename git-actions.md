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

###Update remote to be the same as local
```bash
git push -f origin <branch name>
```
###Reset local to be the same as origin
```bash
git reset --hard origin/<branch name>
```
* do this after fetch

###Change branch-A to become an exact replica of Branch-B (with history)
```bash
(at branch B)$ git merge --strategy=ours branch-A
(at branch A)$ git merge branch-B
```
This will change branch A to completely loook like B by adding just one merge commit but still with the history of both.
