# matrix-riot-docker
an all-in-one docker compose setup for a personal riot client and matrix server

# RECOLIC SETUP GUIDE

## 1  Setup nginx

```
    server {
        listen       443 ssl http2;
        server_name chat.recolic.net chat.recolic.org;
        server_tokens off;
        client_max_body_size 8192M;

        ssl_certificate "/root/.acme.sh/drive.recolic.net/fullchain.cer";
        ssl_certificate_key "/root/.acme.sh/drive.recolic.net/drive.recolic.net.key";
        ssl_session_cache shared:SSL:50m;
        ssl_session_timeout  1d;
        ssl_ciphers EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:!MD5;
        ssl_prefer_server_ciphers on;

        # pass requests for dynamic content to rails/turbogears/zope, et al
        location / {
          proxy_set_header        Host $host;
          proxy_set_header        X-Real-IP $remote_addr;
          proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header        X-Forwarded-Proto $scheme;

          proxy_pass          http://127.0.0.1:3002;
          proxy_read_timeout  90;

          proxy_redirect      http://127.0.0.1:3002 https://chat.recolic.org;
        }

        location /riot/ {
          proxy_set_header        Host $host;
          proxy_set_header        X-Real-IP $remote_addr;
          proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header        X-Forwarded-Proto $scheme;

          rewrite ^/riot(/.*)$ $1 break;

          proxy_pass          http://127.0.0.1:3003;
          proxy_read_timeout  90;

          proxy_redirect      off;
        }

    }

```

## 2 Setup Matrix-Backend and Riot-Web frontend.

```
###################### BEGIN generate synapse config
vim docker-compose.yml
# Edit docker-compose.yml, Uncomment `entrypoint: ["/start.py", "migrate_config"]`

docker-compose up -d ; docker-compose down
vim docker-compose.yml
# Edit docker-compose.yml, Comment `entrypoint: ["/start.py", "migrate_config"]` again
###################### END

###################### BEGIN fix locale
docker-compose up -d
docker exec -ti matrixriotdocker_db_1 psql -U matrix

# Then run the following sql
CREATE DATABASE synapse
 ENCODING 'UTF8'
 LC_COLLATE='C'
 LC_CTYPE='C'
 template=template0
 OWNER matrix;
# SQL end

docker-compose down ; docker-compose up -d
###################### END
```

## 3 Further bugfix

> https://github.com/vector-im/riot-web/issues/3329

To make the integration server working, you still need to setup ONE OF the following options. Visit `https://federationtester.matrix.org/api/report?server_name=chat.recolic.org` when you're done, to confirm if you have done everything well.

1. **NOT-RECOMMENDED** Edit files/homeservers.yml, add the following content

```
listeners:
  - port: 8448
    tls: true
    type: http
    resources:
      - names: [federation]
        compress: false
```

Then expose 8448:8448 in docker-compose.yml.

2. **RECOMMENDED** Since most of us is using nginx proxy, we should set DNS record

```
# SRV doesn't allow CNAME...
chat.recolic.org A 34.80.xxx.xxx
_matrix._tcp.chat.recolic.org SRV   xx xx 443 chat.recolic.org
```

Then the matrix.org federationtester knows that, he should connect `https://chat.recolic.org:443`, rather than `https://34.80.xxx.xxx:8448`.




usage
=====
Update nginx.conf and nginx.init.conf file by replacing example.com references with your domain

Fill out the .env file with your fully qualified domain name and postgres password

If you need to generate certs via letsencrypt first you'll want to start the init docker compose script:
```
docker-compose -f docker-compose.init.yml up
```
then run renew certs with your fully qualified domain name
```
./renew-cert.sh example.com
```
this will hit your nginx proxy with the correct endpoints to authenticate with lets-encrypt, filling out your certs and certs-data folders if successfull.

Then you just run
```
docker-compose up
```
to startup everything
