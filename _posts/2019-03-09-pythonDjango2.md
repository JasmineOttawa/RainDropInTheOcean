---
layout: post
title: Build a web application using Django web framework - part 2 
category: Python
tags: [Python]
---

# step1. Database setup
By default, the configuration uses SQLite.  view following parameters in mysite/settings.py   
```
ENGINE – 'django.db.backends.sqlite3'    # https://docs.python.org/2/library/sqlite3.html  
NAME – The name of your database. If you’re using SQLite, the database will be a file on your computer;  in that case, NAME should be the full absolute path, including filename, of that file. 
INSTALLED_APPS - holds the names of all Django applications that are activated in this Django instance. Apps can be used in multiple projects, and you can package and distribute them for use by others in their projects. By default, INSTALLED_APPS contains the following apps, all of which come with Django:
django.contrib.admin – The admin site. You’ll use it shortly.
django.contrib.auth – An authentication system.
django.contrib.contenttypes – A framework for content types.
django.contrib.sessions – A session framework.
django.contrib.messages – A messaging framework.
django.contrib.staticfiles – A framework for managing static files.
```

Some of these applications make use of at least one database table, though, so we need to create the tables in the database before we can use them. To do that, run the following command:
```
python manage.py migrate
```

check DB content, https://sqlite.org/cli.html
```
(djangodev) root@ubuntu1:/stage/mysite# apt install sqlite3
(djangodev) root@ubuntu1:/stage/mysite# sqlite3
SQLite version 3.11.0 2016-02-15 17:29:24
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .open db.sqlite3
sqlite> .schema
CREATE TABLE "django_migrations" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "app" varchar(255) NOT NULL, "name" varchar(255) NOT NULL, "applied" datetime NOT NULL);
.....
sqlite> .quit
(djangodev) root@ubuntu1:/stage/mysite#
```

# step2. Creating models
A model is the single, definitive source of truth about your data. It contains the essential fields and behaviors of the data you’re storing. Django follows the DRY Principle. The goal is to define your data model in one place and automatically derive things from it.    
DRY - https://docs.djangoproject.com/en/2.1/misc/design-philosophies/#dry

Here I will create a model to store mapping between short and long URL - mapURL(sURL varchar2(6), lURL varchar2 varchar2(512), created timestamp ) 
```
#tinyurl/models.py
from django.db import models
class mapUEL(models.Model):
    sURL = models.CharField(max_length=6)
    lURL = models.CharField(max_length=512)
	  created = models.DateTimeField('date created')
```
+ Each model is represented by a class that subclasses django.db.models.Model.
+ Each model has a number of class variables, each of which represents a database field in the model.    
+ Each field is represented by an instance of a Field class     

# step3. Activating models
models.py will enable Django to:   
+ Create a database schema (CREATE TABLE statements) for this app.   
+ Create a Python database-access API for accessing Question and Choice objects.   

**Tell our project that the tinyapp is installed**  
Add a reference to configuration class in the INSTALLED_APPS setting. The TinyurlConfig class is in the tinyurl/apps.py file, so its dotted path is 'tinyurl.apps.TinyurlConfig'. 
```
vi mysite/settings.py
INSTALLED_APPS = [
    'tinyurl.apps.TinyurlConfig',
    'django.contrib.admin',
```
There is a typo in model name 
```
(djangodev) root@ubuntu1:/stage/mysite# python manage.py makemigrations tinyurl
Migrations for 'tinyurl':
  tinyurl/migrations/0001_initial.py
    - Create model mapUEL
```
Correct the type, re-migrate
```
(djangodev) root@ubuntu1:/stage/mysite# python manage.py makemigrations tinyurl
Did you rename the tinyurl.mapUEL model to mapURL? [y/N] y
Migrations for 'tinyurl':
  tinyurl/migrations/0002_auto_20190309_1321.py
    - Rename model mapUEL to mapURL
```
By running makemigrations, you’re telling Django that you’ve made some changes to your models (in this case, you’ve made new ones) and that you’d like the changes to be stored as a migration.
Migrations are how Django stores changes to your models (and thus your database schema) - they’re just files on disk. You can read the migration for your new model if you like; it’s the file polls/migrations/0001_initial.py. Don’t worry, you’re not expected to read them every time Django makes one, but they’re designed to be human-editable in case you want to manually tweak how Django changes things.

