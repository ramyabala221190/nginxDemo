Purpose of nginx:

=>Serving static website files
=>Running a reverse proxy with load balancing
=>Caching and buffering responses
=>tls termination to support https
=>rewriting requests and responses

Installing nginx:

Easiest way is to pull the docker image


Listing out nginx processes in WSL

ps axf | grep nginx

DESKTOP-R3L1N8A:/mnt/host/c/WINDOWS/system32# ps axf | grep nginx
   10 root      0:00 grep nginx
DESKTOP-R3L1N8A:/mnt/host/c/WINDOWS/system32#


nginx.conf

/etc/nginx/nginx.conf

Below are the contents of the file

----------------------------------------------------------------------------
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
---------------------------------------------------------------------------------------------------------------------

Below directives are in the main context:

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


All the directives within the http {} are in the http context

In the file, we can include additional configuration files. Below file has been included:

    include /etc/nginx/conf.d/*.conf;

    We usually copy our custom configuration to the /etc/nginx/conf.d/default.conf


Below is the default.conf file. This file will come exactly where the include directive is placed in the nginx.conf file.
This means it will be inside the http context.


    server{
    listen 0.0.0.0:80;
    listen [::]:80;
    default_type application/octet-stream;

    gzip                    on;
    gzip_comp_level         6;
    gzip_vary               on;
    gzip_min_length         1000;
    gzip_proxied            any;
    gzip_types              text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_buffers            16 8k;
    client_max_body_size    256M;

    #add_header 'Access-Control-Allow-Origin' 'http://localhost:30004';
    add_header 'Access-Control-Allow-Origin' '*';


    root /usr/share/nginx/html;
    index index.html;

        location /assets/ {
         autoindex on;
        }

        location / {
        #try_files  /index.html =404;
        }
        error_page  404              /index.html;

    }

----------------------------------------------------------------------------------------------------
I am deploying to /usr/share/nginx/html/
When I change the base href to "/sub/", all the JS and css files are looked for within the /sub/ folder

I am hitting http://localhost:8081 in the browser.

http://localhost:8081/sub/styles.ef46db3751d8e999.css
instead of 
http://localhost:8081/styles.ef46db3751d8e999.css

But the index.html is still looked for in the / folder.


Now if I hit localhost:8081/sub in the browser, I get a 404 because there is no sub/

------------------------------------------------------------------------------------------------

Lets say I change Dockerfile
COPY --from=node /app/dist/nginx-demo /usr/share/nginx/html/sub/ but root remains /usr/share/nginx/html/

It means we have deployed the files to /sub/ folder.

When I hit localhost:8081/ in the browser, the autoindex on; esecutes and shows the list of files in /
because there is no index.html in /usr/share/nginx/html/

When I hit localhost:8081/sub in the browser, I get a 301 error

When I hit http://localhost:8081/sub/ in the browser, it works correctly.

-------------------------------------------------------------------------------------

If deploying to /sub, ensure the root also is /usr/share/nginx/html/sub
No need to give bas href="/sub/" in that case.

If deploying to /sub and the root is /usr/share/nginx/html
then base href="/sub/" to ensure that the files are looked within the sub folder within the root

---------------------------------------------------------------------------------------

Client ---->nginx---->Proxying request to backend service

multiple server {} running on different ports ----> proxying to different backend service

------------------------------------------------------------------------------------------

/app is the workdirectory.
We are deploying to /app/sub
COPY - from=node /app/dist/nginx-demo /app/sub
Scenario-I
The nginx root is root /app/sub and base href in index.html is /sub/

Hitting localhost:8081 in the browser
Observatations:
The root is /app/sub and we have deployed the project to /app/sub. The webserver will look for all the files within the root.
index.html was successfully sent because / was the URI and the webserver was able to find index.html within /app/sub/
The JS and CSS files failed with 404 because the URI was /sub/*.css OR /sub/*.js. This means nginx look for /sub/*.css or /sub/*.js within /app/sub. It will look for an additional folder sub within /app/sub and then look for the css/js within that additional sub folder within /app/sub.

How can I fix? 
If your nginx root and the deployment location matches, there is no need to add change your base href.
If you update your base href back to "/", this will correct the 404 errors. This means the *.css or *.js files will looked for within the root directly and not within any subfolder within the root.

Scenario-II
The files are deployed to /app/sub but the nginx root is /app and base href is /. This can be used if you are deploying multiple apps to the same nginx server but in multiple subfolders.
So 1 app is deployed to /sub. Another maybe is deployed to /sub2 etc.
When i hit localhost:8081, index.html is not found within the root /app and hence it displays the directory contents since autoindex directive was set.

When i hit localhost:8081/sub/, the index.html is successfully sent because nginx is looking for the sub folder within the /app folder, where the index.html file does does exist.
Note that the *.css or *.js files fail with 404 because nginx is looking for this files directly within the root /app where it wont be found.
To solve this error, it is important to set the base href to the correct value in the index.html.
So I have set the base href to "/sub/". The nginx root remains /app but the files are deployed to /app/sub.
This solution again will work with index.html only if you hit localhost:8081/sub/ in the browser because the nginx root is /app and nginx will be able to find the index.html within /app/sub only if the URI is /sub/.
So the browser url must be localhost:8081/sub/.
This solution will work automatically for css and js files because /sub/ will automatically be attached to the request because of the base href.

-----------------------------------------------------------------

docker compose --env-file ./dev.env up