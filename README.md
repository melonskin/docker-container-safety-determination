## Project specification


Our project aims to provide a cloud service to verify the security for docker development, preventing malicious code compromising the infrastructures. The safety scan contains two parts, first, we will check the docker image uploaded by the developer in the private registry. Second, we will scan the running docker containers in the production development. Thus to make sure the security of the docker development and deployment.


## Background and motivation


Docker greatly simplifies the deployment and management of application. For example, to deploy an application consisting of a set of services one pulls corresponding docker images from a registry and wires them together. 


However, there are plenty of security vulnerability across the development stacks. During the development, the developers may pull images which contains malicious code or the developers themselves maybe compromised to intentionally inject malicious code to the application. Besides, during the deployment of docker containers in the production environment. The docker image may get attacked, for example due to the security vulnerability of the production environment.  In this project, we propose a solution to address these problems.


## Design


In this project, we propose to build a prototype software to demonstrate our approach. The software mainly contains three parts: 


1. A background crawler that pull the docker images that are pushed to the private registry. 
2. A docker image scanner to determine whether the image is malicious or not. To determine if the given images are malicious or not, we intend to compare the suspicious images with the Reference Data Set (RDS) collected by National Software Reference Library (NSRL). The RDS incorporates application hash values in the hashset which may be considered malicious, i.e. steganography tools and hacking scripts.
    - A local database can be used to cache those scanned files and thus to reduce the cost of scanning.
3. A background scanner to scan the running docker containers in the production environment. We intend to implement scanning scheduling, while use 3rd party tools for container scan.


The basic software we expect to implement contains the above docker image scan and docker container scan. Further, we may focus on the performance optimization for large scale system or we may consider more security vulnerabilities for docker development and deployment and implement approach to tackle them.


## Implementation


This is a service associated with a docker registry that can inspect pushed docker containers and figure out whether they are safe.


1. ClamAV is used to detect virus files.
2. sdhash values are calculated for each file for caching purpose.
3. MongoDB is used to store sdhashes, allow faster examination of previously checked files.
4. A registry application is running as a docker container. The virus checking happens every time we push an image into this registry.
5. Once we have a newly pushed image, the program will download and untar it into a local directory then do virus checking on all files there. 
6. If an image is detected as suspicious, the program will delete it in the registry.
7. The results can be shown in browser with the help of flask server.
8. In the client side or production environment. A background service that monitoring the running containers.
9. If there's malware found in the container, it will delete the related containers as well as the docker images and also printing the log into the console.


## Installation and pre-configuration


See [prerequisites](./prerequisite.md) for details.


## Run the service


### Start helper service


1. Start MongoDB: `$ ~/mongod`
2. Update clamAV DB: `$ sudo freshclam`
3. Start clamAV: `$ sudo service clamav-daemon start`


### Specify IP and PORT


In `endpoint/constants.py`, specify following fields with respect to your server. Also change `notification:endpoint:url` field in `registry/config.yml` to be `http://REGISTRY_IP:WEB_PORT/check_image`.


- `REGISTRY_IP`
- `REGISTRY_PORT`
- `WEB_PORT`


### Launch Registry


1. `$ cd registry/`
2. `$ docker-compose up`


### Launch Endpoint


1. `$ cd endpoint`
2. `$ python endpoint.py`


## On client side


### Run background_scanner


```shell
$ cd client
$ python background_scanner.py
```

### Dokcer pull image from private registry


```shell
$ docker pull REGISTRY_IP:REGISTRY_PORT/image_name
```

### Docker build image


1. See [this link](https://docs.docker.com/engine/reference/builder/#escape)
2. Use `COPY` to copy files in context directory to the new image.
3. See the reference link for `RUN` and `CMD` command.
4. Build with following command. 


```shell
$ docker build -t your_container_name:version_tag context_dir_path
```

### Docker push image


#### Enable push to insecure registry  


Add configuration file to the server host to connect to insecure private registry.  


```shell
$ sudo vim /etc/docker/daemon.json
```


Add below to the configuration file:


```
{
"insecure-registries": ["REGISTRY_IP:REGISTRY_PORT"]
}
```


Restart docker daemon:


```shell
$ sudo service docker stop
$ dockerd &
```

Here is a [reference](https://github.com/docker/distribution/issues/1874).


#### Push to registry


```shell
$ docker tag image_name:tag REGISTRY_IP:REGISTRY_PORT/container_name
$ docker push REGISTRY_IP:REGISTRY_PORT/image_name
```


### Docker save image


```shell
$ docker save -o a.tar container_name:version_tag
```


### Docker run container


```shell
$ docker run image_name:tag
```

### Docker list all containers


```shell
$ docker ps -a
```


### Docker list all images


```shell
$ docker image ls
```


## View push results


Push results can be viewed through browser at `http://REGISTRY_IP:WEB_PORT/results`. Successful pushes are marked with OK. Failed pushes lists those suspicious files.