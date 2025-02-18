![pulls](https://img.shields.io/docker/pulls/signoz/locust?logo=docker&style=flat-square)
![stars](https://img.shields.io/docker/stars/signoz/locust?style=flat-square)

I wanted the image to:
* use Python 3.x
* use the latest version of Locust
* be as small as possible
* be simple to use
* take Locust scripts by means of mounting a volume

Most of the images found on docker hub was old (1-2 yo) so I decided to give it a try.

This one is based on python image for alpine, installs `locustio` package and required dependencies.
It weighs about 130MB.

# Repository structure and images tagging
The git project is organized so it can maintain many versions of the image.
Tagging "system" was chosen to easily distinguish between the versions.

## Repository structure

Folder structure looks like this:

```
   .
   |-- locust version a
   |   |--- python version x
   |   |    |--  OS version 1
   |   |    `--  OS version 2
   |   `--- python version y
   |        |---  OS version 1
   |        `---  OS version 2
   |-- locust version b
   ...
```

Where:
* `locust version` is a specific version of locust (i.e. 0.10.0)
* `python version` is a specific version of python (i.e. 3.6)
* `OS version 1` is a specific version of the operating system (i.e. alpine3.9)

## Tagging structure
The tagging reflects the directory structure and contains information about all the components:
* locust version
* python version
* OS version


For example, for locust 0.10.0 that runs on Python 3.6 on Alpine 3.9 the dockerfile is placed in:
```
   \
   |--- 0.10.0
   | `-- python3.6
   |   `-- alpine3.9
```
The tag for this image will be: **signoz/locust:0.10.0-python3.6-alpine3.9**.


# Usage

published images on docker hub: https://hub.docker.com/r/signoz/locust

The image does not include locust scripts during a build. It assumes, the scripts will be supplied on runtime by mounting a volume (to `/locust` path).

This gives the ability to use the exact same image for different deployments. There is no need to build your image that
would inherit from this one and only include test scripts (although it's also possible).

## Pulling the image
Pull the latest stable version:

```
docker pull signoz/locust
```

Or choose locust, python and OS (Operating System) version you want and pull and the image that is tagged accordingly (see: [Tagging structure](#tagging-structure)):

```
docker pull signoz/locust:1.2.3-python3.9-alpine3.12
```


## Running the image
The image uses the following environment variables to configure its behavior:

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
|LOCUST_FILE   | Sets the `--locustfile` option. | locustfile.py | |
|ATTACKED_HOST | The URL to test. Required. | - | http://example.com |
|LOCUST_MODE   | Set the mode to run in. Can be `standalone`, `master` or `worker`. | standalone | master |
|LOCUST_MASTER | Locust master IP or hostname. Required for `worker` mode.| - | 127.0.0.1 |
|LOCUST_MASTER_BIND_PORT | Locust master port for communication with workers. Used in distributed mode.<br>For master: which port master should bind to.<br>For worker: port to connect on master. | 5557 | 6666 |
|LOCUST_OPTS| Additional locust CLI options. | - | "-c 10 -r 10" |


### Standalone

Basic run, with folder (path in $MY_SCRIPTS) holding `locustfile.py`:
```
docker run --rm --name standalone --hostname standalone -e ATTACKED_HOST=http://standalone:8089 -p 8089:8089 -d -v $MY_SCRIPTS:/locust signoz/locust
```
or, with additional runtime options (in this example, for running without the UI):
```
docker run --rm --name standalone --hostname standalone -e ATTACKED_HOST=http://example.com -e "LOCUST_OPTS=--no-web" -d -v $MY_SCRIPTS:/locust signoz/locust
```

### Master-worker

Run master:
```
docker run --name master --hostname master \
 -p 8089:8089 -p 5557:5557 -p 5558:5558 \
 -v $MY_SCRIPTS:/locust \
 -e ATTACKED_HOST=http://master:8089 \
 -e LOCUST_MODE=master \
 --rm -d signoz/locust
```

and some workers:

```
docker run --name worker0 \
 --link master --env NO_PROXY=master \
 -v $MY_SCRIPTS:/locust \
 -e ATTACKED_HOST=http://master:8089 \
 -e LOCUST_MODE=worker \
 -e LOCUST_MASTER=master \
 --rm -d signoz/locust

docker run --name worker1 \
 --link master --env NO_PROXY=master \
 -v $MY_SCRIPTS:/locust \
 -e ATTACKED_HOST=http://master:8089 \
 -e LOCUST_MODE=worker \
 -e LOCUST_MASTER=master \
 --rm -d signoz/locust
```


For the real brave, Windows PowerShell version:

Basic run:
```
docker run --rm --name standalone `
 -e ATTACKED_HOST=http://localhost:8089 `
 -v c:\locust-scripts:/locust `
 -p 8089:8089 -d `
 signoz/locust
```

Run master:
```
docker run --name master --hostname master `
 -p 8089:8089 -p 5557:5557 -p 5558:5558 `
 -v c:\locust-scripts:/locust `
 -e ATTACKED_HOST='http://master:8089' `
 -e LOCUST_MODE=master `
 --rm -d signoz/locust
```

Run worker:
```
docker run --name worker0 `
 --link master --env NO_PROXY=master `
 -v c:\locust-scripts:/locust `
 -e ATTACKED_HOST=http://master:8089 `
 -e LOCUST_MODE=worker `
 -e LOCUST_MASTER=master `
 --rm -d signoz/locust
```

## Examples
Other simple examples are collected in [examples](./examples) directory. They include some docker-compose files to run locust standalone or distributed.
See the folder for details.

# Building the image
Choose Locust, Python and OS (Operating System) version you want by going into desired directory (see: [Repository structure](#repository-structure))
```
docker build -t signoz/locust:1.2.3-python3.9-alpine3.12 .
```
or, if behind a proxy (and the proxies are defined in HTTP(S)_PROXY variables:
```
docker build --build-arg HTTP_PROXY=$http_proxy --build-arg HTTPS_PROXY=$https_proxy -t signoz/locust:1.2.3-python3.9-alpine3.12 .
```

There is also a simple and messy bash script -- [build-all.sh](build-all.sh) -- for development purposes. It's able to build all the images or images for selected locust version.
