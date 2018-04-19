# Tutorial 2 - Make Development the Same as Production

In *Tutorial 1* you learned how to leverage docker to create development environments
with specific dependencies. The next step is to develop and test deployments
locally that are **EXACTLY** the same as those that will be deployed. 

This next tutorial, will creat a simple *Hello World* python flask web site, test it locally
then deploy it to *Heroko*.

## Container Registries

The mechanism that Docker and other container runtimes use to share and access containers is 
called a *registry*. A *registry* is similar in some respects to *git* in that you can:

- create an artifact locally
- artifacts are uniquely identifiable
- push to a central repository
- pull from a central repository

The docker command uses the following format to identify the repository and image to access:

- For Dockerhub: NAME[:TAG|@DIGEST]
- For all other repositories: <repository DNS>/NAME[:TAG|@DIGEST]

For example, the Heroku repository DNS is `registry.heroku.com` our application name is
`alpine-hello` and for the purposes of Heroku, it needs a process type which is `web` to make
the default image accessable by `registy.heroku.com/alpine-hello/web`.

We can be more specific if we add a TAG or DIGEST. Either will be unique, but a TAG may be logical, like `production`, which can change, while a DIGEST is immutable, so you'll always get exactly what you intend.

For example:

- registy.heroku.com/alpine-hello/web:production
- registy.heroku.com/alpine-hello/web@sha256:406db728d882c63f269b45de8b24341494d7910c00ad3bc4422f91c99ed93509

## Before you start

If you want to run this example, you'll need to get a free [Heroku](https://signup.heroku.com)
account and you will need to install the *heroku cli*.

Installation of CLI is simple if you've got `node` installed. If you have trouble installing
the cli, you may need to update your `node` version, or experiment with using a containerized
version of the heroku CLI (see https://github.com/wingrunr21/alpine-heroku-cli).

If your version of Node is up to date, you only need the following: `npm install -g heroku-cli`

Once you've got the CLI installed, you'll need to log into Heroku and the Heroku docker registry.

Install the Heroku CLI and login to Heroku. The container registry is private, so you'll also need to log into it as well. Logging into a docker registry will add an up to date entry to
`$HOME/.docker/config.json` that will be good for some predefined time period.

```bash
$ heroku login
$ docker login --username=_ --password=$(heroku auth:token) registry.heroku.com
```

## Build, Test and Run Locally

Clone the Heroku sample *Hello World* application and create a Heroku project

```bash
$ git clone https://github.com/heroku/alpinehelloworld.git
$ cd alpinehelloworld
```

Looking at the `Dockerfile` you'll see that the application is a simple python *flask* application using `gunicorn` and that this application will need a `PORT` set in its environment.

Also notice that there are commands to setup a user account and make this the current user 
when running the web server. This is extremely important when developing your container
images and is often ignored. If, for some reason, your container becomes compromised by a 
hacker, you do not want that hacker to have `root` privileges (the default).

Build and run this on the local docker instance.

```bash
$ docker build --rm -t alpine-hello .
$ docker run --rm -ti -e PORT=8080 -p 8080:8080 alpine-hello
```

- `--rm` tells docker to remove any intermediate containers needed to build the container
- `-t` generate a tag for the built container
- `-e` lets you set an environment variable that can be used within th container. Heroku will
pass a PORT to the container when it launches the container

Now that the server is running, visit the url http://localhost:8080 to see the result.

Run the unit test!

```bash
docker run --rm -ti -e PORT=8080 -p 8080:8080 alpine-hello python -m unittest tests
```

If all goes well you should see the something like this:

```
.
----------------------------------------------------------------------
Ran 1 test in 0.023s

OK
```

Now you've got a running application that passes its unit tests!

## Deploy the exact same app to Heroku

No need to rebuild the container image, we've 'tested' it and it's ready for production. All
that's needed is to `tag` the image for the *Heroku Container Registry* and then push the image
to deploy the application. Then general pattern for images with Heroku is `registry.heroku.com/<app>/<process-type>[:TAG]`

```bash
$ docker tag alpine-hello registry.heroku.com/alpine-hello/web:v1.0.0
$ docker push registry.heroku.com/alpine-hello/web:v1.0.0
```

**Note the image digest** It will look something like `digest: sha256:406db728d882c63f269b45de8b24341494d7910c00ad3bc4422f91c99ed93509` and is
a unique identifier for this build artifact. This digest value should be the same for your original build of alpine-hello indicating that
they are identical. You can review this by using the command `docker images |grep alpine-hello |awk '{print $1, $3}'` which will print the
image tags with an abbreviated digest.

Now check that the application is running on Heroku by visiting the url http://alpine-hello.herokuapp.com

## Updating the Application

The general workflow for this example is as follows:

- update your unit tests
- update your application, potentially using the application environment approach
from Tutorial 1.
- build the container, make certain it runs and passes the unit tests
- tag with a new version (v1.0.1) and push the new container to Heroku

## Rolling back

There are times when a deployment goes wrong and you need to back out; containers make this simple.
All that is needed is to replace the container with the old container, which can be done by specifying
either the TAG or the DIGEST.

With Heroku, you'll need to use the TAG. This is easy, simply push the old container tagged with v1.0.0.

```bash
$ docker push registry.heroku.com/alpine-hello/web:v1.0.0
```

Once this is done, Heroku will startup the most recently pushed image which corresponds to the original
version 1.0.0 before the 1.0.1 update.

## Clean Up!

Once you're done, you can delete your Heroku app

```bash
$ heroku apps:delete -c alpine-hello alpine-hello
```

## For Further Consideration

If you are interested in learning more about container deployments, including multi-container
deployments, check out [Container Registry & Runtime (Docker Deploys)(https://devcenter.heroku.com/articles/container-registry-and-runtime)

If you are interested in investigating Digital Ocean to deploy your application, check out
the docker machine for Digital Ocean [here](https://docs.docker.com/machine/examples/ocean/) or the following tutorial: [Deploy with Docker and Digital Ocean](https://coderjourney.com/deploy-docker-digital-ocean/)

## Summary

This tutorial covered

1. Using the docker build command to build a container locally
1. Container Registry operations (login, push, pull)
1. Tagging images
1. Image Digests
1. How to roll back a release
1. Working with the Heroku CLI
