# Deploy a Django Project on Heroku

## Setup

1. Preparation  
A few good organizational practices will make this entire process much smoother.  
    - Your Django project *must* be in its own git repo.
	- Your Django project *should* be at the root of that repo (The `manage.py` file and hidden `.git` folder should be in the same directory.)
	- Your Django project *should* be developed in a virtual environment with `pipenv` or `venv`.

2. Create a Heroku Account  
Go to [heroku.com](https://www.heroku.com) and create a free account.

3. Download the Heroku CLI  
Follow the instructions at [devcenter.heroku.com](https://devcenter.heroku.com/articles/heroku-cli) to download the Heroku CLI for your operating system.

## Create a new Heroku App
In a terminal, `cd` into the root of your project's git repo.  
Run this command:
```console
$ heroku login
```
The `heroku login` command will:
1. Verify that the Heroku CLI is installed
2. Log into your Heroku account from the terminal  

This should open Heroku in your browser.  Follow the prompts to log in.

#### [Create an app from the command line](https://devcenter.heroku.com/articles/git#for-a-new-heroku-app)
To create your app, run this command, replacing `[your-app-name]` with your app's name:
```console
$ heroku create [your-app-name]
```

### Note:
There is a chance that the name you want is taken.  Try variations of your preferred name until you get one that isn't taken.  
Once deployed, your app will be hosted at `https://your-app-name.herokuapp.com/` but you can set up a custom domain (i.e.: `your-app-name.com`, `your-app-name.net`) if you buy the domain from a DNS (Domain Name System) service like [Namecheap](https://www.namecheap.com) or [Google Domains](https://domains.google.com).

## Heroku & Git
Heroku deploys your projects by building them up from git pushes to a remote repo.  
Run the command `git remote -v`:

```console
$ git remote -v
heroku  https://git.heroku.com/your-app-name.git (fetch)
heroku  https://git.heroku.com/your-app-name.git (push)
origin https://github.com/your-username/your-app-name.git (fetch)
origin https://github.com/your-username/your-app-name.git (push)
```
You should see 4 lines of feedback from this command (two for Heroku, and two for GitHub).  If you only see the `origin` lines, run this command to add the Heroku remote:
```console
$ heroku git:remote -a [your-app-name]
set git remote heroku to https://git.heroku.com/your-app-name.git
```

Run `git remote -v` again to verify that the Heroku remote has been added.

## Python Environments & Deployment

One of the first things that happens after you push a project to Heroku is buildpack detection.  Heroku needs to know:

1. What language is this project in?  (Python)
2. Which dependencies (libraries & packages) does Heroku need to install in order to run your project? (Django, Pillow, DRF, etc.)

There are multiple ways to give Heroku this information, and they each involve a file at the root of your git repo:

### 1) `requirements.txt`
A requirements.txt file contains a list of all of your project's python dependencies.  For example, if you are using Django 3.2.5 and Pillow 8.3.1, your requirements.txt looks like this:
```
django==3.2.5
pillow==8.3.1
```
A requirements.txt file can be generated with the `pip freeze` command.  Be sure you are inside your virtual environment **and** at the root of your repo (where the hidden `.git` folder is), and run this command:
```console
$ pip freeze > requirements.txt
```
This will fill requirements.txt with your Python dependencies (creating the file if needed).

### 2) `Pipfile` and `Pipfile.lock`
If you are using `pipenv` for your virtual environment, install dependencies with `pipenv install` instead of `pip install` to create a Pipfile and Pipfile.lock.  Here is what a Pipfile looks like:

```
[[source]]
url = "https://pypi.org/simple"
verify_ssl = true
name = "pypi"

[packages]
django = "*"
pillow = "*"

[dev-packages]

[requires]
python_version = "3.9"
```

Notice the * in the versioning: `django = "*"`.  That means that a specific version of Django wasn't specified.  The Pipfile.lock, however, does keep track of which version of Django has been used in your development environment, and that is the one which Heroku will install when it builds up your project.

If you've been using `pipenv install` to add each dependency, you shouldn't have to do anything extra (like generating a requirements.txt file).  Make sure that each package you are using is in the `[packages]` section of your Pipfile.  (Any dependencies of those packages will be included in the Pipfile.lock)

## `django-on-heroku` and `gunicorn`

Things are always going to be different in a deployment/production enviroment than in a development environment.  There are 3 main things that don't work the same way in a django project deployed to Heroku:
1. `python manage.py runserver` will no longer be used to run your project.  That command is used for local development.  You will use `gunicorn` to run your project via its wsgi file (more on this later).
2. Similarly, your static files (images, JavaScript, CSS) are served differently in production than in development.
3. Your `sqlite` database will not work on Heroku, at least not in an practical way.  Each time a new version of your app is deployed, a new "dyno" is created, always built up from the git repo.  If you run `python manage.py migrate` on your deployed app, it would make a `db.sqlite3` file, which would only persist until a new dyno is spun up: every time you push updates to Heroku, **OR** every time your Heroku app goes to sleep (very often if you are using the free version).  The solution is to use PostgreSQL, which is very easily done with one of these new dependencies.

In fact, [django-on-heroku](https://github.com/pkrefta/django-on-heroku), is going to make a lot of this very simple.  Add two new dependencies to your project:
```console
$ pipenv install django-on-heroku gunicorn # if you are using pipenv and a Pipfile
$ pip install django-on-heroku gunicorn # if you are using requirements.txt
```

And make one change to your `settings.py`, add these lines to the end of the file:
```py
import django_on_heroku
django_on_heroku.settings(locals())
```

The `django-on-heroku` package and these two lines are going to take care of the database problem and the static file problem, just like that.

## The Heroku Procfile

The Procfile (process file) is a special Heroku file that tells Heroku which web processes it needs to run to serve your app.  So create a Procfile at the root of your repo (where the .git folder is).  Hopefully, this is also where your `manage.py` file is.  
Here, I'm using the `tree` command to see that this demo project has `Pipfile`, `Pipfile.lock`, `Procfile` and `manage.py` files, and a `django_demo` project folder with additional files:

```console
$ tree
.
├── Pipfile
├── Pipfile.lock
├── Procfile
├── django_demo
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── manage.py

1 directory, 9 files
```

To deploy on Heroku, we want to use `gunicorn` to run the `wsgi.py` file inside the project folder.  This will do, in production, what `python manage.py runserver` does in development.  We will declare this as a web process in the Procfile:

```
web: gunicorn django_demo.wsgi
```
Be sure to replace `django_demo` with the name of your project folder.  So, that will run the project, but there's one thing missing.  You'll need to migrate your schema from your `models.py` to your new PostrgreSQL database.  So, you will can a release command to the Procfile:

```
web: gunicorn django_demo.wsgi
release: python manage.py migrate
```
If you have a custom user model, you'll need to make sure your release command migrates the user app (and your user model) first, then migrates your other models. Your Procfile will need to look like this:

```
web: gunicorn django_demo.wsgi
release: python manage.py migrate users && python manage.py migrate
```

Be sure to use `python` here, not `py` or `python3`.  This will migrate everything to your new database (create the tables for your schema).  Note that you don't have to run `makemigrations` as well.  That only needs to be done after you make changes to your Models, and all your migrations are already stored as Python files in your repo.  
### Note:
If you have a different file structure (your `manage.py` and project folder are not in the root of the repo), you can still run these commands.  Let's look at one of those file structures here:

```console
$ tree
.
├── Pipfile
├── Pipfile.lock
└── django_demo
    ├── django_demo
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    └── manage.py

2 directories, 8 files
```

So here `manage.py` **and** the project folder are both inside of another `django_demo` folder.  Luckily, Procfile commands can still work:

```
web: cd django_demo && gunicorn django_demo.wsgi
release: cd django_demo && python manage.py migrate
```

The `&&` can be used to run a series of commands in one line.  So each command will `cd` into `django_demo` before running the web process.

## Push to Heroku

The moment of truth.  With git, add and commit the changes you've made locally.  You can still push to GitHub (origin), but you can also push to a new remote: heroku.
```console
$ git push heroku main
```
This will push your branch to main.  If you are on another branch, you can use an altered command to push that branch *as* main: [https://devcenter.heroku.com/articles/git#deploying-from-a-branch-besides-main](https://devcenter.heroku.com/articles/git#deploying-from-a-branch-besides-main)

When this command runs it will do a few things:
1. Push to the heroku remote
2. Look for a Pipfile or requirements.txt to recognize a Python app.
3. Use that file to install all project dependencies.
4. Run your web command.
5. Run your release command.

If you see your migrations being made in the feedback, you're in pretty good shape.  If things go at least 50% right, you should see something about your app being deployed to Heroku.  It will be at `https://your-app-name.herokuapp.com`.  So go there and see if it *really* worked.

## Troubleshooting

There's a decent chance that something went wrong.  Let's start with the feedback that Heroku provides if there are errors.
1. The output after running `git push heroku main`  
You may see an error in this output.  Errors here usually occur somewhere in the process of building up your app.
2. If your app *is* deployed but you see "Application Error" when you visit the page, follow Heroku's advice and run `heroku logs` in your terminal to get an output of what happened.  These errors are usually problems in the Procfile commands.

### Common problems and how to solve them:
* No buildpack detected: Heroku could not tell that your project was written in Python.  Be sure that your requirements.txt or Pipfile is at the root of the repo.
* Module not found: Usually this means there is more than one buildpack file detected, and Heroku read the one you weren't using.  If you're going the requirements.txt route and still have a Pipfile and/or Pipfile.lock, delete those files.
* No web processes running.  You'll see this after an apparently successful deployment, in the `heroku logs` feedback.  Make sure that your Procfile commands are doing what you think they are.  Double check the paths from the route of your repo to your `manage.py` and `wsgi.py` files.

## Running `manage.py` commands
One thing to note is that you have a brand new Postr\greSQL database with empty tables and no data.  That means no users or superusers.  To create a superuser and use the admin panel on your deployed app, run this command:
```console
$ heroku run python manage.py createsuperuser
```
Follow the prompts to create a superuser just as you would in local development.  The `heroku run` prefix can be used to run any django command, including custom management commands you have written yourself.

## Environment variables
To safely hide api keys and other important information that is not included in the repo, you can store them as environment variables (called config variables on Heroku).  There are two ways to do this.
1. Heroku CLI
If you were to set your django SECRET_KEY to 12345 (not advised), you could run this command:
```console
$ heroku config:set SECRET_KEY=12345
```
2. Heroku Dashboard  
This is a good time to introduce the Heroku Dashboard, where you can do much of the management of your app once it's been deployed.  From the dashboard, click on your app, then click on settings >> reveal config vars.  From here, you can set new key value pairs.

Once you've set a config variable, you can access it anywhere in your code (likely your settings.py) with `os.environ.get('SECRET_KEY)`.  This is also where the URL for your new PostgreSQL database is stored.

## File uploads
If your Django application is something like a photo gallery, where the user uploads files through form submission, you will have to use an external file hosting service.  [Cloudinary](https://cloudinary.com/documentation/django_integration) and [Dropbox](https://www.dropbox.com/developers/documentation/python#tutorial) both have SDKs (software development kits) for integrating with Python projects.
