In the un-authenticated version we used port 80 and utilized se of exploits to gain user & root access. In this version, 
we will be exploiting the machine through express.js application and sql injection. 
I'm not familiar with express.js so I had to spend a couple of hours understanding what is it, how you query a server that runs
an express application and how to do that. Well, to sum things up - querying an express.js server is quite the same you do with a regular server that supports RESTFUL api, thus it means we can interact with it via GET and POST (for this server) methods. Initially I wasn't able to get anything using a regular queries, so with using different fuzzers. I used a hint from that the {message:"blalala"} is referenced to a specific api. After a couple of searches I figured it's graphQL. From here, it was quite easy figuring the right query, after reading the following article:
https://blog.apollographql.com/4-simple-ways-to-call-a-graphql-api-a6807bcdb355

I used RESTClient for Firefox (you can also use cURL or any other client) with the following queries:
```
curl -X POST -H 'Content-Type: application/json' -i 'http://10.10.10.121:3000/graphql' --data '{"query" : "{ user {username} }"}'
curl -X POST -H 'Content-Type: application/json' -i 'http://10.10.10.121:3000/graphql' --data '{"query" : "{ user {password} }"}'
```
which gave me the username and password.
(quick clarification: I used json type query to get the username and password from the field 'user')
the password is hashed, it's readlly easy to get the real value here: https://crackstation.net/

Now we can authenticate with our credentials. Going through the source code of Helpdeskz we can see that there is a place where sql query is not filtered, specifically in view_tickets_controller.php in line 94:
```
$attachment = $db->fetchRow("SELECT *, COUNT(id) AS total FROM ".TABLE_PREFIX."attachments WHERE id=".$db->real_escape_string($params[2])." AND ticket_id=".$params[0]." AND msg_id=".$params[3]);
```
 tickets_id and msg_id are not filtered, thus we can try and injection sql queries. 
