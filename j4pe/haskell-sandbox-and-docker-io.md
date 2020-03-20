# Haskell sandbox and Docker.io

# 2013: Create Dockerfile and build docker image

```bash
mkdir haskell-sb
cd haskell-sb
vim Dockerfile
```

or 

```bash
git clone https://github.com/j4/dockerfiles.git
```

## Create Dockerfile 

Edit the file Dockerfile : *vim Dockerfile*

```bash
from         j4pe/saucy
maintainer   j4pe

RUN dpkg-divert --local --rename --add /sbin/initctl
RUN ln -s /bin/true /sbin/initctl

RUN echo "deb http://archive.ubuntu.com/ubuntu saucy main universe" > /etc/apt/sources.list

RUN apt-get update
RUN apt-get upgrade -y 
RUN export DEBIAN_FRONTEND=noninteractive; apt-get install haskell-platform vim -y
RUN useradd ja -d /home/ja -s /bin/bash
```

## Build docker image

Build haskell sandbox

```bash
cd haskell-sb
docker build -t j4pe/haskell-sb .
```

```bash
docker images
```

Run haskell sandbox, create *haskell* file (u+rx) in *$HOME/bin/haskell*

```bash
#!/bin/bash

docker run -u ja -h haskell -i -v=/home/ja:/home/ja -e=HOME=/home/ja -t j4pe/haskell-platform bash
```

and run sandbox !

# 2018: Haskell, Nix and Docker !


[The example repository (GitHub)](https://github.com/apeyroux/sample-haskell-nix-docker)


```bash
├── ChangeLog.md
├── default.nix
├── docker.nix
├── LICENSE
├── Main.hs
├── sample-haskell-nix-docker.cabal
└── Setup.hs
```

## default.nix

```nix
(import <nixpkgs> {}).haskellPackages.developPackage { root = ./.; }
```

## docker.nix

```nix
with import <nixpkgs> {};

let
  helloHaskell = import ./default.nix;
in {
  helloHaskellAppImage = dockerTools.buildImage {
    name = "hello-haskell";
    contents = [
      helloHaskell
    ];
    config = {
      Version = "1.0";
      EntryPoint = ["sample-haskell-nix-docker"];
    };
  };
}
```

## Use Docker and Nix

```bash
nix-build docker.nix -o hello-haskell.docker
cat hello-haskell.docker | docker load
docker run -it --rm hello-haskell:latest
Hello, Haskell!
```