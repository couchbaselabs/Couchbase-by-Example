# Couchbase by Example: Channel, Users, Roles

In `04-ios-sync-progress-indicator`, you learnt how to use RxJS and the request module to import documents into a Sync Gateway database from the Google Places API. To keep it simple, you enabled the GUEST user with access to all channels. In this tutorial, you will configure the Sync Function to allow authenticated users to post reviews.

The sync function validates document contents, and authorizes write access to documents by channel, user, and role.

In total, there will be 3 types of roles in the application:

 - **level-1**: users with the **level-1** role can post reviews but they must be accepted by users with the **level-3** role (i.e. moderators) to be public.
 - **level-2**: users can post reviews without validation needed from moderators. This means they can post a comment without requiring an approval.
 - **level-3**: users can approve reviews or reject them.

## Download Sync Gateway

First you need to have a Sync Gateway instance running with documents (including attachments) to replicate. You will use the NodeJS app from `04-ios-sync-progress-indicator` to do that.

Download Sync Gateway and unzip the file:

> http://www.couchbase.com/nosql-databases/downloads#Couchbase\_Mobile

You will find the Sync Gateway binary in the `bin` folder and examples of configuration files in the `examples` folder. Copy the `exampleconfig.json` file to the root of your project:

```bash
cp ~/Downloads/couchbase-sync-gateway/examples/users-roles.json /path/to/proj/sync-gateway-config.json
```

In the next section, you will change the configuration file to have additional users and roles.

## Channels, Users and Roles

Roles and users can both be granted access to channels. Users can be granted roles, and inherit any channel access for those roles.

![image describing this sentence]()

Channel access determines a user’s read security. Write security can also be based on channels (using requireAccess), but can also be based on user/role (requireUser and requireRole), or document content (using throw).

1. For each channel, define channels for read/write/delete/admin. No documents would be assigned to those channels - these channels would be strictly to manage write security.
2. Use an admin document to manage access to those channels, using the access() call in the sync function.
3. Use requireAccess() to validate document writes, based on those channels.

Giving a role access to a channel. The Moderator role should have access to all channels. Instead of giving each user with the Moderator role access to all channels, you can use a role to do so.

The user that is the owner can add/removes users from a channel.

In `sync-gateway-config.json`, update the db object to read as follow:

```javascript
{
  "log": ["*"],
  "databases": {
    "db": {
      "server": "walrus:",
      "users": {
        "jens": {
          "admin_roles": ["level-1"],
          "password": "letmein"
        },
        "andy": {
          "admin_roles": ["level-2"],
          "password": "letmein"
        },
        "william": {
          "admin_roles": [],
          "password": "letmein"
        },
        "traun": {
          "admin_roles": ["level-3"],
          "password": "letmein"
        }
      },
      "roles": {
        "level-1": {},
        "level-2": {},
        "level-3": {}
      },
    }
  }
}
```

A couple of things are happening above:

 1. You create the user `jens` with the `level-1` role.
 2. You create the user `andy` with the `level-2` role.
 3. You create the user `william` without any role.
 4. You create the user `traun` with the `level-3` role.
 5. You define the 3 roles. Just like users, roles must be explicity created on the Admin REST API or in the config file. 

**Note on creating roles**

The easiest way to create roles is in the configuration file as you did above.

Another way to create roles is through the admin REST API. Provided that you expose an endpoint to create those roles from the application, you can create roles dynamically by sending a request to your app server (blue arrows) which will create the role and send back a 201 Created if it was successful (green arrows).

