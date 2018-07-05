## Step 1 - NGINX Proxy
In this scenario, we want to have a NGINX service running which can dynamically discovery and update it's load balance configuration when new containers are loaded. Thankfully this has already been created and is called nginx-proxy.

Nginx-proxy accepts HTTP requests and proxies the request to the appropriate container based on the request Hostname. This is transparent to the user with happens without any additional performance overhead.

##### Properties
There are three keys properties required to be configured when launching the proxy container.

The first is binding the container to port 80 on the host using `-p 80:80`. This ensures all HTTP requests are handled by the proxy.

The second is to mount the docker.sock file. This is a connection to the Docker daemon running on the host and allows containers to access it's metadata via the API. Nginx-proxy uses this to listen for events and then updates the NGINX configuration based on the container IP address. Mounting file works in the same way as directories using -v /var/run/docker.sock:/tmp/docker.sock:ro. We specify :ro to restrict access to read-only.

Finally, we can set an optional `-e DEFAULTHOST=<domain>`. If a request comes in and doesn't make any specified hosts, then this is the container where the request will be handled. This enables you to run multiple websites with different domains on a single machine with a fall-back to a known website.

#### Task
Use the command below to launch nginx-proxy.

```console
docker run -d -p 80:80 -e DEFAULT_HOST=proxy.example -v /var/run/docker.sock:/tmp/docker.sock:ro --name nginx jwilder/nginx-proxy
```

Because we're using a DEFAULT_HOST, any requests which come in will be directed to the container that has been assigned the `HOST proxy.example`.

##### Request
You can make a request to the web server using `curl http://docker`. As we have no containers, it will return a `503` error.

## Step 2 - Single Host
Nginx-proxy is now listening to events which Docker raises on start / stop. A sample website called `katacoda/docker-http-server` has been created which returns the machine name it's running on. This allows us to test that our proxy is working as expected. Internally it's a PHP and Apache2 application listening on port `80`.

##### Starting Container
For Nginx-proxy to start sending requests to a container you need to specify the `VIRTUAL_HOST` environment variable. This variable defines the domain where requests will come from and should be handled by the container.

In this scenario we'll set our `HOST` to match our `DEFAULT_HOST` so it will accept all requests.

```console
docker run -d -p 80 -e VIRTUAL_HOST=proxy.example katacoda/docker-http-server
```

##### Testing
Sometimes it takes a few seconds for NGINX to reload but if we execute a request to our proxy using `curl http://docker` then the request will be handled by our container.

## Step 3 - Cluster
We now have successfully created a container to handle our HTTP requests. If we launch a second container with the same `VIRTUAL_HOST` then `nginx-proxy` will configure the system in a round-robin load balanced scenario. This means that the first request will go to one container, the second request to a second container and then repeat in a circle. There is no limit to the number of nodes you can have running.

#### Task
Launch a second container using the same command as we did before.

```console
docker run -d -p 80 -e VIRTUAL_HOST=proxy.example katacoda/docker-http-server
```

##### Testing
If we execute a request to our proxy using `curl http://docker` then the request will be handled by our first container. A second HTTP request will return a different machine name meaning it was dealt with by our second container.

#### Step 4 - Generated NGINX Configuration
While nginx-proxy automatically creates and configures NGINX for us, if you're interested in what the final configuration looks like then you can output the complete config file with docker exec as shown below.

```console
docker exec nginx cat /etc/nginx/conf.d/default.conf
```

Additional information about when it reloads configuration can be found in the logs using `docker logs nginx`


