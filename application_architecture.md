The applications within IHTSDO follow the same broad model


  User -> | Nginx | -> | Application |

# Nginx

Nginx acts as a proxy the application, providing TLS support where neccasery and optionally serving static files for the application. A typical site with TLS support and redirection from HTTP to the SSL would be as follows.

```
# Redirect HTTP traffic to HTTPS, acting as default site
server {
  server_name www.testservice.com testservice.com;
  listen 80 default;
  rewrite ^ https://$host$request_uri permanent;
}

# Serve HTTPS traffic for the given host and the default if the Host header
# doesn't match another server block
server {
  server_name www.testservice.com testservice.com;
  listen 443 ssl default;

  # SSL Configuration. This section aim to get an A rating at https://www.ssllabs.com/ssltest
  ssl_certificate     /etc/ssl/certs/testservice.crt;
  ssl_certificate_key /etc/ssl/certs/testservice.key;
  ssl_dhparam /etc/nginx/dhparam.pem;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_ciphers 'EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS';
  ssl_session_cache shared:SSL:50m;
  ssl_session_timeout 5m;

  add_header Strict-Transport-Security max-age=15768000;

  # Server static assets
  location / {
      rootdir /srv/http/testservice.com;
  }

  # Pass requests to anything under /app_path via the proxy to the listening application
  location /app_path {
      proxy_pass http://localhost:10010/app_path
      proxy_http_version 1.1;
	  # Set header for the application to use
      proxy_set_header Connection "";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto "https";
      proxy_set_header X-Url-Scheme $scheme;
	  # General tuning
      proxy_connect_timeout 150;
      proxy_send_timeout 100;
      proxy_read_timeout 100;
      proxy_buffers 4 32k;
      client_max_body_size 8m;
      client_body_buffer_size 128k;
      proxy_busy_buffers_size    64k;
      proxy_temp_file_write_size 64k;
	  # Redirect http:// responses to https://
      proxy_redirect http:// https://;
  }
}
```

For test purposes, generate a self signed certificate and configure accordingly. Your application can use headers passed to it to ascertain the nature of the incoming connection to, for example, return correct URLs to the client.

Note that where an application bundles it's own nginx configuration this will usually just be a basic configuration to allow the application to run to some degree on deployment.

```
server {
        listen          80 default;
        server_name testservice;
        sendfile off; # Avoid file descriptor trouble in a virtual machine environment

        location /mapping-rest {
	        client_max_body_size    2048m;
	        proxy_set_header        Host    $http_host;
	        proxy_pass                              http://localhost:8080/test-service;
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location / {
                root /srv/http/otf-mapping-service-web;
        }
}
```

The actual configuration will usually be put in place, when required, by the projects [Ansible role](https://github.com/IHTSDO/ihtsdo-ansible/tree/master/roles).

# Java application configuration

The application should run on a given port - see the [service ports document](service_ports.md) for assignments, adding your application if neccasery.

Maven is configured to produce stand alone jar files which only need a configuration file to run. To run this jar outside of the package run:

```sh
/usr/bin/java -Xms256m -Xmx512m -jar app.jar -Dyour.config.property.name=config.properties -httpPort=10010 -resetExtract -extractDirectory extract
```

Change your.config.property.name to the name used by your application to specific the configuration file. 

The application will continue to run in the foreground and listen on for HTTP on the port given by -httpPort=


You can also configure nginx to proxy directly to Tomcat. Make sure Tomcat does not listen on the same port as nginx and change the proxy_pass line in the nginx configuration accordingly.

