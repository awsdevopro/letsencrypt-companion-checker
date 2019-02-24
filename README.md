# letsencrypt-companion-checker

**letsencrypt-nginx-proxy-companion** is a lightweight companion container for [**nginx-proxy**](https://github.com/jwilder/nginx-proxy).

It handles the automated creation, renewal and use of Let's Encrypt certificates for proxyed Docker containers.

Please note that [letsencrypt-nginx-proxy-companion does not work with ACME v2 endpoints yet](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/issues/319).

### Features:
* Automated creation/renewal of Let's Encrypt (or other ACME CAs) certificates using [**simp_le**](https://github.com/zenhack/simp_le).
* Let's Encrypt / ACME domain validation through `http-01` challenge only.
* Automated update and reload of nginx config on certificate creation/renewal.
* Support creation of Multi-Domain (SAN) Certificates.
* Creation of a Strong Diffie-Hellman Group at startup.
* Work with all versions of docker.

### Requirements:
* Your host **must** be publicly reachable on **both** port `80` and `443`.
* Check your firewall rules and **do not attempt to block port `80`** as that will prevent `http-01` challenges from completing.
* For the same reason, you can't use nginx-proxy's [`HTTPS_METHOD=nohttp`](https://github.com/jwilder/nginx-proxy#how-ssl-support-works).
* The (sub)domains you want to issue certificates for must correctly resolve to the host.
* Your DNS provider must [answers correctly to CAA record requests](https://letsencrypt.org/docs/caa/).
* If your (sub)domains have AAAA records set, the host must be publicly reachable over IPv6 on port `80` and `443`.

![schema](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/blob/master/schema.png)

## Basic usage (with the nginx-proxy container)

Three writable volumes must be declared on the **nginx-proxy** container so that they can be shared with the **letsencrypt-nginx-proxy-companion** container:

* `/etc/nginx/certs` to store certificates, private keys and ACME account keys (readonly for the **nginx-proxy** container).
* `/etc/nginx/vhost.d` to change the configuration of vhosts (required so the CA may access `http-01` challenge files).
* `/usr/share/nginx/html` to write `http-01` challenge files.

Example of use:

### Step 1 - nginx-proxy

Start **nginx-proxy** with the three additional volumes declared:

```shell
$ docker run --detach \
    --name nginx-proxy \
    --publish 80:80 \
    --publish 443:443 \
    --volume /etc/nginx/certs \
    --volume /etc/nginx/vhost.d \
    --volume /usr/share/nginx/html \
    --volume /var/run/docker.sock:/tmp/docker.sock:ro \
    jwilder/nginx-proxy
```

Binding the host docker socket (`/var/run/docker.sock`) inside the container to `/tmp/docker.sock` is a requirement of **nginx-proxy**.

### Step 2 - letsencrypt-nginx-proxy-companion

Start the **letsencrypt-nginx-proxy-companion** container, getting the volumes from **nginx-proxy** with `--volumes-from`:

```shell
$ docker run --detach \
    --name nginx-proxy-letsencrypt \
    --volumes-from nginx-proxy \
    --volume /var/run/docker.sock:/var/run/docker.sock:ro \
    jrcs/letsencrypt-nginx-proxy-companion
```

The host docker socket has to be bound inside this container too, this time to `/var/run/docker.sock`.

### Step 3 - proxyed container(s) - Must jump to the Step 4 before running proxied container.

Once both **nginx-proxy** and **letsencrypt-nginx-proxy-companion** containers are up and running, start any container you want proxyed with environment variables `VIRTUAL_HOST` and `LETSENCRYPT_HOST` both set to the domain(s) your proxyed container is going to use.

[`VIRTUAL_HOST`](https://github.com/jwilder/nginx-proxy#usage) control proxying by **nginx-proxy** and `LETSENCRYPT_HOST` control certificate creation and SSL enabling by **letsencrypt-nginx-proxy-companion**.

Certificates will only be issued for containers that have both `VIRTUAL_HOST` and `LETSENCRYPT_HOST` variables set to domain(s) that correctly resolve to the host, provided the host is publicly reachable.

```shell
$ docker run --detach \
    --name your-proxyed-app \
    --env "VIRTUAL_HOST=subdomain.yourdomain.tld" \
    --env "LETSENCRYPT_HOST=subdomain.yourdomain.tld" \
    --env "LETSENCRYPT_EMAIL=mail@yourdomain.tld" \
    awsdevopro/apache-php56
```

Albeit **optional**, it is **recommended** to provide a valid email address through the `LETSENCRYPT_EMAIL` environment variable, so that Let's Encrypt can warn you about expiring certificates and allow you to recover your account.

The containers being proxied must expose the port to be proxied, either by using the `EXPOSE` directive in their Dockerfile or by using the `--expose` flag to `docker run` or `docker create`.

If the proxyed container listen on and expose another port than the default `80`, you can force **nginx-proxy** to use this port with the [`VIRTUAL_PORT`](https://github.com/jwilder/nginx-proxy#multiple-ports) environment variable.

Example using [apache-php56](https://hub.docker.com/r/awsdevopro/apache-php56) (expose and listen on port 8282):

```shell
$ docker run --detach \
    --name your-proxyed-app \
    --env "VIRTUAL_HOST=othersubdomain.yourdomain.tld" \
    --env "VIRTUAL_PORT=8282" \
    --env "LETSENCRYPT_HOST=othersubdomain.yourdomain.tld" \
    --env "LETSENCRYPT_EMAIL=mail@yourdomain.tld" \
    awsdevopro/apache-php56
```

Repeat [Step 3](#step-3---proxyed-containers) for any other container you want to proxy.


### Step 4 Troubleshooting - Once Step 1 and Step 2 are done.

**Must Check: Go to https://port-checker.info/ and check if 80 and 443 is open or not.**

* If you give Input 80 and 443 and click CHECK-PORT, Output as follows in green letters. If they are close, then red letters with closed for the host will appear. 
```
Port 80 is OPEN on host Public-IP-XX
Port 443 is OPEN on host Public-IP-XX 
```
* If you see green letters with open ports, you go get running proxied container in the step 3. 

* If you see red letters with closed ports you're stuck here. Please login to your wifi connected home router and forward 443<-->443 and 80<-->80 to the ubuntu:16.04 host. For me it's TP-Link router, I login to the router and forward it to host ubuntu:16.04 private IP <192.168.1.103>. If any issues, please write a question to <aalmamun.ece08@gmail.com> and give Title as "80,443 LETSENCRYPT ISSUE", otherwise, I'm counting as spam. Please provide more details in the email body, like OS, Router, Router Model, if possible with screenshots. By default "web management port" is set to 80, you need to change it other than 80. This way you can forward 80 to 80 only.



* Check nginx-proxy logs 
```
$ docker logs -f nginx-proxy

WARNING: /etc/nginx/dhparam/dhparam.pem was not found. A pre-generated dhparam.pem will be used for now while a new one
is being generated in the background.  Once the new dhparam.pem is in place, nginx will be reloaded.
forego     | starting dockergen.1 on port 5000
forego     | starting nginx.1 on port 5100
dockergen.1 | 2019/02/23 16:44:22 Generated '/etc/nginx/conf.d/default.conf' from 1 containers
dockergen.1 | 2019/02/23 16:44:22 Running 'nginx -s reload'
dockergen.1 | 2019/02/23 16:44:22 Watching docker events
dockergen.1 | 2019/02/23 16:44:22 Contents of /etc/nginx/conf.d/default.conf did not change. Skipping notification 'nginx -s reload'
dockergen.1 | 2019/02/23 16:44:34 Received event start for container 97cdde6d2714
dockergen.1 | 2019/02/23 16:44:34 Contents of /etc/nginx/conf.d/default.conf did not change. Skipping notification 'nginx -s reload'
2019/02/23 16:44:34 [notice] 51#51: signal process started
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
dhparam generation complete, reloading nginx
dockergen.1 | 2019/02/23 16:46:11 Received event start for container 91f07238711f
dockergen.1 | 2019/02/23 16:46:11 Generated '/etc/nginx/conf.d/default.conf' from 3 containers
dockergen.1 | 2019/02/23 16:46:11 Running 'nginx -s reload'
nginx.1    | subdomain.yourdomain.tld 66.133.109.36 - - [23/Feb/2019:16:46:32 +0000] "GET /.well-known/acme-challenge/H12ImVg58WCifFb6cq0rqJHSuYk7RH6lhk7O0-3fX14 HTTP/1.1" 200 87 "-" "Mozilla/5.0 (compatible; Let's Encrypt validation server; +https://www.letsencrypt.org)"


You're done!

Please browse your website at https://subdomain.yourdomain.tld
```
