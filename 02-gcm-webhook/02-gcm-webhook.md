# Couchbase by Example: Sync Gateway Webhooks

In the previous post, you learned how to set up Google Cloud Messaging with the Service Worker and Push API to handle notifications and used PouchDB + Sync Gateway to sync registration tokens. In this tutorial, you will focus exclusively on Web Hook to dispatch the notifications to particular users.

We will continue on exploring Timely News, a news application to  notify you of new articles matching your interests.

## Scenarios

There are different scenarios for sending a push notification:

- Group Messaging: this concept was introduced in GCM to send notifications to up to 20 devices simultaneously. It’s very well suited for sending notifications to all devices that belong to a single user
- Up and Down: a user updated a document and other users should be notified about it through a Push Notification

## Data Model

Let’s start with the smallest document, a Profile document holding the registration tokens of the user’s devices:

	{
	    "type": "profile",
	    "name": "Oliver",
	    "subscription": "free", // other values "expired", "premium"
	    "topics": ["g20", "science", "nsa", "design"],
	    "registration_ids": ["AP91DIwQ", "AP91W9kX"]
	}

And the article document may have the following properties:

	{
	    "type": "article",
	    "title": "Design tools for developers",
	    "content": "...",
	    "topic": "design"
	}

## Group Messaging

Imagine a scenario where a user is currently signed up on a freemium account and inputs a invite code to access the premium plan for a limited time. It would be nice to send a notification to all the user’s devices to they can access the new data.

**Brief**: Send a one-off notification to freemium users that also have an invite code to unlock other devices.

Download Sync Gateway [here][1] copy. You can find the Sync Gateway binary in the `bin` folder and examples of configuration files in the `examples` folder. Copy the `exampleconfig.json` file to the root of your project:

	cp ~/Downloads/couchbase-sync-gateway/examples/exampleconfig.json /path/to/proj/sync-gateway-config.json

Add three users in the configuration file and add the Webhook string in the log array to monitor the webhook events:

	{
	  "log": ["CRUD", "HTTP+"],
	  "databases": {
	    "db": {
	      "server": "walrus:",
	      "users": {
	        "zack": {
	          "password": "letmein"
	        },
	        "ali": {
	          "password": "letmein"
	        },
	        "adam": {
	          "password": "letmein"
	        },
	        "GUEST": {"disabled": true}
	      }
	    }
	  }
	}

Add a web hook called `invite_code` with the following properties:

	  "event_handlers": {
	    "document_changed": [
	      {
	        "handler": "webhook",
	        "url": "http://localhost:8000/invitecode",
	        "filter": `function(doc) {
	              if (doc.type == "profile" && doc.invite_code) {
	                  return true;
	              }
	              return false;
	          }`
	      }
	    ]
	  }

Start Sync Gateway.  

Create a new file `main.go` to handle the webhook:

	'' http.HandleFunc("/invitecode", func(w http.ResponseWriter, r *http.Request) {
	''     log.Println("ping")
	''     // send activation notification to all devices
	'' })
  
Start the Go server:

	$ go run main.go

Using curl, make a POST request to `:4984/db/bulk_doc` to save 3 Profile documents simultaneously:

	curl -H 'Content-Type: application/json' \
	     -vX POST http://localhost:4985/db/_bulk_docs \
	     --data @profiles.json

**NOTE**: To save space on the command line, the `--data` argument specifies that the request body is in `profiles.json`.

Notice that only Ali’s Profile document is POSTed to the endpoint:

![][image-1]

In the next section, you will configure a second web hook to notify all users when a new article that matches their interest is published.

## Up and Down

Add another web hook that filters only documents of type `article`:

	 {
	    "handler": "webhook",
	    "url": "http://localhost:8000/new_article",
	    "filter": `function(doc) {
	        if (doc.type == "article") {
	            return true;
	        }
	        return false;
	    }`
	  }

Add another handler in your Go server:

	http.HandleFunc("/new_article", func(w http.ResponseWriter, r *http.Request) {
		log.Println("ping")
	})

Check that the webhook is working as expected added Article document:

	curl -H 'Content-Type: application/json' \
	     -vX POST http://localhost:4985/db/_bulk_docs \
	     --data @articles.json

In this case, you have to do a bit more work to figure out what set of users to notify. This is a good use case for using a view to index the Profile documents end emitting the topic as the key and registrations IDs as the value for every topic in the topics array.

To register a view, we can use the Sync Gateway PUT `/_design/ddocname` endpoint:

	curl -H 'Content-Type: application/json' \
	     -vX PUT http://localhost:4985/db/_design/extras \
	     --data @view.json

Notice that the article we posted above has design in it’s topic and the only user subscribed to this topic is adam. Consequently, if you query that view with the key "design", only one (key, value) pair should return with the topic as key and device tokens as value:

	curl -H 'Content-Type: application/json' \
	     -vX GET ':4985/db/_design/extras/_view/user_topics?key="design"'

Now, you can edit handler in `main.go` to subsequently query the `user_interests` view with the key being the topic name of the article:

	handler

Run the curl request again and you will see the list of device tokens to send a push notification to.

## Conclusion

In this lesson, you learned how to use Web Hooks in the scenario of GCM Push Notification and used Couchbase Server Views to access additional information at Webhook Time™.




[1]:	http://packages.couchbase.com/builds/mobile/sync_gateway/1.1.0/1.1.0-16/couchbase-sync-gateway-community_1.1.0-16_x86_64.tar.gz

[image-1]:	http://i.gyazo.com/7ec3dd332f2d029af364590a4c2e3e63.gif