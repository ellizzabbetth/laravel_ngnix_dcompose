

Adding PHP Container:
 
The other important thing is the port on which this PHP interpreter listens for work you could say.
And this port is actually defined here in nginx conf.

Nginx is sending  PHP requests to port 3000.
Nginx.conf line 12, I have an address.

nginx.conf line 12
php:3000
127.0.0.1:3000

utilizing the fact that all the containers created in one docker compose file
are part of the same network and can discover each other by name.




PHP container exposes 9000 according to docs.
In Dockerfile:
    PHP:
        ports: 
            - '3000':'9000'      
            Map: host_machine_port:container_port   

So by the names specified here as services, as long as the code executes 
inside of the container, which definitely will be the case here.

So here I'm referring to the PHP container by using its name, php.
The nginx conf  wants to send my request to handle some PHP file to port 3000.

And that means that here in docker compose, I wanna ensure that this port is being used.

Now by looking into the docker file, of this official PHP image,
which I opened here on GitHub, we see that this image actually exposes port 9000 internally.

So that is what will be exposed by the container.
And again, I'm on the GitHub page of this PHP Docker image,
and then navigate it down into some folders to view the docker file of the image we're using.

So the PHP container exposes port 9000. (so nginx exposes port 3000)

I actually expect port 3000 from nginx to php,
and therefore here, we have to map 3000 to 9000.

We have our host machine first and then we have our container internal port.
However, this is now the tricky part. Is this really something we need here?

Keep in mind that in the end, it's our nginx server which wants to talk to this PHP container.
Which is why we're referencing the php container by name in the nginx conf.

Well, if that's the case, nginx.conf 3000 port is going straight to the container
and it's not going to be sent from our local host machine.
So this mapping won't do anything here.

Hence, whilst we can map here
in case we want to interact with that PHP container
from our host machine, we actually don't need to do that
just to send a request here.

Instead we should change this in the nginx config
to port 9000.

And I just want to emphasize why we're doing that,
because we have direct container to container communication.
Not through our local host.
So with that changed and yet nginx config,
we are now done and we finished this PHP container.


Any requests to the php service are sent directly from the server (i.e. nginx) service - NOT from our local machine.

So a ports: entry mapping ‘3000:9000’ (which Max adds and then removes from the docker-compose.yaml file) wouldn’t do anything to facilitate any direct communication from the server container to the php container. 

However, the problem is that, at the beginning of this lecture, the server service sends requests for the php service to port 3000 (as defined in the nginx.conf file) but the php service expects requests on port 9000 (as defined in its Dockerfile).

So, how do you fix that?  According to the nginx.conf file (see entry: fastcgi_pass php:3000;), all requests which come from our server (nginx) service in turn get sent straight to the php service on port 3000. 

So we need to change the entry fastcgi_pass php:3000; in the nginx.conf file to fastcgi_pass php:9000;. With this edit, the nginx service (i.e. the service named server in our docker-compose.yaml file) will now send requests directly to the php service on port 9000, which fixes their communication problem - since the php container (in its Dockerfile) expects input to come in on port 9000.

TL;DR Our local host has nothing to do with the direct communication between web server and php.  Communication via ports shouldn’t be defined in our php service in the docker-compose.yaml file, but instead should be defined in the nginx.conf file, for the server service, to match the port that the official PHP image exposes in its Dockerfile.

Original  <host_port>:<container_port>
---------------------------------
nginx:      php:3000 or 9000:3000
php service port mapping in Dockerfile: 3000:9000

Change
--------------------------------
nginx:      9000:9000
php service port mapping in Dockerfile: removed


nginx server wants to talk to php container (direct container to container communication) not through localhost.
From the php docs we see that php exposes port 9000.

docker-compose run --rm composer create-project --prefer-dist laravel/laravel .

docker-compose run artisan

=================================


The server is our main entry point which will serve the application and which will then forward requests
to the PHP interpreter. And the PHP interpreter will eventually indirectly talk to the MySQL database,
because in our code we will connect to that.

But we'll have a problem here. Our server, which is the main entry point currently
doesn't know anything about our source code. Our interpreter does but that's not enough.

The incoming request hits our server first, and it only forwards requests to PHP files to the PHP interpreter.

That means that these PHP files need to be exposed to the server and that simply means that we need
to add an extra volume in the nginx docker service.

And that extra volume, will be another bind mount where I bind my source folder
to the var/www/html folder inside of the container. And I'm picking this folder because in the nginx config,
it's the var/www/html folder from which we're serving our content and where we are looking for files.
Well, of course we're looking in the var/www/html/public folder but that is a folder 
which exists in the src folder: src/public

Bind src folder to volume in nginx

  server:
    image: 'nginx:stable-alpine'
    ports: 
      - '8000:80'
    volumes:
      - ./src:/var/www/html
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro


Run server, php, and mysql services in detached mode:
    docker-compose up -d server php mysql