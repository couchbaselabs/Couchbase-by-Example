# Couchbase by Example: Sign up and Login

With the Sync Gateway API, you can create users and authenticate on the client side as a specific user to replicate the data this user has access to. There are two ways to create users:

- In the configuration file under the `users` field.
- On the Admin REST API.

To provide a login and sign up screen, you must setup an app server that handles the user creation accordingly as the admin port (4985) is not publicly accessible. In this tutorial, you'll learn how to:

- Use the Admin REST API to create a user.
- Setup an App Server with NodeJS to manage the users.
- Design a Login and Sign up screen in a sample Android app to test your App Server.

## Getting started

Download Sync Gateway and unzip the file:

> http://www.couchbase.com/nosql-databases/downloads#Couchbase\_Mobile

For this tutorial, you won't need a configuration file. For basic config properties, you can use the command line options. The binary to run is located in `~/Downloads/couchbase-sync-gateway/bin/`. Run the program with the `--help` to see the list of available options:

```bash
~/Downloads/couchbase-sync-gateway/bin/sync_gateway --help
Usage of /Users/jamesnocentini/Downloads/couchbase-sync-gateway/bin/sync_gateway:
  -adminInterface="127.0.0.1:4985": Address to bind admin interface to
  -bucket="sync_gateway": Name of bucket
  -configServer="": URL of server that can return database configs
  -dbname="": Name of Couchbase Server database (defaults to name of bucket)
  -deploymentID="": Customer/project identifier for stats reporting
  -interface=":4984": Address to bind to
  -log="": Log keywords, comma separated
  -logFilePath="": Path to log file
  -personaOrigin="": Base URL that clients use to connect to the server
  -pool="default": Name of pool
  -pretty=false: Pretty-print JSON responses
  -profileInterface="": Address to bind profile interface to
  -url="walrus:": Address of Couchbase server
  -verbose=false: Log more info about requests
```

For this tutorial, you will specify the `dbname`, `interface`, `pretty` and `url`:

```bash
~/Downloads/couchbase-sync-gateway/bin/sync_gateway -dbname="smarthome" -interface="0.0.0.0:4984" -pretty="true" -url="walrus:"
```

To create a user, you can run the following in your terminal:

```bash
$ curl -vX POST -H 'Content-Type: application/json' \
       -d '{"name": "adam", "password": "letmein"}' \
       :4985/smarthome/_user/
```

**NOTE**: The name field in the JSON object should not contain any spaces.

This should return a `201 Created` status code. Now, login as this user on the standard port:

```bash
$ curl -vX POST -H 'Content-Type: application/json' \
       -d '{"name": "adam", "password": "letmein"}' \
       :4984/smarthome/_session
```

The response will contains a `Set-Cookie` header and the user's details in the the body.

All of the Couchbase Mobile SDKs have a method to specify a user's name and password for authentication so you will most likely not have to worry about making that second request to login.

## App Server

In this section, you'll use the necessary Admin REST API endpoints publicly to allow users to sign up through the app.

You'll use the `http-proxy` NodeJS module to proxy request to the Sync Gateway.

![](http://cl.ly/image/0O203c1S3B0L/Custom%20Auth%20Signup%20(4).png)

Open a new file `server.js` with the following:

```javascript
var http = require('http')
  , httpProxy = require('http-proxy')
  , request = require('request').defaults({json: true});

// 1
var proxy = httpProxy.createProxyServer();
// 2
var server = http.createServer(function (req, res) {

  // 3
  if (/signup.*/.test(req.url)) {
    console.log('its signup time');

    req.on('data', function (chunk) {
      var json = JSON.parse(chunk);
      var options = {
        url: 'http://0.0.0.0:4985/smarthome/_user/',
        method: 'POST',
        body: json
      };
      
      request(options, function(error, response) {
        res.writeHead(response.statusCode);
        res.end();
      });

    });

    req.on('end', function () {

    });

  // 4
  } else {
    proxy.web(req, res, {target: 'http://0.0.0.0:4984'});
  }

});

server.listen(8000);
```

Here's what is happening step by step:

1. Instantiate a new instance of the proxy server.
2. Instantiate a new instance of the http server.
3. Check if the url path is `/signup` and proxy the request on the admin port 4985.
4. Proxy all other requests on the user port 4984.

From now on, you can use one url to create users and perform all other operations available on the user port.

> http://localhost:8000

Create another user to test the everything is working as expected:

```bash
$ curl -vX POST -H 'Content-Type: application/json' \
       -d '{"name": "andy", "password": "letmein"}' \
       :8000/signup/
```

And to login as this user:

```bash
$ curl -vX POST -H 'Content-Type: application/json' \
       -d '{"name": "andy", "password": "letmein"}' \
       :8000/smarthome/_session
```

In the next section, you will create an simple Android app with a login and signup screen to test those endpoint.

## Android app

Open Android Studio and select **Start a new Android Studio project** from the **Quick Start** menu:



Name the app **SmartHome**, set an appropriate company domain and project location, and then click **Next**:

![](http://cl.ly/image/2h3R3r1K041F/Screen%20Shot%202015-07-29%20at%2015.23.20.png)

On the Target Android Devices dialog, make sure you check **Phone and Tablet**, set the Minimum SDK to **API 22: Android 5.1 (Lollipop)** for both, and click **Next**:

![](http://cl.ly/image/241n02472f13/Screen%20Shot%202015-07-29%20at%2015.24.15.png)

On the subsequent **Add an activity to Mobile** dialog, select Add **Blank Activity** and name the activity **Welcome Activity**:

![](http://cl.ly/image/1d2F2D372K1m/Screen%20Shot%202015-07-29%20at%2015.27.18.png)

Next, you will add the Android Design Library as a dependency as it provides slick EditText inputs that you will use for the Login and Signup screens.

In `build.gradle`, add the reference to the design library:

```
compile 
```

Add the two buttons in a LinearLayout

Add the two method handlers and invoke the method creation intention

Create the signup activity

Change the Java inheritance from ActionBarActivity to Activity for the java classes

Change the theme in `styles.xml` to `Theme.AppCompat.Light.DarkActionBar`.

Add the xml template for the signup screen.



## Conclusion

In this tutorial, you learnt how to use the Admin REST API to create users and authenticate with Android SDK against Sync Gateway.