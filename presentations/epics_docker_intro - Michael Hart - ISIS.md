class: center, middle

# EPICS with Docker

Michael Hart

---

## Topics

- What is Docker
- Features that are appealing for EPICS
- Running EPICS inside Docker
- Issues we came across
- How we solved them

---

## What is Docker

- Toolset to package, distribute and run services
- Package ("Image") contains all dependencies
- Runs in sandbox environment isolated from host
- Own network interfaces on virtual network
- Native to Linux, but can run on Windows/OSX via VM

---

## Not a Virtual Machine!

- Doesn't launch full OS environment
- Processes run directly on the host
- Doesn't reserve resources, just uses what it needs
- Very lightweight, fast to launch

---

## Basic usage

Interface looks similar to Git in many ways.

Images can be shared via DockerHub, by pulling and pushing from local repository.
```
$ docker pull account/image_name
$ docker push account/image_name
$ docker images
$ docker run -it account/image_name [parameters ...]
```

---


## In Action

Own network interfaces:
```
$ docker run -it ubuntu:14.04
root@90382d86bf5c:/# ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:02  
          inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:50 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:8158 (8.1 KB)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

---

## In Action

Inside Docker container:
```
$ docker run -it ubuntu:14.04
root@90382d86bf5c:/# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  18180  3192 ?        Ss   05:49   0:00 /bin/bash
root        17  0.0  0.0  15580  2064 ?        R+   05:55   0:00 ps aux
root@90382d86bf5c:/# sleep 10
```

On Host:
```
$ ps aux | grep sleep
root     19275  0.0  0.0   4360   644 pts/21   S+   06:56   0:00 sleep 10
```

---

## Building an Image

"Dockerfile" specifies instructions to build image, similar to a Makefile.

Each command creates a new layer.
```
FROM ubuntu:trusty

RUN apt-get update && apt-get install [...]

RUN git clone https://github.com/EuropeanSpallationSource/MCAG_setupMotionDemo.git \
    && [...]

COPY ./startup.sh /

CMD ["/startup.sh"]
``` 

---

## Images and Layers

An image contains everything needed to run. Images consist of several layers.

![image layers](https://docs.docker.com/engine/userguide/storagedriver/images/image-layers.jpg)

---

## Extending existing images

FROM command references layers of existing image (often an OS base image).

![extend](https://docs.docker.com/engine/userguide/storagedriver/images/saving-space.jpg)

---

## Containers

A container is a layer created when an image is run.

<img src="https://docs.docker.com/engine/userguide/storagedriver/images/container-layers-cas.jpg" width="600" />

---

## Running multiple instances

Very cheap, since image is reused.

![many_containers](https://docs.docker.com/engine/userguide/storagedriver/images/sharing-layers.jpg)

---

## Building an EPICS base image

- Other images can reference using FROM command
- Prevents replication of work when updates required
- Only needs to be (re)built once
- Faster build times for final images

---

## Using Alpine as base OS

- Much smaller than Ubuntu or other base images
- Uses musl instead of glibc
- Has some small quirks for building EPICS as a result
- Requires build argument to define constant

---

## Docker limitations and workarounds

- Cannot define environment variables dynamically
- Profile and rc files don't get sourced automatically
- If you launch more than one process, signals don't get forwarded correctly
- Solved using an init script and project called tini

---

## Example Usage

Dockerfile:
```
FROM dmscid/epics-base:latest

RUN apk --no-cache add python

COPY 10-setup-environment.sh /etc/profile.d/10-setup-environment.sh

ENTRYPOINT ["/init.sh", "python"]
```

---

## EPICS Gateway

- Based on EPICS base image
- Open gateway that forwards all requests
- Created mainly to forward traffic between host and docker network on Windows/OSX
- Could be useful in other scenarios?

---

## When is Docker useful?

- Working on many projects with different or conflicting dependencies, avoid "polluting" host

- Share working setups without everyone having to install, configure, keep updated, etc

- Test networking functionality on a single machine

- Simulate networked systems, especially as part of unit tests
