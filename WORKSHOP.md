# From Project to Productionized on Heroku

###### Workshop presented by Casey Faist @ PyCon 2019

Updated for the PyCon 2020 Virtual Heroku Workshop, this project demonstrates the principles of 12 Factor Architecture and the basic steps most Django applications need to achieve that in your application for deploying to production. Heroku is our deployment server, but the principles demonstrated are applicable to any deployment schema and are a best practice for building robust, quick-recover applications in the cloud.

This project utilizes the [Getting Started with Python on Heroku](https://github.com/heroku/python-getting-started) application.

>Note: To clone and run this project locally from scratch, check out the related [Getting Started with Python on Heroku](https://devcenter.heroku.com/articles/getting-started-with-python) article on Dev Center.

## Prerequisites

[ ] Signup for [Heroku](https://signup.heroku.com/)
[ ] Install the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli#download-and-install)
[ ] Clone this Repo

## Step 1: Update the .gitignore

This will enable you to smoothly run your application locally and in production with minimal headache.

In addition to whatever you'd normally place in your .gitignore, be sure to [untrack](https://git-scm.com/docs/git-rm#Documentation/git-rm.txt---cached) the following files.

```
/your-venv-directory        # Learn how to create a local environment in LOCALSETUP.md
__pycache__
db.sqlite3                  # not needed if you're using Postgres locally
yourapp/static/
```

>note: Don't add your migrations to .gitignore! We're not covering any app 
migrations here, but your migrations should live in your source control - see the [Django docs](https://docs.djangoproject.com/en/2.2/topics/migrations/#the-commands) for more details.

Commit this and untrack files as necessary, and you're good to go.

## Step 2: Modularize your settings

### 2.a - Settings folder

This step is pretty straight forward, but can lead to errors if you forget to update all the references. 

In the same directory that your `settings.py` file is located, create a new
directory called `settings/`. Place your `settings.py` file into that new dir and rename it - I chose `base.py`, but you can use `dev.py` or whatever makes sense to you.

### 2.b

Now navigate to your `manage.py`. Somewhere around the 6th line, you'll see the main process setting the default `DJANGO_SETTINGS_MODULE` to `yourproject.settings`. Update that to `yourproject.settings.base`, or whatever you named your local settings file.

### 2.c

Navigate to `wsgi.py`, located in `yourproject/`. You'll see a similar line there setting the default `DJANGO_SETTINGS_MODULE`; change it the same way that you changed `manage.py`.

>If you set the project up to run locally and have issues with the settings module after this step, you can use an untracked `.env` to set the `DJANGO_SETTINGS_MODULE` variable in your local environment.

## 3 Changes to settings/base.py

For twelve factor deployment and deployment on Heroku, a few changes are needed in your `base.py` file.

### Whitenoise

[Whitenoise](http://whitenoise.evans.io/en/stable/django.html#using-whitenoise-with-django) is a package for managing static assets. If you check, you can see it in our `requirements.txt` file, so it'll automatically get installed when we deploy to Heroku thanks to the Python Buildpack - but we still have to install it as middleware in our Django application.

Scroll to the `MIDDLEWARE` list and place the following line at index 1, or 2nd in the list:

`'whitenoise.middleware.WhiteNoiseMiddleware',`

The order that middleware is loaded is important, which is why we need to add this change to our local.py file. More on that in a bit.

For local-prod parity, let's enable WhiteNoise's [caching and compression](http://whitenoise.evans.io/en/stable/django.html#add-compression-and-caching-support) locally by adding the following line to the bottom of your `local.py`:

`STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'`

## Step 4: heroku.py

We're splitting out our local and production settings, and now it's time to add a file specifying our production settings. I named this file `heroku.py` since that's where I'll be deploying, but `prod.py` works just as well.

To successfully deploy on Heroku, you can copy paste the following into your `heroku.py` file:

```
"""
Production Settings for Heroku
"""

import environ

# If using in your own project, update the project namespace below
from gettingstarted.settings.base import * 

env = environ.Env(
    # set casting, default value
    DEBUG=(bool, False)
)
# # reading .env file
# environ.Env.read_env()

# False if not in os.environ
DEBUG = env('DEBUG')

# Raises django's ImproperlyConfigured exception if SECRET_KEY not in os.environ
SECRET_KEY = env('SECRET_KEY')

ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

# Parse database connection url strings like psql://user:pass@127.0.0.1:8458/db
DATABASES = {
    # read os.environ['DATABASE_URL'] and raises ImproperlyConfigured exception if not found
    'default': env.db(),
}
```

You'll see on line 8 we're importing our `gettingstarted.settings` config - make sure to update `base` to whatever you named your first settings file.

### 4.a Django-Environ

Django-Environ, installed and imported as `environ`, is a third party package that enables you to easily pull config variables from your environment. You'll see it listed in the `requirements.txt` file, which means it'll get installed automatically on Heroku. To learn more about it's features, you can check out [the django-environ docs](https://django-environ.readthedocs.io/en/latest/)

### 4.b Database connection

The `settings.py` file that Django automatically generates for you utilizes a `sqlite3` database, but that's not ideal for production because data stores should be treated as attached resources. Using the sqlite database means that all your data will disappear on each deploy and dyno restart!

Luckily, Heroku automatically provisions you a Postgres database and supplies it's connection via config var when you create a new Heroku app. Our friend `environ` can pull the database environment variable that Heroku automatically gives so that our application can permanently store data with minimal fuss.

### 4.c Speaking of Databases

You'll also see in our `requirements.txt` file that we have [`psycopg2`](http://initd.org/psycopg/docs/). This is a helper package and while it requires no code setup, it's crucial to include (and keep up to date) to use the Postgres databases supplied by Heroku.

### 4.d Speaking of Dependencies

A key factor of a 12 Factor App is that the dependencies needed for the application are explicitly declared. We've done that with two files. One is the standard Pip dependency file `requirements.txt`, which we've already looked at.

The other is `runtime.txt`, here specifying `python-3.8.2`. This tells the Heroku Python Buildpack to install that specific version of Python 3 into your application's environment before the application is built. You can leave out this file - but the buildpack would then install the default version. The default version installed is documented on [Devcenter](https://devcenter.heroku.com/articles/python-support#specifying-a-python-version).

## 5 The Procfile

Our application's settings have been configured, but we still need to pass some information to our environment. In 12 Factor Architecture, we want to execute our codebase as one (or more) processes.

Heroku will automatically start these processes for us - but we have to tell it which ones using the `Procfile`.

Create a file named `Procfile` at your project's root directory (the same level as your `manage.py` file) and place the following in it:

```
release: python3 manage.py migrate
web: gunicorn gettingstarted.wsgi --preload --log-file -
```

There! We've defined two processes. 

### The Release phase

The [release phase](https://devcenter.heroku.com/articles/release-phase) occurs after the build but before the application's other workers are booted. This is a great place for maintenance tasks that should be done each time the application is re-deployed, like our `migrate` task.

### The Web Process

This process is what actually serves our application. [Gunicorn](https://devcenter.heroku.com/articles/python-gunicorn) is a WSGI Server package, you'll see it declared in our `requirements.txt` file. The command specified here is pretty basic - you can specify max connections, number of workers and a host of other features and Heroku will run them automatically. Check out the [Gunicorn docs](https://docs.gunicorn.org/en/latest/settings.html) for more settings options that your application likely does not need.

### What about the Worker Process?

There's another process type that's important to mention - the Worker Process. Like the Web process, this creates a persistent,scalable number of dynos with resources to run a process. Emails, queues and other background jobs should go on Worker processes, and you can define more than one Worker process in your Procfile. We won't be using them today, but they're an essential tool in any Django developer's toolkit.


## Step 6: Heroku Create!

It's finally time! First, make sure that all these changes are checked into git.

Then, then from the root directory, run the following command:

```
heroku create
```

You have an app!

This app will receive your codebase as a slug. It's where all the environment variables get set, all the connections are specified. Once we've pushed your code to your app, you'll be able to scale up dynos - which run those separate processes from above!

Before moving on, go ahead and grab the name of your site - it'll be something like `enigmatic-taiga-57440.herokuapp.com`. 

## Step 7: Heroku Config

Before we push the code up, let's set the config vars (or we'll see errors!)

You can remind yourself what environment variables we need by looking at your `heroku.py` file, but to get this project ready to run, you can use the CLI `heroku config` command.

```
heroku config:set ALLOWED_HOSTS=enigmatic-taiga-57440.herokuapp.com
heroku config:set DJANGO_SETTINGS_MODULE=gettingstarted.settings.heroku
```
For the SECRET_KEY, you'll need to generate a new secret. For this demo, it doesn't matter what it is - it's great to use a secure hash generator, or a password manager's generator. Just be sure to keep this value secure, don't reuse it, and NEVER check it into source code!

```
heroku config:set SECRET_KEY=YOURSECUREGENERATEDPASSWORD
```
Lastly, Heroku will automatically detect the number of concurrent processes you want to run on each dyno. Depending on the resource usage of your process, this can make each dyno handle more requests more quickly - but for now, let's stick to one process.

```
heroku config:set WEB_CONCURRENCY=1
```

## Step 8: Provision a Database

The last thing we need to do is provision a database. If you run:

```
heroku addons
```

And see:

```
No add-ons for app enigmatic-taiga-57440
```

This means we need to add the database on to our Heroku app. Let's create a free Postgres database with the following command:

```
heroku addons:create heroku-postgresql:hobby-dev
```

And you'll see Heroku spin up and attach the resource for you:

```
Creating heroku-postgresql:hobby-dev on ⬢ enigmatic-taiga-57440... free
Database has been created and is available
 ! This database is empty. If upgrading, you can transfer
 ! data from another database with pg:copy
Created postgresql-clean-67950 as DATABASE_URL
Use heroku addons:docs heroku-postgresql to view documentation
```

## Step 9: Heroku Deploy

Alright. It's all led to this. It is time to [deploy to Heroku!](https://devcenter.heroku.com/articles/deploying-python)

Double check all your changes are checked into git on the master branch, then run:

```
git push heroku master
```
Did you add your changes to another branch? No worries! (And great branch hygiene.) Use this command instead:

```
git push heroku yourbranch:master
```

## Step 10 Scale Up

Now, [scale up](https://devcenter.heroku.com/articles/getting-started-with-python#scale-the-app) your web [process](https://12factor.net/processes) to 1 dyno:

```
heroku ps:scale web=1
heroku open
```

Tada!! You've done it!

## Logs

To see your website's activity, you can run the following command:

```
heroku logs --tail
```

There's much more you can do with your application from here - but hopefully this is enough to get you started! We hope you've enjoyed this demo :)