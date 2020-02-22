# TLS only
## 1. Create server certs
```bash
cd tls
```

create > 
```bash
sudo openssl req -x509 -nodes -days 365 -subj '/C=IN/ST=Bangalore/L=ECity/O=Subrata POC/OU=POC/CN=subratapoc' -newkey rsa:2048 -keyout nginx.key -out nginx.crt
```

verify > 
```bash
openssl x509 -in nginx.crt -text -noout
```

## 2. Setup Nginx Conf
```
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/nginx.crt;
    ssl_certificate_key /etc/ssl/nginx.key;
    server_name localhost;
    server_tokens off;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
## 3. Setup docker
```
FROM nginx:latest
COPY default.conf /etc/nginx/conf.d/
COPY nginx.crt /etc/ssl/
COPY nginx.key /etc/ssl
COPY sample.html /usr/share/nginx/html
```

```bash
docker build -t nginx:latest .
docker run -p 8123:80 -p 8124:443 --name nginx-ssl -tid nginx-ssl
docker ps
docker exec -it<container-id> bash
```
## 4. Test
SSL Works 
```bash
curl -k https://localhost:8124/sample.html
```
expected output>>
```
<<!DOCTYPE html>
<html>
<body>

<h1>My First NGINX POC</h1>

<p>I am protected with TLS :)</p>

</body>
</html>
```

Verify certificate is matched as you created in step 1
```bash
openssl s_client -connect localhost:8124 -servername subratapoc 2>/dev/null
```

# Mutual TLS
## 1. Create client certs
```bash
cd mtls
```
create > 
```bash
sudo openssl req -x509 -nodes -days 365 -subj '/C=IN/ST=Bangalore/L=ECity/O=Subrata POC Clinet/OU=POC/CN=subratapocclient' -newkey rsa:2048 -keyout nginx-client.key -out nginx-client.crt
```
verify >
```bash
openssl x509 -in nginx-client.crt -text -noout
```
## 2. Setup Nginx Conf
```
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/nginx.crt;
    ssl_certificate_key /etc/ssl/nginx.key;
    ssl_client_certificate /etc/ssl/nginx-client.crt;
    ssl_verify_client on;
    server_name localhost;
    server_tokens off;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
## 3. Setup docker
```FROM nginx:latest
COPY default.conf /etc/nginx/conf.d/
COPY nginx.crt /etc/ssl/
COPY nginx.key /etc/ssl
COPY nginx-client.crt /etc/ssl
COPY sample.html /usr/share/nginx/html
```
```bash
docker build -f Dockerfile -t nginx-mtls:latest .
docker run -p 8133:80 -p 8134:443 --name nginx-mtls -tid nginx-mtls
docker ps
docker exec -it<container-id> bash
```
## 4. Test
Only TLS does not work anymore 
```bash
curl -k https://localhost:8134/sample.html
```
expected output>>
```<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx</center>
</body>
</html>
```
With cert & key works
```bash
$curl -k https://localhost:8134/sample.html --cert nginx-client.crt --key nginx-client.key
```

expected output>>
```<!DOCTYPE html>
<html>
<body>

<h1>My First NGINX POC</h1>

<p>I am protected with Mutual TLS :)</p>

</body>
</html>
```
Verify certificate is matched as you created in step 1
```bash 
openssl s_client -connect localhost:8134 -servername subratapoc 2>/dev/null
```
You should below along with other details
```
Acceptable client certificate CA names
/C=IN/ST=Bangalore/L=ECity/O=Subrata POC Clinet/OU=POC/CN=subratapocclient
Server Temp Key: ECDH, X25519, 253 bits
```

## 5.Notes

https://scmquest.com/nginx-docker-container-with-https-protocol/

https://www.sslshopper.com/article-most-common-openssl-commands.html