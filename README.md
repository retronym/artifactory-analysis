## Artifactory SSH proxy performance

### /etc/nginx.conf

```
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log;
#error_log  /var/log/nginx/error.log  notice;
#error_log  /var/log/nginx/error.log  info;

pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
    # Jason Zaugg dynamic IP
    debug_connection 124.170.132.122;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # TODO: keep access log for 90 days
    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    index   index.html index.htm;
    ...
}
```

### /etc/nginx/conf.d/jenkins.conf
```
limit_req_zone $binary_remote_addr zone=one:10m rate=1000r/s;

...

upstream artifactory {
  server 127.0.0.1:8081 fail_timeout=0;
}

# based on https://gist.github.com/plentz/6737338, among others
server {
  listen 443 ssl default deferred;
  server_name scala-ci.typesafe.com;

  ssl on;
  ssl_certificate /etc/nginx/ssl/scala-ci.crt;
  ssl_certificate_key /etc/nginx/ssl/scala-ci.key;
  # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
  ssl_dhparam /etc/nginx/ssl/dhparam.pem;

  # enable session resumption to improve https performance
  # http://vincent.bernat.im/en/blog/2011-ssl-session-reuse-rfc5077.html
  ssl_session_cache    shared:SSL:10m;
  ssl_session_timeout  10m;

  #enables all versions of TLS, but not SSLv2 or 3 which are weak and now deprecated.
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

  #Disables all weak ciphers
  ssl_ciphers 'AES128+EECDH:AES128+EDH:!aNULL';

  # enable ocsp stapling (mechanism by which a site can convey certificate revocation information to visitors in a privacy-preserving, scalable manner)
  # http://blog.mozilla.org/security/2013/07/29/ocsp-stapling-in-firefox/
  ssl_stapling on;
  ssl_stapling_verify on;
  resolver 8.8.4.4 8.8.8.8 valid=300s;
  resolver_timeout 10s;

  # config to enable HSTS(HTTP Strict Transport Security) https://developer.mozilla.org/en-US/docs/Security/HTTP_Strict_Transport_Security
  # to avoid ssl stripping https://en.wikipedia.org/wiki/SSL_stripping#SSL_stripping
  add_header Strict-Transport-Security "max-age=31536000; includeSubdomains;";

  # when serving user-supplied content, include a X-Content-Type-Options: nosniff header along with the Content-Type: header,
  # to disable content-type sniffing on some browsers.
  # https://www.owasp.org/index.php/List_of_useful_HTTP_headers
  # currently suppoorted in IE > 8 http://blogs.msdn.com/b/ie/archive/2008/09/02/ie8-security-part-vi-beta-2-update.aspx
  # http://msdn.microsoft.com/en-us/library/ie/gg622941(v=vs.85).aspx
  # 'soon' on Firefox https://bugzilla.mozilla.org/show_bug.cgi?id=471020
  add_header X-Content-Type-Options nosniff;

  # This header enables the Cross-site scripting (XSS) filter built into most recent web browsers.
  # It is usually enabled by default anyway, so the role of this header is to re-enable the filter for
  # this particular website if it was disabled by the user.
  # https://www.owasp.org/index.php/List_of_useful_HTTP_headers
  add_header X-XSS-Protection "1; mode=block";

  ssl_prefer_server_ciphers on;
  
  # "/artifactory" is determined by context path in node["artifactory"]["catalina_base"]/conf/Catalina/localhost/artifactory.xml
  # e.g.: <Context path="/artifactory" docBase="${artifactory.home}/webapps/artifactory.war" processTlds="false">
  location /artifactory {
    # limit_req               zone=one burst=10000 nodelay;

    proxy_set_header        Host $host;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto $scheme;
    proxy_redirect          http:// https://;
    proxy_pass              http://artifactory;

    proxy_connect_timeout   150;
    proxy_send_timeout      100;
    proxy_read_timeout      100;
    proxy_buffers           8 32k;

    client_max_body_size    128m;
    client_body_buffer_size 64m;
    # Temporarily set by Jason Zaugg on 2018.03.26 to diagnose intermittent performance problem in https proxy of artifactory.
    proxy_buffering off;
  }
```
