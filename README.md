# tomcat-cluster-redis-as-session-manager

##1. Redis:

**Installation:**

Download, extract and compile Redis with:

	$ wget http://download.redis.io/releases/redis-3.0.0.tar.gz
	$ tar xzf redis-3.0.0.tar.gz
	$ cd redis-3.0.0
	$ make

The binaries that are now compiled are available in the src directory. Run Redis with:

	$ src/redis-server

##2. Storing sessions in redis:

For storing session ion redis, i am using Webapp Runner (https://github.com/jsimone/webapp-runner) which allow you to launch an exploded or compressed war that is on your filesystem into a tomcat container with a simple java -jar command.

**Usage:**

**Clone and Build:**

	git clone git@github.com:jsimone/webapp-runner.git
	mvn package

**Execute:**

	java -jar target/webapp-runner.jar path/to/my/project

**Store your sessions in redis:**

In versions 7.0.29.1 and newer support for a session manager that stores sessions in redis is built in.

To use it add --session-store redis to your startup command:

	$ java -jar target/dependency/webapp-runner.jar --session-store redis target/<appname>.war

*Then make sure that Redis environment variable is available for configuration: REDIS_URL*

**Running multiple instances :**

you can run multiple instances of your app by changing port with --port option (so that you can cluster them later using nginx as load balancer)
	
	java -jar target/webapp-runner.jar --port 8080 path/to/my/project
	java -jar target/webapp-runner.jar --port 8081 path/to/my/project

*Further reads: go to https://github.com/jsimone/webapp-runner for detailed documentation.*



##3. Nginx as load balancer to form cluster:

**Download:** download Nginx from
 
	 http://nginx.org/en/download.html

**Installation:**
After extracting the source, run these commands from a terminal:

	sudo ./configure
	sudo make
	sudo make install

By default, Nginx will be installed in /usr/local/nginx. You may change this and other options with the compile-time options .

**Tweak congiguration:**

lets setup the round robin load balancer. You would need to use the nginx upstream module. The upstream module allows you to group servers that can be referenced by the proxy_pass directive. In your nginx.conf, locate the http block and add the upstream block to create a group of servers:

	upstream tomcat  {
		server localhost:8080;
		server localhost:8081;
	    }

Finally, you need to locate the location block within the server block and map the root(/) location to the tomcat_servers group created using the upstream module above.

	location / {
	proxy_pass http://tomcat_servers;
	}

Thats it!. Restart nginx and you should now be able to send a request to http://localhost and the request will be served by one of the Tomcat servers.

**Running Nginx:**

	/usr/local/nginx/sbin/nginx 


##4. Testing entire system:

Make sample grails app with two action:

	def read() {
		println("i am here");
		render text: session.id+ "msg:"+session.msg

	    }

    	def write(){
		session.name=params.msg
		render text:'ok'+" "+session.msg
	    }	


create war.

Follow steps described above to setup running environment and deploy war.

Go to browser and hit "write" action.

	http://localhost:8080/hello/write?msg=hello

It will store this session into redis.

Now open another tab and hit "read" action 

	http://localhost/hello/read

You will see a value that you have setted up in your write action. To test it further just make one server down and you will notice your cluster is still up and running. You can go one level further by making all servers down and resuming them again and you will notice your previously persisted values are still available. 


