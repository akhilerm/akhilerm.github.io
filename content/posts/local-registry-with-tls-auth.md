---
title: "Deploying Local Registry with TLS and Auth"
date: 2024-11-24T16:14:55+05:30
draft: false
categories:
- Tech
- Blog
tags:
- registry
- docker
- tls
- containerd
- ctr
---

If you are working with container images, there will be cases where you want to have a proper registry setup locally
with TLS and authentication for testing as well as CI purposes. The below steps outline how a local registry can be 
deployed with a self signed certificate.

1. Create a directory for storing the registry data, certificates and authentication data.
```
mkdir -p ./my-docker-registry/{auth,certs,data}
```

2. Now create the authentication data using `htpasswd`
```
docker run --rm --entrypoint htpasswd httpd:2 -Bbn <user> <password>  > ./my-docker-registry/auth/htpasswd
```

3. Create a new file `./my-docker-registry/certs/domain.ext` with the below content for adding a Subject Alternative Name (SAN)
into the certificate which is required by modern clients.
```
[req]
default_bits       = 4096
distinguished_name = req_distinguished_name
req_extensions     = req_ext
x509_extensions    = v3_ca
prompt             = no

[req_distinguished_name]
C = US
ST = State
L = Locality
O = Organization
OU = Organizational Unit
CN = localhost

[req_ext]
subjectAltName = @alt_names

[v3_ca]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
IP.1 = 127.0.0.1
``` 

4. Now create the self signed certificate
```
openssl req -newkey rsa:4096 -nodes -sha256 -keyout ./my-docker-registry/certs/domain.key -x509 -days 365 -out ./my-docker-registry/certs/domain.crt -config ./my-docker-registry/certs/domain.ext
```

5. Copy the generated certificates into the trust store of docker, containerd (if you plan to use ctr) and system for the domain at
at which you will be starting the registry (`localhost:5000` in this case).
```
sudo mkdir -p /etc/docker/certs.d/localhost:5000
sudo mkdir -p /etc/containerd/certs.d/localhost:5000
sudo cp ./my-docker-registry/certs/domain.crt /etc/docker/certs.d/localhost:5000/ca.crt
sudo cp ./my-docker-registry/certs/domain.crt /etc/containerd/certs.d/localhost:5000/ca.crt
```

6. Restart containerd and docker
```
sudo systemctl restart containerd
sudo systemctl restart docker
```

7. Create a `./my-docker-registry/config.yml` for the registry configuration
```
version: 0.1
storage:
  filesystem:
    rootdirectory: /var/lib/registry 
#Other optional configuration
#http:
#  tls:
#    minimumtls: tls1.3
``` 

8. Start the registry container at port 5000
```
docker run -d  -p 5000:5000 --restart=always --name registry   -v $PWD/my-docker-registry/data:/var/lib/registry   -v $PWD/my-docker-registry/auth:/auth   -e "REGISTRY_AUTH=htpasswd"   -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm"   -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd   -v $PWD/my-docker-registry/certs:/certs -v $PWD/my-docker-registry/config.yml:/etc/docker/registry/config.yml  -e REGISTRY_HTTP_ADDR=0.0.0.0:5000   -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt   -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key   registry:2.8.3
``` 

9. Now login to the local registry
```
docker login -u <user> -p <password> https://localhost:5000 

WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /home/akhil/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

10. Now the images can be pulled pushed from the local registry
```
akhil@akhilerm:~ $ docker push localhost:5000/busybox:latest
The push refers to repository [localhost:5000/busybox]
58f32e9504c8: Pushed 
latest: digest: sha256:ff0b2bbabd0147f23a4b4b499175a2aadf4b775285ea4cfdeb7b30fa3af4bdb8 size: 527

akhil@akhilerm:~ $ ctr image pull --user <user>:<password>  localhost:5000/busybox:latest
localhost:5000/busybox:latest                   saved
└──manifest (ff0b2bbabd01)                      complete        |++++++++++++++++++++++++++++++++++++++|
   ├──layer (13c1f3d7904f)                      complete        |++++++++++++++++++++++++++++++++++++++|
   └──config (27a71e19c956)                     complete        |++++++++++++++++++++++++++++++++++++++|
application/vnd.docker.distribution.manifest.v2+json sha256:ff0b2bbabd0147f23a4b4b499175a2aadf4b775285ea4cfdeb7b30fa3af4bdb8
Completed pull from OCI Registry (localhost:5000/busybox:latest)        elapsed: 0.1 s  total:  2.1 Mi  (24.9 MiB/s)
```

The following docker and containerd versions were used for creating the above docs
```
akhil@akhilerm:~ $ docker version
Client: Docker Engine - Community
 Version:           26.1.3
 API version:       1.45
 Go version:        go1.21.10
 Git commit:        b72abbb
 Built:             Thu May 16 08:33:29 2024
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          26.1.3
  API version:      1.45 (minimum version 1.24)
  Go version:       go1.21.10
  Git commit:       8e96db1
  Built:            Thu May 16 08:33:29 2024
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v2.0.0
  GitCommit:        207ad711eabd375a01713109a8a197d197ff6542
 runc:
  Version:          1.1.12
  GitCommit:        v1.1.12-0-g51d5e946
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```
