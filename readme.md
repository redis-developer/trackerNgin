# Why TrackerNgin 
- To help Public and emergency Services 
 	- With location of their vehicles
	- Communicate with Trackee
- To help users Find these service vehicles That is nearest to them.

## Key Terms 
#### 1. The Tracker/Services : 
	- Ambulance Command Center
	- Public Transport Command Center
	- Police Vehicle Command Center
	- Vehicle Towing Command Center
	- Garbage Management Command Center
	- And more….


#### 2. The Trackee:
##### Ambulance
	- Bus/Train/Metro
	- Police Vehicle
	- Towing Vehicle
	- Garbage management Vehicle
	- And More…….

#### 3. The User:
	- Me and You

## Key features:
#### 1. One to many and many to one tracking
#### 2. Alias , can be set at any time and can represent
		1. Vehicle
		2. Person
		3. Number beds in hospital (Covid use case)
		4. and many more.
#### 3. Filter users to be monitored
#### 4. Send message to online users ( 1 to 1 and 1 to all)

## How redis helps us
- Redis HASH key allows us to store info
 - Redis SEARCH Helps Tracker to get the location of all Trackees and authenticate using multiple parameters ( email and phone )
	- Redis GEO helps us to give the distance ( GEO RADIUS )
 - Redis RPUSH and LPOP works to perform simple emergency PERSISTENT messaging

## Feeding data
- Api calls to exsting GPS location providers
- Simple Mobile app ( for trackees )
- Browser based interface ( mainly for trackers and users ).


## How it works
- There is a flag in the API app setting to sync with RDBMS [Refer install doc](	/install.md) 
- We have following FT indexes for RediSearch

###Note the indexes and how they are called
```
############################
#InDexes redisearch        #
############################
#FT.CREATE idx:users ON hash PREFIX 1 "users:" SCHEMA searchable_email TEXT SORTABLE phone TEXT SORTABLE password TEXT SORTABLE
#searchable email is created to avoid issues with "@" in email
users_idx = Client('idx:users', conn=RedisClient)
#FT.CREATE idx:trackerlist ON hash PREFIX 1 "users:" SCHEMA isTracker TEXT SORTABLE exposed TEXT SORTABLE
#this is used to send tokens to logged in users
trackerList_idx = Client('idx:trackerlist', conn=RedisClient)
#FT.CREATE idx:trackers ON hash PREFIX 1 "users:" SCHEMA Trackers TEXT SORTABLE
# this is to search the hashes for all users who have tracker as me to be added in Locations(GET) api
trackers_idx = Client('idx:trackers', conn=RedisClient)
###############################
```

1. Register 
	1. User enters details and submits
		- *check if user exists on redis*
		```
		users_idx.search("{}|{}".format(email, phone))
		```
		- *check if user exists in db if dbsync is enabled*
		- *if exists send error*
	2. We create an hash key  (users:"phone number of user") and upload all data
		```
		RedisClient.hset('users:{}'.format(key), mapping=user_data)
		```
	3. Data looks like
		- users {
			- email: text, as in id@domain.com,
			- phone: text, 
			- password: text ( hashed ),
			- Token: text , 
			- isTracker: "True" or "False" , 
			- exposed: "True" or "False" , 
			- searchable_email: stored as fn_ln_domain_com,
			- alias: text , 
			- Verification_Code: text (random generated),
			- email_verified: "True" or "False", 
			- Trackers: "[list of trackers]", 
			- Location: "[longi, lati]", 
			- last_update: date as string
			}
		- Trackers and Locations are arrays stored as string
		- password is sha512 hashed
	5. if RDBMS sync is on , data is written to rdbms else its all upto redis
	4. Send Confirm email link to email


2. VerifyEmail
	1. The user clicks on the link Verification_Code is changed to a new one ( To avoid reuse ).
	```
	db_Veri_Code= RedisClient.hget("users:{}".format(phone),'Verification_Code')
	 ```
	3. if codes match email_verified is set to True and 
	```
	RedisClient.hset('users:{}'.format(phone), mapping={"email_verified":"True", "Verification_Code":new_code})
	```
	2. Success and failure message given as required
	3. if RDBMS sync is on , data is written to rdbms else its all upto redis


