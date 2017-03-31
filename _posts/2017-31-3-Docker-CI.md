

# Using private docker registry alternatives

# Summary:

Running containers within containers and using private registries implie configuration burden and try/error approach. While using a private registry works fine pushing from the host to the container, the problems arise where you want to push images from another container.

Under MAC the main issue found was that was possible to push images using
push localhost:5000/<myimage>

However this was not possible from the container itself
push registry:5000/<myimage>

Several options were tried including  using insecure registries, registries proxied by nginx to setup TLS negotiation, and using
registry images using certs and CA crts.

Eventually the simples solution is based on hosting images in private repositories in DockerHub.

I will explore the alternatives available and share some instructions that I hope will be useful to someone.

## 1. Run a private docker registry as docker container

### 1.1 Running a registry (non-secure)
```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```
You need to add the --insecure-registry flag /etc/default/docker.

In a MAC docker runs on a internal VM running alpine accesible using

```bash
screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
ctr+a+d EXIT
```

NOTE: Insecure registries does not support login/password access

But this approach will not allow remote push from the container :(

### 1.2 Running a private registry

#### 1.2.1 Using SSL certs accesible from the docker registry

First we need to generate self-signed certs
```bash
docker run --rm -v /tmp/registry/ssl:/certs -e SSL_DNS=registry paulczar/omgwtfssl
```
The host directory where the certs are  will be mapped as a volum with the registry once container is started, so registry pick up the SSL config

The registry.env file should look like

```
Contents of the file
# location of registry data
REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/tmp/registry/data

# location of TLK key/cert
REGISTRY_HTTP_TLS_KEY=/tmp/registry/ssl/key.pem
REGISTRY_HTTP_TLS_CERTIFICATE=/tmp/registry/ssl/cert.pem

# location of CA of trusted clients
REGISTRY_HTTP_TLS_CLIENTCAS_0=/tmp/registry/ssl/ca.pem
```bash

Inspiration from http://tech.paulcz.net/2016/01/deploying-a-secure-docker-registry/

```bash
docker run -d --name registry -v /tmp/registry:/opt/registry -p 5000:5000 --restart always --env-file /tmp/registry/config/registry.env registry:2
```

#### 1.2.2 Runnning a secured docker Registry with nginxproxy

You need to copy cert files, created in the previous steps
```bash
/tmp/registry/ssl » mkdir -p /tmp/security/external                                                                              mfarache@OEL0043
/tmp/registry/ssl » cp cert.pem /tmp/security/external                                                                           mfarache@OEL0043
/tmp/registry/ssl » cp key.pem /tmp/security/external

```

As we are going to be protecting our registry with simple http auth lets create a htpasswd. That could be done using some fancy docker image but this time I went through the simplest path

Visit http://www.htaccesstools.com/htpasswd-generator/

Assuming you use registry:XXXXXXX
copy the results so your file looks like

```
/tmp/security/external » cat docker-registry.htpasswd                                                     registry:$apr1$Rtbdcp0T$M/mH/oUjyBNRsrOSTrQUz.
```

So now we start a registry and a nginx proxy to provide upstream TLS negotiation
```bash
docker run -d --name registry -v /tmp/registry:/registry -e "SETTINGS_FLAVOR=local" -e "STORAGE_PATH=/registry" registry
```

```bash
docker run -d -p 443:443 -v /tmp/security/external:/etc/nginx/external --link registry:registry --name nginx-registry-proxy marvambass/nginx-registry-proxy
```
Once our containers are up and running

```bash
/tmp/security/external » docker login https://localhost:443                                                                      mfarache@OEL0043
Username: registry
Password:
Login Succeeded
```

Wohoo!!!! so now we can interact with it pushing/pulling images in a secure way.

## 2. Using a private DockerHub repository
Create an account in DockerHub, i.e dockermau using your email.

To see that everything works
```bash
~ » docker login --username=dockermau                                                                                     
Password:
Login Succeeded

docker tag busybox dockermau/busybux
docker push dockermau/busybux

The push refers to a repository [docker.io/dockermau/busybux]
c0de73ac9968: Mounted from library/busybox

Pushin to remote DockerHub registry

latest: digest: sha256:32f093055929dbc23dec4d03e09dfe971f5973a9ca5cf059cbfb644c206aa83f size: 527
```

This solution works fine, the docker hub repository can be marked as private so only repository owner can pull/push images
To improve speed and reduce times, we can add a Squid cache container that would improve performance caching images near end

## 3. Using Amazon ECR
There is a great aritcle [here][1]

[1]: http://rancher.com/using-amazon-container-registry-service/

Luckily enough was happy with option 2, so there was not reason to get tied to AWS again. Dockerhub does the job pretty well.




Next post will be around setting a continuosu development pipeline using Jenkins.