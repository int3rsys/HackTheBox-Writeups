In the un-authenticated version we used port 80 and utilized se of exploits to gain user & root access. In this version, 
we will be exploiting the machine through express.js application and sql injection. 
I'm not familiar with express.js so I had to spend a couple of hours understanding what is it, how you query a server that runs
an express application and how to do that. Well, to sum things up - querying an express.js server is quite the same you do with a regular
server that supports RESTFUL api, thus it means we can interact with it via GET and POST (for this server) methods. Initially I wasn't able to get anything