3. Login 
	1. User is authenticated
		- Check if user exists on redis
		```
		RedisClient.hexists("users:{}".format(phone),'password')
		```
		- Check if user exists in db if dbsync is enabled
		- If in db and not un redis pull gfrom dn and add
		```
		RedisClient.hset('users:{}'.format(phone), mapping=user_data)
		```
		- If does exists send error
		- The authe happens on all 3 : phone, email and password
		*we use redis search here on idx:users*
		```
		`users_idx.search("@searchable_email:{}  @phone:{} @password:{}".format(searchable_email, phone, password))
		```
	2. If success send user details along with tokens ( phone, email, alias, token, isTracker)



4. Load the UI
	-  ![screen](https://locatorservices.ngintec.com/ss/usersettings.png) 

5. For All calls needing registered user authentication is done based on token and id
	```
	RedisClient.hget("users:{}".format(phone),'Token')
	```

6. Once the user turns on  tracking switch.

	1. Trackee
		- Note: Trackee has a [mobile app](https://github.com/ngintec/trackernginmobileapp.git) as well 
		- ![Interface for  Trackee](https://locatorservices.ngintec.com/ss/trackee.png)
		- Trackee's need to add their trackers .they can have multiple trackers.
			- ![screen](https://locatorservices.ngintec.com/ss/addtracker.png) ![screen](https://locatorservices.ngintec.com/ss/viewtrackers.png) ![screen](https://locatorservices.ngintec.com/ss/deletetracker.png)

			```
			#fetch existing trackers list and add the new one to list
			existing_trackers= ast.literal_eval(RedisClient.hget("users:{}".format(phone),"Trackers").decode("utf-8"))
		    if tracker not in existing_trackers:
		        existing_trackers.append(tracker)
		    #Update back the list
		    RedisClient.hset("users:{}".format(phone),key='Trackers',value=str(existing_trackers))
			```

			- Note: Same concept is followed to delete the trackers from the list

		- Fetches the location from the device ( mobile , browser)

		- Makes an api call at the requested Update frequency ( default 60s ) and set the location
			```
			RedisClient.hset("users:{}".format(phone),key='Location',value=str(location))
			```
			- *updateFrequency can be changed on the fly and is not stored in backend* 
			- ![screen](https://locatorservices.ngintec.com/ss/updateFrequency.png)

		- Trackee can update alias --> this is valid when same drivers drive different bus numbers.
		![screen](https://locatorservices.ngintec.com/ss/updatealias.png)
		```
		RedisClient.hset("users:{}".format(phone),key='alias',value=alias)
		```

	2. Tracker
		- ![Interface for  Tracker](https://locatorservices.ngintec.com/ss/tracker.png)

		- Fetches the location from the device ( mobile , browser)

		- Makes an api call at the requested Update frequency ( default 60s )
			- *updateFrequency can be changed on the fly and is not stored in backend*
			```
			RedisClient.hset("users:{}".format(phone),key='Location',value=str(location))
			```

		- Also fetches the location of all Trackees from REDIS and puts markers on the graph.
			- *we use redis search here on idx:trackers*
			```
			trackers_idx.search(phone)
			```

		- Tracker can invite users and Filter from the list of current users whom he want to see.
			- ![screen](https://locatorservices.ngintec.com/ss/invite.png) ![screen](https://locatorservices.ngintec.com/ss/filter.png)


7. Tracker and Trackee can message each other and other ( all ) in case of emergencies.
	* uses redis pubsub and Server side events * 
	```
	#publish a message
		RedisMq.rpush("messages:{}".format(Msg_to), json.dumps(message))
	#read a message
		RedisMq.lpop("messages:{}".format(phone))
	```

	- ![screen](https://locatorservices.ngintec.com/ss/sendone.png) ![screen](https://locatorservices.ngintec.com/ss/receiveone.png)
	- ![screen](https://locatorservices.ngintec.com/ss/sendmany.png) ![screen](https://locatorservices.ngintec.com/ss/receivemany.png)


8. Logout
	1. Clear all stored credentials

## For Nomal users 

- ![Interface for  User](https://locatorservices.ngintec.com/ss/user.png)

1. No login is required

2. They choose the service from list
	- The list is obtained using redisearch idx:trackerlist

	```
	trackerList_idx.search("@isTracker:True  @exposed:True")
	```

	- Only services flagged as exposed  and isTracker=True are shown in list

3. on Search GEO radius is used for services with in 50KM and top 5 are returned

	```
	#below longitude and latitude is of the searching user
	RedisClient.georadius(service, longitude, latitude, 50, unit="km", withdist=True, withcoord=True, count=5, sort="ASC")
	```

	- ![Search Result](https://locatorservices.ngintec.com/ss/search.png)
