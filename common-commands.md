#Common Commands
<i>collection of terminal commands I keep forgetting and always look up the internt</i>

###Download from another server
```bash
rsync -avP <public address of instance e.g.: dog.com>:<full path of file or folder to be downloaded> .
```
* this only works if your pem has already been added to the instance `ssh-add`
* `-a` a summary of a bunch of switches that you normally will want
* `-v` verbose
* `-P` to show the progress
* 

###Upload to a server
```bash
rsync -avP <full path of file or folder to be uploaded> <public addess of instance>:<upload path>
```
same explanations as above.

###Open the folder of the current path of the terminal (ubuntu)
```bash
nautilus --browser ~/some/directory
```
Rather simple but I keep forgetting.

###Backup and restore dump to a new database (PostgreSQL)
```bash
pg_dump <db name> > <outfile>
createdb <new db name>
psql <new db name> <outfile>
```
This only works if the current user is the correct db owner (If the current ubuntu user has the same name as the db user that owns the postgres db. If not, run `sudo su - postgres` to turn the curent user the  postgres root superuser.

###Backup and restore dump to a new database (Django methods)
```bash
./manage.py dumpdata -o <outfile>
```
assuming youre now on youre new machine with a clean db
```bash
./manage.py migrate
./manage.py loaddata <outfile>
```
This maybe heavily inefficient and buggy since loaddata is, i believe, designed for fixtures only.
