# From Project to Productionized on Heroku

###### Workshop presented by Casey Faist @ PyCon 2019

Updated for the PyCon 2020 Virtual Heroku Workshop, this project demonstrates the principles of 12 Factor Architecture and the basic steps most Django applications need to achieve that in your application for deploying to production. Heroku is our deployment server, but the principles demonstrated are applicable to any deployment schema and are a best practice for building robust, quick-recover applications in the cloud.

This project utilizes the [Getting Started with Python on Heroku](https://github.com/heroku/python-getting-started) application.

>Note: To clone and run this project locally from scratch, check out the related [Getting Started with Python on Heroku](https://devcenter.heroku.com/articles/getting-started-with-python) article on Dev Center.

## Prerequisites

[ ] Signup for [Heroku](https://signup.heroku.com/)
[ ] Install the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli#download-and-install)
[ ] Clone this Repo

## Step 1: Add a .gitignore

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


## Step 6: Heroku Create!

It's finally time! First, make sure that all these changes are checked into git.

Then, then from the root directory, run the following command:

```
heroku create
```

You have an app!

This app will receive your codebase as a slug. It's where all the environment variables get set, all the connections are specified. Once we've pushed your code to your app, you'll be able to scale up dynos - which run those separate processes from above!

Before moving on, go ahead and grab the name of your site - it'll be something like `warm-springs-95291.herokuapp.com`. 

## Heroku Config

Before we push the code up, let's set the config vars (or we'll see errors!)

You can remind yourself what environment variables we need by looking at your `heroku.py` file, but to get this project ready to run, you can use the CLI `heroku config` command.

```
heroku config ALLOWED_HOSTS=your-app-name.com
heroku config DJANGO_SETTINGS_MODULE=dynowiki.settings.heroku
```
For the SECRET_KEY, you'll need to generate a new secret. For this demo, it doesn't matter what it is - it's great to use a secure hash generator, or a password manager's generator. Just be sure to keep this value secure, don't reuse it, and NEVER check it into source code!

```
heroku config SECRET_KEY=YOURSECUREGENERATEDPASSWORD
```
Lastly, Heroku will automatically detect the number of concurrent processes you want to run on each dyno. Depending on the resource usage of your process, this can make each dyno handle more requests more quickly - but for now, let's stick to one process.

```
heroku config WEB_CONCURRENCY=1
```

## Heroku Deploy

Alright. It's all led to this. It is time to [deploy to Heroku!](https://devcenter.heroku.com/articles/deploying-python)

Double check all your changes are checked into git on the master branch, then run:

```
git push heroku master
```
Did you add your changes to another branch? No worries! (And great branch hygiene.) Use this command instead:

```
git push heroku yourbranch:master
```

## Scale Up

Now, [scale up](https://devcenter.heroku.com/articles/getting-started-with-python#scale-the-app) your web [process](https://12factor.net/processes) to 1 dyno:

```
heroku ps:scale web=1
heroku open
```

Tada!! It's a dynowiki!

## Logs

To see your website's activity, you can run the following command:

```
heroku logs --tail
```

## Some Optional Commandline Things

Because of COURSE we want to play with it right?

To make a superuser and create the first article on your wiki:

```
heroku run python manage.py createsuperuser
```

Then login with the credentials you supplied on https://your-app-name.com and make your first article.

Click on the article to edit, and you can even upload a photo! Or can you?

## S3 Buckets

Because Heroku uses an ephemeral system, the media you upload will disappear after your next deploy, config update or daily reboot. 

Heroku has some excellent docs on [how to get set up with S3](https://devcenter.heroku.com/articles/s3), and while preparing for this talk I came across a [thorough guide](https://simpleisbetterthancomplex.com/tutorial/2017/08/01/how-to-setup-amazon-s3-in-a-django-project.html) on how to use another third-party package, `django-storages`.

Another option you have is to use a Heroku Addon like [Bucketeer](https://elements.heroku.com/addons/bucketeer). This removes the headache of managing the AWS bucket setup manually, and automatically adds the AWS config vars to your Heroku environment.

Either way, some code changes are required to make use of them. We won't go into detail on S3 setup here since both services cost money - but we will talk about the changes needed to get one hooked up if we have time.

### When to S3

Until your application needs caching at the level of a CDN, you don't need to host your static files on S3 - they'll be automatically generated and re-collected on each rebuild.

Media files - those uploaded by users or generated by API - are another story, and should be hosted in S3.

### Project Changes for S3

This assumes you're using Bucketeer, but the changes are quite similar if you've configured an S3 bucket yourself.

Example updated from [this Stack Overflow post](https://stackoverflow.com/questions/19915116/setting-django-to-serve-media-files-from-amazon-s3), a nicely straightforward overview.

First, if you look in your `requirements.txt`, you'll see the dependencies `boto3` and [django-storages](https://django-storages.readthedocs.io/en/1.7.1/backends/amazon-S3.html). We'll need both of them to connect to S3. If you're working in your own project, pip install these packages.

Next, add a file `s3utils.py` to your project inside the `myproject` directory - in this project, it should be inside `dynowiki/`

```
from storages.backends.s3boto3 import S3Boto3Storage                                                          

MediaRootS3BotoStorage  = lambda: S3Boto3Storage(location='media')

```

Then, add the following to your `heroku.py` settings file:

```
# S3 Config
INSTALLED_APPS += ('storages',)

AWS_STORAGE_BUCKET_NAME = env('BUCKETEER_BUCKET_NAME')
# Bucketeer requires media files to be in /public
S3_URL = f"http://{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com/public/"
MEDIA_URL = f"{S3_URL}{MEDIA_ROOT}/"
DEFAULT_FILE_STORAGE = 'dynowiki.s3utils.MediaRootS3BotoStorage'
AWS_ACCESS_KEY_ID = env('BUCKETEER_AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = env('BUCKETEER_AWS_SECRET_ACCESS_KEY')
```
