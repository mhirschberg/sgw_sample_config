# Configure Couchbase SyncGateway with scopes and collections
## Introduction and prerequisites
This document describes the configuration of Couchbase SyncGateway with scopes and collections.
It has been verified on both MacOS and Linux. Windows users are on their own.

Here's what you need:
* A docker engine on your local machine, to run both Couchbase Server and Couchbase SyncGateway
* [cblite tool](https://github.com/couchbaselabs/couchbase-mobile-tools/releases) to work with a local database. Under `Assets` please download the binaries for your OS and set the executable flag on `bin/cblite` after extracting.
* Clone this repository and `cd sgw_sample_config/files` - that will be your working directory 
* Create a new docker network, for instance `docker network create -d bridge sg_workshop`
* We're going to use `cb-server` as a name for our Couchbase Server, please edit your `hosts` file and add `cb-server` to the line resolving `127.0.0.1`

## Installing and configuring Couchbase Server
It's as simple as running this:
```
docker run -d --name cb-server  --network sg_workshop -p 8091-8096:8091-8096 -p 11210-11211:11210-11211 couchbase
```

Once it's started and ready to use, open `http://localhost:8091` in the browser of your choice.
The configuration is fairly simple: 
* Click on `Setup new cluster`
* Give the cluster a name
* Select an admin user name
* Choose a password for that user
* Click `Next: Accept terms`
* Select `I accept the terms & conditions` and click `Configure Disk, Memory, Services`
* De-select the services `Search`, `Analytics`, `Eventing`, `Backup` leaving only `Data`, `Query`, `Index` enabled
* Click `Save and finish`
* Click on `Buckets` on the left side and then `load sample buckets` on the right side
* Select `travel-sample` and click `Load sample data`
* On the left side, click `Security` and then `Add user` on the top right
* This user will connect from the SyncGateway, so choose a meaningful name - I use `SG_account` - and a password, select the role `Sync Gateway` for the bucket `travel-sample` under `Mobile` and finish by clicking the `Add User` button
* Now switch to your working directory and edit `sgw_bootstrap.json` file, you need to replace the name and the password with the ones you just created

## Install and configure SyncGateway
### Starting a SGW container
As simple as 
```
docker run -p 4984-4985:4984-4985 --network sg_workshop --name sync-gateway -d -v `pwd`/sgw_bootstrap.json:/etc/sync_gateway/sync_gateway.json couchbase/sync-gateway:3.1.2-enterprise -adminInterface :4985 /etc/sync_gateway/sync_gateway.json
```

Once it's up running, you can verify that everything works as expected by looking at the log file.
Switch to the container:
```
docker exec -it sync-gateway bash -i
```
And tail the log file. You should see something like that:
```
tail -4 /tmp/sg_log_output.log
2024-02-29T10:28:17.944Z [INF] Finished initializing server connections
2024-02-29T10:28:17.944Z [INF] Starting metrics server on 127.0.0.1:4986
2024-02-29T10:28:17.944Z [INF] Starting admin server on :4985
2024-02-29T10:28:17.944Z [INF] Starting server on :4984 ...
```
Exit the container by typing `exit`

### Configure a SyncGateway database
We're going to tell the SyncGateway which Couchbase bucket, scope and collections we'd like to use, which documents we'd like to see. 
Have a look at `sgw_bucket_config.json`. We're using the bucket `travel-sample`, the scope `inventory` with the collections `airline` (sendindg the documents of `airline` type to the `airlines` channel) and `hotel` (sending the documents of `hotel` type to the `hotels` channel).
```
{
  "name": "travel-sample",
  "bucket": "travel-sample",
  "scopes" : {
    "inventory": {
      "collections": {
        "airline" : {
            "sync": `function(doc, oldDoc, meta) { channel("airlines") }`,
            "import_filter": `function(doc) { return doc.type == "airline" }`
        },
        "hotel" : {
          "sync": `function(doc, oldDoc, meta) { channel("hotels") }`,
          "import_filter": `function(doc) { return doc.type == "hotel" }`
        }
      }
    }
  },
  "num_index_replicas": 0
}
```
Let's make the communication safe and store tne `base64` encoded SGW user/password combination in a variable. Please change the user name and password accordingly:
```
export DIGEST=$(echo -n <SGW_USER_NAME>:<SGW_USER_PASSWORD> | base64)
```

Now we're ready to configure the SGW database:
```
curl --location --request PUT 'http://localhost:4985/travel-sample/' --header "Authorization: Basic $DIGEST" --header 'Content-Type: application/json' -d@sgw_bucket_config.json
```

### Configure SyncGateway role
We've got a SGW datavase, so let's create a user role to access it - we'll give that role full access to the channels `airline` and `hotels`, created in the previous step.
```
curl -X POST "http://localhost:4985/travel-sample/_role"  --header "Authorization: Basic $DIGEST" -H "accept: */*" -H "Content-Type: application/json" --data-raw '{"name":"airline_agent", "admin_channels":["airlines", "hotels"]}'`
```

### Configure SyncGateway user
Let's say, in our example we've got a very basic travel agency. 
In my example the user working with our SGW database is called `agent`, but please feel free to chose another name.
Please open the `user_config.json` file and change the user name and the password. Note, the user is assigned to the role `airline_agent` created in the previous step.

Ready to create the user:
```
curl -X POST "http://localhost:4985/travel-sample/_user/"   --header "Authorization: Basic $DIGEST" -H "accept: */*" -H "Content-Type: application/json" -d@user_config.json
```

## Verify the configuration
I hope there were no errors.
Trusting the configuration is good, but verifying is better.

### Create local cblite database
Swith to the directory with the `cblite` binary.
Create a local database by executing
```
./cblite --create traveldb.cblite2
```
This command will create an empty database `traveldb` and open it r/w.

To sync our configured `inventory` scope with the `airline` and `hotel` collections, we need to create them locally, too.
```
(cblite) mkcoll inventory/hotel
Created collection 'inventory/hotel'.
(cblite) mkcoll inventory/airline
Created collection 'inventory/airline'.
```
### Pull the documents from the remote database
SyncGateway should send the documents exactly as configured. Please change the user name and password to the values from your `user_config.json` file
```
(cblite) pull --user <LOCAL_USER_NAME>:<LOCAL_USER_PASSWORD> --collection inventory.airline --collection inventory.hotel ws://localhost:4984/travel-sample
...
Completed 1104 docs in 0.527002 secs; 2094 docs/sec
```

Something has been pulled! But what exactly? Let's see...
```
(cblite) cd inventory/airline
(cblite) ls -l --limit 10

airline_10              1-af3785ca ---     184    0.08K
airline_10123           1-1563766c ---       7     0.1K
airline_10226           1-c1f55e4c ---      60     0.1K
airline_10642           1-cafc7402 ---       9     0.1K
airline_10748           1-52382b13 ---     143     0.1K
airline_10765           1-7334925f ---      52     0.1K
airline_109             1-4244111a ---     167    0.09K
airline_112             1-5a40eddf ---      54    0.08K
airline_1191            1-6b57f630 ---      48    0.07K
airline_1203            1-5fa2c672 ---       6    0.07K
(Stopping after 10 docs)

(cblite) get airline_10
{
  "_id": "airline_10",
  "callsign": "MILE-AIR",
  "country": "United States",
  "iata": "Q5",
  "icao": "MLA",
  "id": 10,
  "name": "40-Mile Air",
  "type": "airline"
}
```
We just verified that the right documents have been pulled correctly.

### Change a document locally
We own a small travel agency, right? And we just realised that the airline from the above example has been rebranded and has a new name, `42-Kilometer Air`.
Let's open the document `airline_10` and reflect that. I use `vim` editor with the `--with` flag, but please feel free to use whatever works best for you.
```
(cblite) edit --with /usr/bin/vim airline_10
```

It's a good idea to make sure the change has been persisted:
```
(cblite) get airline_10
{
  "_id": "airline_10",
  "callsign": "MILE-AIR",
  "country": "United States",
  "iata": "Q5",
  "icao": "MLA",
  "id": 10,
  "name": "42-Kilometer Air",
  "type": "airline"
}
```

### Push local changes to the remote database
The document is still stored locally. 
Without further ado, let's push all the changes in the `airline` collection to the remote database (please change the local user name and password as you've configured):
```
(cblite) push --user <LOCAL_USER_NAME>:<LOCAL_USER_PASSWORD> --collection inventory.airline ws://localhost:4984/travel-sample
...
Completed 1 docs in 0.319762 secs; 3 docs/sec
```

Now, go back to the Couchbase UI, click `Buckets` and then `Documents` for our `travel-sample` bucket.
Under `Keyspace` select the scope `inventory` and the `airline` collection.
Click on our `airline_10` document.
Observe the change being in place:
```json
{
  "id": 10,
  "type": "airline",
  "name": "42-Kilometer Air",
  "iata": "Q5",
  "icao": "MLA",
  "callsign": "MILE-AIR",
  "country": "United States"
}
```

Magic, innit?
