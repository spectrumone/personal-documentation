#Django Migrations
<i>advanced stuff that I have to be cautious on doing</i>

###Rolling back to previous migrations
```bash
./manage.py migrate <app name> -l #this will show you the migration logs
./manage.py migrate <app name> 0034 #roll back to a specified migration
```
* Sometimes this will not work as there are migrations that for some reason, can't be rolled back. An example would be data migrations.

###Rerunning a previous migration
```bash
./manage.py migrate <app name> -l
./manage.py migrate --fake <app name> 0010
./manage.py migrate <app name> 0011 #run the migration you want to rerun
./manage.py migrate --fake <app name> 0020 #assume this is the latest migration
```
