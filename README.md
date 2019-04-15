# Graylog content pack for nginx using JSON logging

This is directly based on the Graylog2 [nginx-docker content pack](https://github.com/ronlut/graylog-content-pack-nginx-docker).

It is designed for people using nginx(1.11.8+) in a docker container.

### Content Pack provides
  * 1 GELF UDP input for both nginx error and access logs, sent from Docker directly, with the following Extractors:
    * Extract JSON fields from gelf message
    * Reduce message to path
    * Lookup Remote Address Geolocation
    * [Error log] extract fields
    * [Error log] Reduce error log to message only
  * 6 streams to sort the log messages
    * nginx - All requests that were logged into the nginx access_log or nginx_error_log
    * nginx requests - All requests that were logged into the nginx access_log
    * nginx errors - All requests that were logged into the nginx error_log
    * nginx HTTP 404s - All requests that were answered with a HTTP 404 by nginx
    * nginx HTTP 4XXs - All requests that were answered with a HTTP code in the 400 range by nginx
    * nginx HTTP 5XXs - All requests that were answered with a HTTP code in the 500 range by nginx
  * 1 Lookup table for the Geolocation
    * 1 Data Adapter - configured to use MaxMind city database(see notes below).
    * 1 Lookup Cache - To provide in-memory caching for the MaxMind lookups
  * 1 Grok pattern not included in the default pack
    * IPORHOSTORUNDERSCORE
  * 1 Dashboard
    * Displays counts and graph of requests and response codes in the last hour and 24 hours.
    * Maps the requests based on Geolocation data from MaxMind.

### Design points

  * Docker GELF driver: The advantage of using docker's GELF driver is that you get a LOT of extra information you'll otherwise (e.g. syslog) won't get.
    * List of additional metadata fields you're getting when using docker's GELF driver [(source)](https://www.graylog.org/post/centralized-docker-container-logging-with-native-graylog-integration):
      * Hostname – Name of the Docker host
      * Container ID – Full ID of the container
      * Container Name – Human readable name of the container
      * Image ID – ID of the image used to create this container
      * Image Name – Human readable image name
      * Command – Command or entrypoint that is executed inside of the container
      * Tag – A tag that was given on creation time to identify containers easily
      * Creation time – A timestamp when this container was started
      * Log level – Was the message send to STDOUT or STDERR?

  * JSON log output: The core advantage of using JSON is that you can add arbitrary fields to the nginx logging and they will be mapped in Graylog by the extractor rather than having to delve into complex regex expressions to do things.

### Configuring nginx

# Note: You need to run at least nginx version 1.11.8 for escaped JSON support.

The following log format can be placed into the http block of the nginx configuration, either by placing it in the /etc/nginx/nginx.conf file or adding it to an included file under /etc/nginx/conf.d/ , before the server block. The access_log and error_log directives can either go in the http block or the server block(best in the http block, so it is global).

After the format and directives are in place, just reload the nginx configuration with: `nginx -s reload`

This configuration will send various NGINX variables to Graylog. You can log other useful information for each request by adding any other [NGINX variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables) into the JSON.

**nginx access log format:**

```
  log_format gelf_json escape=json '{ "timestamp": "$time_iso8601", '
         '"remote_addr": "$remote_addr", '
         '"connection": "$connection", '
         '"connection_requests": $connection_requests, '
         '"pipe": "$pipe", '
         '"body_bytes_sent": $body_bytes_sent, '
         '"request_length": $request_length, '
         '"request_time": $request_time, '
         '"response_status": $status, '
         '"request": "$request", '
         '"request_method": "$request_method", '
         '"host": "$host", '
         '"upstream_cache_status": "$upstream_cache_status", '
         '"upstream_addr": "$upstream_addr", '
         '"http_x_forwarded_for": "$http_x_forwarded_for", '
         '"http_referrer": "$http_referer", '
         '"http_user_agent": "$http_user_agent", '
         '"http_version": "$server_protocol", '
         '"remote_user": "$remote_user", '
         '"http_x_forwarded_proto": "$http_x_forwarded_proto", '
         '"upstream_response_time": "$upstream_response_time", '
         '"nginx_access": true }';
```

**nginx access_log and error_log directives:**
```
  access_log  /var/log/nginx/access.log  gelf_json;
  error_log  /var/log/nginx/error.log warn;
```

### Docker usage

I recommend using the [official nginx image](https://hub.docker.com/_/nginx) or a derivative of it. If you do that, the /var/log/{access,error}.log files should already be linked to the right place. If you build your own, you should check the nginx [Dockerfile](https://github.com/nginxinc/docker-nginx/blob/8921999083def7ba43a06fabd5f80e4406651353/mainline/jessie/Dockerfile#L21-L23) for an example.

**CLI**

To launch using a docker run command:

    `docker run -d --name <InstanceName> --log-driver=gelf --log-opt gelf-address=udp://<GraylogIP>:12401 --log-opt tag=<OptionalTag> <ImageName> <OptionalCommand>`

Examples:
    `docker run --rm --log-driver=gelf --log-opt gelf-address=udp://<GraylogIP>:12401 --log-opt tag=HelloWorld busybox echo Hello Graylog`
    `docker run -d --name nginx --log-driver=gelf --log-opt gelf-address=udp://<GraylogIP>:12401 --log-opt tag=prod_frontend -v /path/to/vhost/configs:/etc/nginx/conf.d nginx:latest`

**docker-compose.yml**

    image: nginx:latest
    volumes:
      - /path/to/vhost/configs:/etc/nginx/conf.d
    logging:
        driver: gelf
        options:
            gelf-address: "udp://<GraylogIP>:12401"
            tag: optional_tag
