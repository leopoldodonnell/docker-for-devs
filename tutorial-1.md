# Tutorial 1 - Application Environments the Easy Way

Managing multiple languages, frameworks, databases and multiple versions of each of these
can be a real challenge. Even if you've managed to accomplish this, you're not always
confident that you are pulling in all of the correct dependencies.

Container development can help here. Since containers are in many ways isolated from
your workstation, you can setup a clean environment with the exact dependencies you
need to get your work done.

The examples here, demonstrate how you can setup an instant development environment on 
your workstation for *nodejs*, *python*, and *ruby on rails*. This tutorial won't go into
multi-server development, we'll leave this to another tutorial.

In each of the examples, you'll use the default [Dockerhub](https://hub.docker.com)
language/framework container to start up a *bash* shell with access to files underneath
the current working directory and exposing the framework default TPC/IP port so you
can use files from your filesystem within the container and can connect to the container
from your browser.

## Node React

Start a bash shell with Nodejs React development environment using the
default `node` container image. This is a good start for most `node` development and
includes everything you'll need.

```bash
$ docker run -ti --rm -w /share -v $PWD:/share -p 3000:3000 node /bin/bash
```

This docker `run` command is pretty typical for the container development environment
pattern.

- `--rm` tells docker to delete the container after you quit out of the container. There
are times when you won't use this flag, but this is more typical.
- `-ti` tells docker that this is an interactive terminal
- `-w` sets the working directory in the container
- `-v $PWD:/share` mounts the current working directory to `/share` in the container
- `-p 3000:3000` exposes port 3000 (on the right) from inside the container to port
3000 (on the left) exernally.

Try running a few commands from within your container shell...

```bash
$ node --version
$ npm --version
$ yarn --version
```

Create your basic React application and start it up...

```bash
$ npx create-react-app node-react-hello
$ cd node-react-hello
$ yarn start
```

When the application is ready, use the url http://localhost:3000 to open up the default
index page. Update this page by editing `src/App.js` using your local code editor. 
Once you've saved your changes, the browser will refresh itself with the latest changes.

`^c` will exit your server, then enter `exit` at the shell pompt to exit your container
cleanly.

## Django - Python 3.X

Container development for *Django/Python* follows the same pattern. First we'll run through
the default, then we'll switch and test with an older version of *python*

Start a bash shell with a Django Python development environment

```bash
$ docker run -ti --rm -w /share -v $PWD:/share -p 8000:8000 python /bin/bash
```
The only difference here is that you'll be exposing port 8000 as this is the typical
port used for Django development. Feel free to use whatever works for you.

You'll need to install `Dajango`, so install it and test to see what versions you're
using...

```bash
$ pip install django
$ python --version
$ django-admin.py --version
```

Create a Django project and then add an application for Hello World

```bash
$ django-admin.py startproject django_hello
$ cd django_hello
$ django-admin startapp DjangoHelloApp
```

Now you'll need to update a few things to get the application working:

1. Edit `django_hello/settings.py` to add `'DjangoHelloApp'` to `INSTALLED_APPS`
1. Add `from DjangoHelloApp.views import django_hello` to `django_hello/urls.py`
1. Add the route `url(r'DjangoHelloApp/$', django_hello)` to `django_hello/urls.py`
1. Add the view for `django_hello` to `DjangoHelloApp/views.py`

```python
# from django.shortcuts import render
from django.http import HttpResponse

# Create your views here.
def django_hello(request):
    return HttpResponse("Hello World")
```

Now run the server...

```bash
$ python manage.py runserver 0.0.0.0:8000
```

**Note:** the bind address `0.0.0.0` will enable python to expose the port on all available networks in the container

Visit the the url http://localhost:8000/DjangoHelloApp to see your application. Update `DjangoHelloApp/views.py` and refresh
to see your changes.

`^c` will exit your server, then enter `exit` at the shell pompt to exit your container
cleanly.

Next, start a new container with an older version of *python* to see if your code still works...

```bash
$ docker run -ti --rm -w /share -v $PWD:/share -p 8000:8000 python:2.7 /bin/bash
```

Now verify the *python* version, install django and startup the server

```bash
$ python --version
$ pip install django
$ cd django_hello
$ python manage.py runserver 0.0.0.0:8000
```

Visit the the url http://localhost:8000/DjangoHelloApp again and note that it still works! This
isn't always the case and the exercise could be useful for testing that you haven't broken
anything if you're working on code that needs to support multiple language or framework versions.

`^c` will exit your server, then enter `exit` at the shell pompt to exit your container
cleanly.

## Ruby On Rails

In this final example, you'll start up a *Rails* environment using the same pattern with two
small exceptions.

1. the runing container will be named to make it easy to use with other docker commands
1. the container won't be started with the `--rm` flag, so its state will persist accross
starts and restarts. This helps with Rails since you would otherwise lose your installed
dependencies when you exit your container.

Start a bash shell in a named container with a Rails environment

```bash
$ docker run -ti --name my-rails-env -w /share -v $PWD:/share -p 3000:3000 rails /bin/bash
$ rails --version
```

Create and startup a basic rails app...
```bash
$ rails new rails_hello
$ cd rails_hello
$ rails server
```

Check that your server is running using the url http://localhost:3000

Now startup a new terminal shell. You can see what containers are running with the docker
`ps` command.

```bash
$ docker ps
```
With output similar to:

```
CONTAINER ID        IMAGE               COMMAND             CREATED                  STATUS              PORTS        NAMES
7839bdc9b572        rails               "/bin/bash"         Less than a second ago   Up 5 seconds        0.0.0.0:3000->3000/tcp   my-rails-env
```

Most docker commands for containers will either work with the `CONTAINER ID` or the `NAME` which are `7839bdc9b572` and `my-rails-env`
respectively in this example.

Now connect to the named running rails container using
the docker `exec` command, then create a Welcome controller...

```bash
$ docker exec -ti my-rails-env /bin/bash
$ cd rails_hello
$ bin/rails generate conroller Welcome index
```

This will create the controller, the view and will setup a route for the `welcome/index`

Try it in your browser using the url http://localhost:3000/welcome/index

Next update the view to say *Hello World* by editing `views/welcome/index.html.erb`

```html
<h1><i>Hello World!</i></h1>
<p>Find me in app/views/welcome/index.html.erb</p>
```

Refresh your browser and you will see the update. 


Exit from your second container shell using `exit` at the shell pompt to exit your container.

Now you'll use the docker `stop` command to stop your container as if you'd moved on to some other
work, or ended for the day.

```bash
$ docker stop my-rails-env
```

Restarting a stopped container is simple; just use the `start` command. Once this is done, you can
reconnect to your docker container and start your server.

```bash
$ docker start my-rails-env
$ docker exec -ti my-rails-env /bin/bash
$ cd rails_hello
$ rails server
```

Any time you want to stop your work, you can use the `docker stop` command from a terminal outside of
your container. Once you've finished working with your container and have stopped it, you can clean up
with the `rm` command.

```bash
$ docker stop my-rails-env
$ docker rm my-rails-env
```
## Summary

This tutorial covered:

1. Running a container as a shell
1. Mounting local files into a container
1. Exposing a port out of a container
1. Attaching to a container from a second shell
1. Listing, stoping and restarting a container
1. Using different versions of a container to test against different dependencies
