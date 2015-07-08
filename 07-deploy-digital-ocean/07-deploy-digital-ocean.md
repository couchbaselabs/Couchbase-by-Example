## Deploy Digital Ocean

So far, you have been running Sync Gateway and perhaps Couchbase Server to follow the tutorials.

Now it's time to learn how to easily deploy both applications and perhaps an App Server sitting alongside Sync Gateway to a PaaS.

First you will use the Docker images for Sync Gateway and [Couchbase Server](https://registry.hub.docker.com/u/couchbase/server/) available in the Docker registry.

Then, you will create a `Dockerfile` for NodeJS app that import data from the Google Places API to Sync Gateway. The NodeJS app you will deploy is in `04-ios-sync-progress-indicator`.

## Why Docker?

If you've ever done any application development and deployment then you know how difficult it can be to ensure that your development and production servers are the same or at least similar enough that it is not causing any major issue.

With docker, you can build a container that houses everything you need to configure your application dependencies and services.

This container can then be shared and run on any server or computer without having to do a whole bunch of setup and configuration.

Create a new Digital Ocean Droplet

> https://cloud.digitalocean.com/droplets/new

Pick 2GB for the RAM:

![](http://cl.ly/image/2D2w0H0a0a2W/Screen%20Shot%202015-07-08%20at%2009.27.21.png)

Choose Ubuntu for the distributions and on the Applications tab, select Docker. Create the Droplet and connect to it via SSH.

## Installing Sync Gateway and Couchbase Server

In the command line, install Couchbase Server with the following:

```
docker run -d -p 8091:8091 couchbase/server
```

Open the getting started wizard in the Chrome on port 8091

![](http://cl.ly/image/2o07072W1l3T/Screen%20Shot%202015-07-08%20at%2009.47.34.png)

After completing the wizard, navigate to the `Data Buckets` tab to find the buckets:

![](http://cl.ly/image/0s1f3m2O1v1r/Screen%20Shot%202015-07-08%20at%2009.48.47.png)

Now, you will install Sync Gateway with the config file 

Open and go through the 

Copy the necessary files from the `04-ios-sync-progress` tutorial to this project:

```
git clone git@github.com:couchbaselabs/Couchbase-by-Example.git
cd couchbase-by-example/04-ios-sync-progress-indicator
cp requestRx.js sync-gateway-config.json sync.js ./../07-deploy-digital-ocean/
```

Create a new copy of the config file to set it up with Couchbase Server:

```
cd 07-deploy-digital-ocean/
cp sync-gateway-config.json production-sync-gateway-config.json
```

Update `production-sync-gateway-config.json` with the server IP and bucket name:

```javascript
{
  "log": ["*"],
  "databases": {
    "db": {
      "server": "http://46.101.14.135:8091/",
      "bucket": "default",
      "users": { "GUEST": { "disabled": false, "admin_channels": ["*"] } }
    }
  }
}
```

Push the files to a github repository.

Specify the url to the production config file in the docker command to run the Sync Gateway container:

```bash
$ docker run -d -p 4984:4984 -p 4985:4985 couchbase/sync-gateway http://git.io/vq25r
```

**NOTE**: You ran `docker run` but this time specified the `-d` flag. It tells Docker to run the container and put it in the background, to daemonize it.

Run the `docker ps` command to check that both containers are running:

```
root@MyApp:~# docker ps
CONTAINER ID        IMAGE                    COMMAND                CREATED             STATUS              PORTS                                                                           NAMES
ca7d4358941a        couchbase/sync-gateway   "/usr/local/bin/sync   2 minutes ago       Up 2 minutes        0.0.0.0:4984-4985->4984-4985/tcp                                                focused_bell
c9411d002831        couchbase/server         "couchbase-start cou   52 minutes ago      Up 52 minutes       8092/tcp, 11207/tcp, 11210-11211/tcp, 0.0.0.0:8091->8091/tcp, 18091-18092/tcp   grave_feynman
```

Use the `docker logs` command specifying the container id to print the stdout to your console.

**TIP**: Use the `-f` flag to flow the logs.

In the next section, you write a simple Dockerfile to deploy the NodeJS application.

## Deploying an App Server

Open a new file named `Dockerfile` and paste the following:

```
FROM ubuntu: 14.04

// install nodejs

// install dependencies

// start the sync script on demand
```

Let's first pull down an existing image and run it in a new container. To do this, run:

```
docker run ubuntu /bin/echo "Hello from Docker"
```

It will use the latest ubuntu image if it isn't available locally on the computer it will pull down the image from the docker hub.

It will then create and start a new container with that image, run the echo command and stop the container.

Docker containers only run as long as they needed.

Let's go ahead and create a container that will serve a static html file through nginx. Create a new directory:

```
mkdir codetv_static
```

In that directory, create a new Dockerfile. We're using a Dockerfile because it lets us specify much more than just creating a container via the command line.

We first need to tell Docker which base image to use for this container using the `FROM` keyword, in this case, the latest ubuntu image.

Next, we'll use the `RUN` keywork to install nginx:

```
RUN apt-get update
RUN apt-get install -y nginx
RUN echo "daemon off;" >> /etc/nginx/nginx.conf
```

We'll need to tell Nginx not to stop the master process when it boots up otherwise docker will stop the container after nginx starts.

We'll override the defaut site, the default index.html that's created when nginx is installed with our custom one:

```
ADD index.html /usr/share/nginx/html/index.html
```

From the container, you're going to expose port 80 and then the entrypoint is the command that is run when the container is initialized.

```
EXPOSE 80

ENTRYPOINT /usr/sbin/nginx
```

Before you go ahead an build the image, you need to create the static html file that we're dropping in.

Build the docker image:

```
docker build -t codetv_static:nginx .
```

Now you can run the container:

```
docker run -p 80:80 codetv_static:nginx
```

It will map port 80 on the docker container to port 80 on the docker container to port 80 on the docker host which will start the container.

To grab the ip of the docker host:

```
boot2docker ip
```

Put that ip in the browser and you can see the 

Commit the new container image so you can use it as a base image later on:

```
docker commit -m "some message" id codetv_static:nginx
```

## Part 2

You're going to create a docker container for a NodeJS application and then deploy that container to a droplet on Digital Ocean.

Normally we wouldn't run the database in the same container but the purposes of this demo, this is perfectly fine.

In the directory of your NodeJS app, create a new `Dockerfile` file and specify a base image for the container.

In the next lines, add the commands to install the dependencies, nodejs.

Now, you will focus on the mode application specific things.

You will utilise the built-in caching in the `npm install` step so that if the package.json file hasn't changed, the gems aren't rebundled. This saves an incredible amount of time when developing. For a rails app, it would be:

```
WORKDIR /tmp
ADD Gemfile Gemfile
ADD Gemfile.lock Gemfile.lock
RUN /bin/bash -l -c "bundle install"
```

After that, you will add the NodeJS app directory and the Nginx configuration:

```
ADD ./ /var/www/journal
ADD config/journal.conf /etc/nginx/sites-enabled/journal
```

With docker containers, we need to have everything startup pretty much automatically. You can use a simple bash script to do that.

Expose port 80 again and the start_server command as the entry point.

Commit the changes in git and push to the remote.

Configure a droplet on digital ocean that has docker running.

Clone the remote repository and cd in that directory.

Now build the container like normal:

```
docker build -t codetv/journal .
```

This doesn't get into some of the things like connecting to a database outside of a container or deploying changes to the code base. We'll get into those next.

The goal in this tutorial is to:

 - easily deploy the sync gateway config file.
 - easily deploy couchbase server

First, install couchbase server on a DO droplet.
Second, install SG with your config file.
Third, write a Dockerfile for your app server and install it in your droplet. Drop in the shell to execute some commands.

