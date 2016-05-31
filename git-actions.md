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