![](http://cl.ly/image/3D0606230F1C/Dynamic%20Roles.png)

In the next section, you will add the Sync Function to handle write and read operations for the three different types of documents (`restaurant`, `review`, `profile`).

## Sync Function

Before you continue testing the sync function, let's discuss general principles that apply to all Sync Functions.

Read and write access to documents are independent. In fact write access is entirely governed by your sync function: unless the sync function rejects the revision, a client can modify any document. All the require* functions act as validators but also write access APIs.

It's very common to see sync function creating lots and lots of channels. This is absolutely fine. However, it can get cumbersome to assign each user in turn to a channel. Instead you can use a role!

Let this sink in one more time, users can be granted roles and inherit any channel access for those roles.

This means you can grant a user access to multiple channels by simply assigning a role. This is very powerful because it means you can grant a role access to a channel and when the profile comes along, simply assign the user to that role.

With roles, you don't need to assign every single user to a channel. You simply grant the role access to the channel and assign the users to the role.

Replace the sync function in `sync-gateway-config.json`:

```
function(doc, oldDoc) {
if (doc.type == "restaurant") {
requireRole("moderator")
channel(doc.restaurant_id);
} else if (doc.type == "comment") {
switch(doc.role) {
case "level-1":
// write access
requireRole(doc.role);
channel(doc.owner + "-in-review");

// read access
access(doc.owner, doc.owner + "-in-review");
access("role:level-3", doc.owner + "-in-review");

break;
case "level-2":
// write access
requireRole(doc.role);
channel(doc.restaurant_id);
break;
case "level-3":
requireRole("beginner");
channel(doc.restaurant_id);
break;
}
} else if (doc.type == "profile") {
requireRole("level-3");
role(doc.name, "role:" + doc.role);
}
}
```

Here's what's happening:

 1. Users with the **level-1** role have write access because you call the channel function. Then grant that user and the **level-3** access to this channel. This is where the power of roles really shines. By granting a role access, you are granting all the users with that role access to the channel. This will test the write security with requireRole and requireAccess.
 2. Documents of type comment created by a **level-2**: the document should go in the same channel as the restaurant it belongs to (i.e. **Trustee** users have write and read access to the restaurant channels). This will test the write security with requireRole.
 3. Documents of type comment created by a **level-3**: the document should go in the channel assigned to the restaurant. **Moderator** users also have read access to the `in-review` channel. They can modify documents in this channel.

Restart Sync Gateway.

In this example, you are utilising the 3 main features of roles:

 - Granting a role access to a channel and indirectly all the users with that role.
 - Granting write permission using a role.
 - Assigning a role to a user.

Using roles to their full potential can greatly simplify your sync function and data model.

**Scenario 1**

Documents of type `review` created by a **level-1** user: the document should go in the `{user_name}-in-review` channel and the **level-3** should have access to this channel too.

Login as `jens` and replace the token below.

```bash
curl -vX POST -H 'Content-Type: application/json' \
     --cookie 'SyncGatewaySession=d007ceb561f0111512c128040c32c02ea9d90234' \
     :4984/db/ \
     -d '{"type": "comment", "role": "level-1", "owner": "jens"}'
```


 - Check that user `jens` has access to the channel `jens-in-review` and the comment document is in there.
 - Check that user `traun` has access to channel `jens-in-review`.

**Scenario 2**

Granting write access using a role.

Login as `andy` and replace the token below.

```bash
curl -vX POST -H 'Content-Type: application/json' \
              --cookie 'SyncGatewaySession=6e7ce145ae53c83de436b47ae37d8d94beebebea' \
              :4984/db/ \
              -d '{"type": "comment", "role": "level-2", "owner": "andy", "restaurant_id": "123"}'
```

- Check that the comment was added to the restaurant channel.

**Scenario 3**

Assigning a role to a user. Assign `william` to role `level-3`. Logged in as Traun.

Login as `traun` and replace the token below.

```bash
curl -vX POST -H 'Content-Type: application/json' \
              --cookie 'SyncGatewaySession=3a5c5a67ff67643f8ade175363c65354584429e9' \
              :4984/db/ \
              -d '{"type": "profile", "name": "william", "role": "level-3"}'
```

 - Check that William has role `level-3`.
 - Check that `william` has access to the `jens-in-review` channel.

**NOTE**: Notice that the user `william` has role `level-2` and `level-3` now.

## Conclusion

In this tutorial, you learnt how to use channels and requireRole to dynamically validate and perform write operations. You could assign multiple channels at once to multiple users.