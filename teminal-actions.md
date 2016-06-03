#Terminal Actions
<i>collection of terminal commands I keep forgetting and always look up the internt</i>

###Download from another instance
```bash
rsync -avP <address of instance e.g.: dog.com>:<full path of file or folder to be downloaded> .
```
* this only works if your pem has already been added to the instance `ssh-add`
* `-a` a summary of a bunch of switches that you normally will want
* `-v` verbose
* `-P` to show the progress