```
(djangodev) root@ubuntu1:/stage/mysite/tinyurl/migrations# ls -ltr
total 12
-rw-r--r-- 1 root root    0 Mar  9 12:00 __init__.py
-rw-r--r-- 1 root root  624 Mar  9 13:19 0001_initial.py
drwxr-xr-x 2 root root 4096 Mar  9 13:20 __pycache__
-rw-r--r-- 1 root root  319 Mar  9 13:21 0002_auto_20190309_1321.py
```
Note that model name is mapUEL in 001, it's mapURL in 002, looks like it's not an update, it's append     
There is a command that will run the migrations for you and manage your database schema automatically - that's called migrate.   
First, let's see what SQL that migration would run, the sqlmigrate takes migration names and returns their SQL:   
```
(djangodev) root@ubuntu1:/stage/mysite# python manage.py sqlmigrate tinyurl 0001
BEGIN;
--
-- Create model mapUEL
--
CREATE TABLE "tinyurl_mapuel" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "sURL" varchar(6) NOT NULL, "lURL" varchar(512) NOT NULL, "created" datetime NOT NULL);
COMMIT;
(djangodev) root@ubuntu1:/stage/mysite# python manage.py sqlmigrate tinyurl 0002
BEGIN;
--
-- Rename model mapUEL to mapURL
--
ALTER TABLE "tinyurl_mapuel" RENAME TO "tinyurl_mapurl";
COMMIT;
```
You can also run python manage.py check; this checks for any problems in your project without making migrations or touching the database.  
run migrate again to create those model tables in your database: python manage.py migrate   
```
(djangodev) root@ubuntu1:/stage/mysite# python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions, tinyurl
Running migrations:
  Applying tinyurl.0001_initial... OK
  Applying tinyurl.0002_auto_20190309_1321... OK
```
The migrate command takes all the migrations that haven’t been applied (Django tracks which ones are applied using a special table in your database called django_migrations) and runs them against your database - essentially, synchronizing the changes you made to your models with the schema in the database.  
The 3 steps to making model changes:    
1. Change your models (in models.py).
2. Run python manage.py makemigrations to create migrations for those changes
3. Run python manage.py migrate to apply those changes to the database.

# step4. Playing with the API
To invoke Python shell by "python manage.py shell", Because manage.py sets the DJANGO_SETTINGS_MODULE environment variable, which gives Django the Python import path to your mysite/settings.py file  
Explore the database API:  
```
(djangodev) root@ubuntu1:/stage/mysite# python manage.py shell
>>> from tinyurl.models import mapURL    # Import the model classes we just wrote.
Traceback (most recent call last):
  File "<console>", line 1, in <module>
ImportError: No module named 'polls'
>>> from tinyurl.models import mapURL
>>> mapURL.objects.all()                #empty, no URL mapping was created yet 
<QuerySet []>
>>> from django.utils import timezone
>>> q = mapURL(sURL="aaaaaa",lURL="https://jasmineottawa.github.io/RainDropInTheOcean/python/2019/03/08/pythonDjango/",created=timezone.now())
# Save the object into the database. You have to call save() explicitly.
>>> q.save()
# Now it has an ID.
>>> q.save()
>>> q.id
1
>>> q.sURL
'aaaaaa'
>>> q.lURL
'https://jasmineottawa.github.io/RainDropInTheOcean/python/2019/03/08/pythonDjango/'
>>> q.created
datetime.datetime(2019, 3, 9, 14, 16, 29, 949143, tzinfo=<UTC>)
>>> q.sURL="bbbbbb"
>>> q.save
<bound method Model.save of <mapURL: mapURL object (1)>>
>>> mapURL.objects.all()
<QuerySet [<mapURL: mapURL object (1)>]>
```

<QuerySet [<mapURL: mapURL object (1)>]>  isn’t a helpful representation of this object. 
Define a __str__() methods to your models, make reprentation better when dealing with the interactive prompt. objects’ representations are used throughout Django’s automatically-generated admin.
```
#tinyurl/models.py
from django.db import models
from django.utils import timezone   # reference Django’s time-zone-related utilities in django.utils.timezone

class mapURL(models.Model):
    # ...
    def __str__(self):
        return self.lURL
```
Add another methods
```
import datetime     #  reference Python’s standard datetime module, put at the top of models.py, without this one, you will get error: NameError: name 'datetime' is not defined
...
    def was_created_recently(self):
        return self.created >= timezone.now() - datetime.timedelta(days=1)

```

Now see what's change: 
```
(djangodev) root@ubuntu1:/stage/mysite# python manage.py shell
>>> from tinyurl.models import mapURL
>>> mapURL.objects.all()
<QuerySet [<mapURL: https://jasmineottawa.github.io/RainDropInTheOcean/python/2019/03/08/pythonDjango/>]>
>>> mapURL.objects.filter(id=1);
<QuerySet [<mapURL: https://jasmineottawa.github.io/RainDropInTheOcean/python/2019/03/08/pythonDjango/>]>
>>>

>>> mapURL.objects.filter(lURL__startswith='https://jasmineottawa')
<QuerySet [<mapURL: https://jasmineottawa.github.io/RainDropInTheOcean/python/2019/03/08/pythonDjango/>]>
>>> from django.utils import timezone
>>> current_year = timezone.now().year
>>> mapURL.objects.get(created__year=current_year)
<mapURL: https://jasmineottawa.github.io/RainDropInTheOcean/python/2019/03/08/pythonDjango/>
>>> mapURL.objects.get(id=2)
...
tinyurl.models.mapURL.DoesNotExist: mapURL matching query does not exist.
>>>q = mapURL.objects.get(id=1)
>>>q.was_created_recently()
>>> q.was_created_recently()
True
```

# step5. Introducing the Django Admin  
Generating admin sites for your staff or clients to add, change, and delete content is tedious work that doesn’t require much creativity. For that reason, Django entirely automates creation of admin interfaces for models.   
Django was written in a newsroom environment, with a very clear separation between “content publishers” and the “public” site. Site managers use the system to add news stories, events, sports scores, etc., and that content is displayed on the public site. Django solves the problem of creating a unified interface for site administrators to edit content.  
The admin isn’t intended to be used by site visitors. It’s for site managers.  

+ Creating an admin user
+ Start the development server
+ Enter the admin site
+ Make the poll app modifiable in the admin
+ Explore the free admin functionality

