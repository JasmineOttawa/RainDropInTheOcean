---
layout: post
title: try a app on Django web framework 
category: Python
tags: [Python]
---

# preparation 1, top 10 python web framework
  https://hackernoon.com/top-10-python-web-frameworks-to-learn-in-2018-b2ebab969d1a
  The top one is  Django + SQLite

# preparation 2, create a ubuntu 16.04 VM on OpenStack environment , IP1
Use customized script to enable root access, on Ubuntu it's ssh, on CentOS it's sshd
```
#!/bin/bash
#setup root access 
sed -i 's/PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
service ssh restart
echo -e "rootpw\nrootpw" | passwd root
```

#step1, install python
after VM startup, it is python 3.5 automatically.  on CentOS7.5 default is 2.7.5. Installation is more smoothy on CentOS.   
root@ubuntu1:~# python3
Python 3.5.2 (default, Nov 17 2016, 17:05:23)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> exit()

If not, it could be installed per https://docs.python-guide.org/starting/installation/
apt-get update 
apt-get install python3.6

#step2, install pip
Installing pip will automatically install the latest version of setuptools. https://pip.pypa.io/en/latest/installing/
```
root@ubuntu1:~# curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
root@ubuntu1:~# python3 get-pip.py
......
Installing collected packages: pip, setuptools, wheel
Successfully installed pip-19.0.3 setuptools-40.8.0 wheel-0.33.1
root@ubuntu1:~# pip -V
pip 19.0.3 from /usr/local/lib/python3.5/dist-packages/pip (python 3.5)
```

#step3, install virtualenv 
virtualenv is a tool to create isolated Python environments. virtualenv creates a folder which contains all the necessary executables to use the packages that a Python project would need.
```
https://docs.python-guide.org/dev/virtualenvs/#virtualenvironments-ref
root@ubuntu1:~# pip install --user pipenv
Successfully installed certifi-2018.11.29 pipenv-2018.11.26 virtualenv-16.4.3 virtualenv-clone-0.5.1
root@ubuntu1:~# pip install virtualenv
Requirement already satisfied: virtualenv in ./.local/lib/python3.5/site-packages (16.4.3)
root@ubuntu1:~# apt install virtualenv
root@ubuntu1:~# virtualenv --version
15.0.1
```

#step4, install django in virtualenv
```
root@ubuntu1:/stage# apt-get update 
root@ubuntu1:/stage# apt-get install python3-venv
root@ubuntu1:/stage# python3 -m venv /stage/.virtualenvs/djangodev
root@ubuntu1:/stage# source /stage/.virtualenvs/djangodev/bin/activate
(djangodev) root@ubuntu1:/stage#

(djangodev) root@ubuntu1: pip install Django
....
Installing collected packages: pytz, django
Successfully installed django-2.1.7 pytz-2018.9
You are using pip version 8.1.1, however version 19.0.3 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
(djangodev) root@ubuntu1:pip install --upgrade pip
(djangodev) root@ubuntu1:python
>>> import django
>>> print(django.get_version())
2.1.7
>>> exit()

```

#step5, First project
https://docs.djangoproject.com/en/2.1/intro/tutorial01/

For first project, let's do some initial setup, auto-generate some code that established a Django project - a collection of settings for an instance of Django, including database configuration, Django-specific options and applications-specific settings.    
```
(djangodev) root@ubuntu1:/stage# django-admin startproject mysite
(djangodev) root@ubuntu1:/stage# tree mysite
mysite          # container for your project, its name doesn't matter to Django, you can rename it to anything you like. 
    manage.py   # a command line utility that lets you interact with this Django project in various ways. further info: https://docs.djangoproject.com/en/2.1/ref/django-admin/
    mysite      # actual Python package for your project. its name is the python package name you'll need to use to import anything inside it. 
        __init__.py  # an empty file tells python that this directory should be considered a python package.  https://docs.python.org/3/tutorial/modules.html#tut-packages
        settings.py  # settins/configurations for this Django project. https://docs.djangoproject.com/en/2.1/topics/settings/
        urls.py      # URL declarations for this project, a "table of contents" of your site, https://docs.djangoproject.com/en/2.1/topics/http/urls/
        wsgi.py      # an entry-point for WSGI-compatible web servers to serve your project. https://docs.djangoproject.com/en/2.1/howto/deployment/wsgi/
 ```

 Run development server
 ```
 (djangodev) root@ubuntu1:/stage/mysite# python manage.py runserver
Performing system checks...

System check identified no issues (0 silenced).

You have 15 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.

March 09, 2019 - 02:48:39
Django version 2.1.7, using settings 'mysite.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

curl http://127.0.0.1:8000/    # on local node,  returns 200   
http://IP1:8000/               # doesn't work from my laptop  

1st try : adding rule in security group for HTTP,     
2 options for remote:  CIDR/security group, what is the difference?   
  
try2 : change port:   
python manage.py runserver 80   
  
try3: to listen on all available public IPs  
python manage.py runserver 0:80  
  
this time http://IP1:80 got response
```
Invalid HTTP_HOST header: '10.12.5.107'. You may need to add '10.12.5.107' to ALLOWED_HOSTS.
.......
You're seeing this error because you have DEBUG = True in your Django settings file. Change that to False, and Django will display a standard page generated by the handler for this status code.
```

In project settings.py file,set ALLOWED_HOSTS like this :   
ALLOWED_HOSTS = ['IP1', 'localhost', '127.0.0.1']   

after changing settings.py, manage.py load the change automatiaclly.    
"The install worked successfully! Congratulations!"   
















