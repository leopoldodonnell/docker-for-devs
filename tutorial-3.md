# Tutorial 3 - Dockerfile and Docker Compose

Typical modern architectures require more than one process. This tutorial introduces `docker-compose` to orchestrate
running more than one server with more than one network. Furthermore, they are setup to be built AND to be able to
make on-going development changes. We'll also look more carefully at some of the more popular commands that are part
of the `Dockerfile` format.

Everything you do with `docker-compose` can be performed using `docker` commands directly. What `docker-compose` does
is give you a simple decalaritive way, through a *yaml* file, to configure one or more containers with their networking,
volumes, processes and environment. You then use `docker-compose` to interact with your configuration to run, stop,
restart, grab logs and more.

`docker-compose` will use the `docker-compose.yml` file by default. You can override this with a flag, but its usually best to stick with the default to keep things simple and avoid confusion.

## The Example Voting Application

The *Example Voting Application* consists of five servers, three are coded as part of the application and two, a
*redis* and *postgresql* database are used as is.

Each of these servers corresponds to a container:

- `vote`: the vote server is a *python flask* web application used to enter votes
- `result`: the result server is an *angular nodejs* web application used to view the results
- `worker`: the worker is a *.Net* application that processes votes and stores them in the database
- `redis`: the redis server is used to provide a voting queue that the worker consumes and holds the results
- `db`: the database is a *postgresql* server used to store the votes

Go and get the the project...

```bash
$ git clone git@github.com:dockersamples/example-voting-app.git
$ cd example-voting-app
```

## Building the Application

Container images that need to be built are specified by the `build` keyword in the `docker-compose.yml` file.

For example: `build: ./vote` will tell docker-compose to look in the `./vote` directory for a `Dockerfile` and *context*
(all of the files) to build the image. This image will be called `example-voting-app_vote` by default. If you would like more control,
`build` will let you specify the context directory, the dockerfile and the image.

**For example:** The following will use files from `the-voting-app` directory, and the Dockerfile `./the-voting-app/Dockerfile.vote`
to create the image with the tag `my-private-registry.my-company.com/vote:dev`.

```yaml
vote:
  build:
    context: ./the-voting-app
    dockerfile: ./the-voting-app/Dockerfile.vote
  image: my-private-registry.my-company.com/vote:dev
``` 

For the time being, you'll use the defaults and all of the containers can be built with a single command:

```bash
$ docker-compose build
```

Alternatively, you can specify a specific container to build by providing it as an argument. Eg. `docker-compose build vote`

### Disecting the result Dockerfile

We'll use the Dockerfile for the **result** application to review the `Dockerfile` commands.

The file is short enough, so we'll include it here:

```dockerfile
FROM node:8.9-alpine

RUN mkdir -p /app
WORKDIR /app

RUN npm install -g nodemon
RUN npm config set registry https://registry.npmjs.org
COPY package.json /app/package.json
RUN npm install \
 && npm ls \
 && npm cache clean --force \
 && mv /app/node_modules /node_modules
COPY . /app

ENV PORT 80
EXPOSE 80

CMD ["node", "server.js"]
```

This file contains the following Dockerfile commands:

- **FROM** - Dockefiles usually start by adding to an existing container image. This one uses the *Alpine* version of *nodejs*
- **RUN** - Run commands will essentially run a command in the default shell for the operating system. You can string commands 
together will `&&` to optimize the container by reducing layers. *RUN* commands are used to install and configure software
- **WORKDIR** - Is the working directory to use going forward. This can change multiple times within a Dockerfile, with
the container runtime working directory set to the last one encountered.
- **COPY** - This application needs files moved into the container. Copy will copy a file or a directory tree to a location inside the
container image
- **ENV** - The ENV command will set an environment variable with a default that can be used during the building of a container, or at runtime. You can override these variables when you launch a container
- **EXPOSE** - While not explicitly necessary, EXPOSE will expose a port for network access to the container
- **CMD** - CMD can provide a default command line to run when the container starts. The format above is the preferred method provided
as an array of strings to represent the command `node server.js` from the `/app` directory.

