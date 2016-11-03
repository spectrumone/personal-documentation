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
psql <new db name> < <outfile>
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

###Backup a db with a password and then restore using Django
```bash
PGPASSWORD=<postgres user password> pg_dump -h127.0.0.1 -Upostgres -Ocx <db name> > /<path>/<name>-`date +%Y%m%d%H%M`.sql
```
  * The default is to perform local peer authentication based on your current username. If you would like to use a password you must specify the hostname with `-h localhost`.
  * `-c` Output commands to clean (drop) database objects before you create or restore the plain text dump.
  * `-x` Prevent dumping of access privileges
  * `-O` To make a script that can be restored by any user, but will give that user ownership of all the objects.
  * `-U` User name to connect as. The password is the password of the user.

```bash
createdb <new db name> #set this db in the django settings.
cat <dump file> | ./manage.py dbshell
```
  * Runs the command-line client for the database engine specified in your ENGINE setting, with the connection parameters specified in your USER, PASSWORD, etc., settings.
  * For PostgreSQL, this runs the psql command-line client.

###Create automated backups
  This is fairly simple. Just run a script or a command in the crontab that does backups. Preferrably there should a script for daily backups and hourly backups. An example of a backup command would be the one db backup line on top.