Much more can be said for all of these commands. Refer to the [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
for more information.

## Configuring the Application

You've already looked at the `build` specifications in the `docker-compose.yml` file. The following will go over what the
rest of the configuration does to get the `example-voting-app` applicaiton to work.

### version

The `version` directive tells `docker-compose` which version of the `docker-compose` specification to use when parsing the file. This file is using version `3`.

### services

Each container in this specification is considered a **service**. Each service is indented under the keyword `services:` with each service beginning with its name and followed by the service's specification.

This application has five services, vote, result, worker, redis and db. Images and containers that are built/started will, by default, begin with the name of the application. Once started, for example, the `vote` service will be running in the container named `example-voting-app_vote`.

### image

The `image` keyword identifies the name/tag of the container image to run. If the image is being built, it will use the specified name, if the image exists on the docker host (eg your workstation), it will use that, otherwise it will attempt to pull the image.

In the `example-voting-app` both the `redis` and the `db` services specify their images which will be pulled if necessary.

### volumes

Volumes can be created, or simply shared using the `docker-compose.yml` file. This example uses both.

The `volumes` keyword, at the same level as `services` is used to
create any volumes that may be needed by your application. This example creates a volume `db-data` for the database so its data will persist beyond restarts.

The `volumes` keyword found under a service are used to specify and array (entries are specified with the `-`) of one or more filesystems that will be mounted within the service container and where that filesystem will be mounted. For example, the `db-data` volume is mounted as `/var/lib/postresql/data` in the `db` service.

More intesting for development is the use of volume mounting by the `vote`, `result` and `worker` services. In each of these cases, the specification mounts the local directory for that service at runtime.

This is an important container development pattern that enables the same code to build a final container, or for on-going development. In each of the above cases, you can make changes to service code then refresh the browser contents to see your changes. 

Once you are satisfied, you can rebuild, restart, test and push your container.

### networks

Specifying networks for services enable containers to discover the services running in other containers that are on the same network. For example, the `vote` service has the `back-tier` network and `redis` also has this network, so `vote` processes can access the `redis` service using `redis` instead of specifying the `IP`.

Like `volumes`, the `networks` keyword for specifying networks exists at the same level of `services`. The example creates two networks, `front-tier` and `back-tier`. Each service then specifies which networks they will be able to communicate over.

As you can see the web servers, `vote` and `result` are available on both networks while the `db` and `redis` are only availabe over the `back-tier`. This will simulate a deployment where you have services that are available on an external network and services that are exclusively internal.

### ports

The `ports` keyword specifies the ports that will be exposed by the service. This is a yaml array which can be written as a
JSON array ( ["one", "twho", "..."] ), or as a single item per indented line that is preceded by a `-`.

Services on the same network can use a container's internal port while external access is only available via the extenal ports. Specifying a single port only identifies the internal port while specifying an internal and external port is specified by separating the ports with a `:`

For example, `redis` is available to internal traffic on port `6379` while the result service is available externally on `5001` for http and `5858` for client messages.

### command

All of the container images have been setup to run a specific command when they are started. Development can require that the default is overriden to help with making live changes and debugging.

You will notice that the `command` keyword has been overriden for the `result` service. By default, the container image will simply run `node`, but this will mean that you'll have to restart the server each time you make a change. The solution to this is to replace `node` with `nodemon` when you start the container.

Command can be specified as a string, as an array of strings, or
as a yaml array.

eg:

```yaml
command: nodemon server.js
# OR
command: ["nodemon", "server.js"]
# OR
command:
  - nodemon
  - server.js
```

### depends_on

Lastly, there is the `depends_on` keyword. Use this keyword when you need to explicitly control the order that services start.

In `example-voting-app`, `worker` shouldn't start before `redis`

## Managing the Application

While that was a lot of background, starting a number of services up with `docker-compose` is easy...

```bash
$ docker-compose up
```

This will start up the services in the foreground so you can see the logs.

Stopping the application is just as easy. From another terminal ...

```bash
$ docker-compose down
```
Now let's start the services as daemons...

```bash
$ docker-compose up -d
Creating network "example-voting-app_front-tier" with the default driver
Creating network "example-voting-app_back-tier" with the default driver
Creating example-voting-app_result_1 ... done
Creating redis                       ... done
Creating example-voting-app_vote_1   ... done
Creating db                          ... done
Creating example-voting-app_worker_1 ... done
$
```
Check that they're running...

```bash
$ docker-compose ps
```

```
           Name                          Command               State                      Ports
-------------------------------------------------------------------------------------------------------------------
db                            docker-entrypoint.sh postgres    Up      5432/tcp
example-voting-app_result_1   nodemon server.js                Up      0.0.0.0:5858->5858/tcp, 0.0.0.0:5001->80/tcp
example-voting-app_vote_1     python app.py                    Up      0.0.0.0:5000->80/tcp
example-voting-app_worker_1   /bin/sh -c dotnet src/Work ...   Up
redis                         docker-entrypoint.sh redis ...   Up      0.0.0.0:32768->6379/tcp
```

Start reviewing the logs...

```bash
$ docker-compose logs -f
Attaching to example-voting-app_worker_1, example-voting-app_result_1, redis, db, example-voting-app_vote_1
result_1  | [nodemon] 1.17.3
result_1  | [nodemon] to restart at any time, enter `rs`
result_1  | [nodemon] watching: *.*
result_1  | [nodemon] starting `node server.js`
result_1  | Thu, 19 Apr 2018 14:22:55 GMT body-parser deprecated bodyParser: use individual json/urlencoded middlewares at server.js:68:9
result_1  | Thu, 19 Apr 2018 14:22:55 GMT body-parser deprecated undefined extended: provide extended option at ../node_modules/body-parser/index.js:105:29
result_1  | App running on port 80
result_1  | Connected to db
worker_1  | Connected to db
worker_1  | Connecting to redis
worker_1  | Found redis at 172.23.0.4
redis     | 1:C 19 Apr 14:22:53.670 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis     | 1:C 19 Apr 14:22:53.670 # Redis version=4.0.9, bits=64, commit=00000000, modified=0, pid=1, just started
...
```

`^c` will stop streaming the logs.

If you want to zero in on a single service...

```bash
$ docker-compose logs vote
```

Exiting and cleaning up...

```bash
$ docker-compose down
```

## Workflow

Your workflow will depend on how your containers are setup. If your servers need to be recompiled the workflow will look something like...

- Start your services
- Make changes to your sourcecode
- Rebuild the affected container images
- Use `docker-compose restart <services affected> and test


If you are using an interpreted framework like `nodejs` you can mount the source code as a volume, as is done with the `result` service. The workflow is now...

- Start your services
- Make changes to your sourcecode and test as you go

## Summary

This tutorial covered

1. Using docker-compose to build a group of containers
1. The docker-compose specification
1. Connecting multiple container together
1. Managing the running lifecycle of a group of containers
1. Using docker-compose in development context to speed up development.
