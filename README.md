# Package CrowdStrike's Falcon Linux Sensor as a Container
This project helps build the scaffolding for customers to containerize their falcon sensor.

## Pre-Launch Checklist
* Install `docker` if not already present on the build host. In theory 
[podman](https://developers.redhat.com/blog/2019/02/21/podman-and-buildah-for-docker-users/) should be
a drop-in replacement for development on Linux hosts, but has not yet been tested.

* Your CrowdStrike Customer ID (CID) is required to ensure the container associates itself with your account upon launch. Your CID can be found at [https://falcon.crowdstrike.com/hosts/sensor-downloads](https://falcon.crowdstrike.com/hosts/sensor-downloads).

* Update entrypoint.sh with your CID, e.g.:
  ```console
  CLOUDSIM_CID="YOURCID"
  ```

  This could be replaced with a sed one-liner such as ``sed -i s/YOURCID/xyz/r entrypoint.sh``. Try not to commit your CID to your Git repo! 

## Build
Build the container using the [included Dockerfile](https://github.com/CrowdStrike/dockerfiles/blob/master/Dockerfile) through a command such as:

```shell
$ docker build --no-cache=true --build-arg \
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
VCS_REF=$(git rev-parse --short HEAD) \
-t falcon-sensor:latest .
```

## Run
`falcon-sensor` is the default command of the container.  It can be invoked
as follows:

```shell
$ docker run --rm -ti --privileged \
--net=host -v /var/log:/var/log falcon-sensor
```

This is like running falcon-sensor directly on the console.  Standard terminal
output appears. Pressing Control-C would cause `docker` client to pass
`SIGINT` along to the sensor, which would then exit.

`--net=host` is required for the containerized sensor to talk to the kernel
module over netlink and to Cloudsim over localhost. Host `pid`, `uts` and
`ipc` namespaces are passed through to sensor container for easy access to
host resources. Following additional host files and directories need to be
provided to the sensor by mounting them within the container:

```bash
/var/run/docker.sock    # for the sensor to query Docker engine
/var/log                # for logs
/etc/os-release         # or its equivalent based on the distro
```

Sensor container picks up `falconctl` configuration from its environment.
Following variables are supported.  They map to the similarly named
`SET_OPTIONS` of `falconctl`.  Typically, the environment variables
are set through a Kubernetes `configmap`.  They can also be set with
`-e` option to `docker run` on the command line.

```shell
FALCONCTL_OPT_CID
FALCONCTL_OPT_AID
FALCONCTL_OPT_APD
FALCONCTL_OPT_APH
FALCONCTL_OPT_APP
FALCONCTL_OPT_TRACE
FALCONCTL_OPT_FEATURE
FALCONCTL_OPT_MESSAGE_LOG
FALCONCTL_OPT_BILLING
FALCONCTL_OPT_ASSERT
FALCONCTL_OPT_MEMFAIL_GRACE_PERIOD
FALCONCTL_OPT_MEMFAIL_EVERY_N
```

The sensor can be run as a background service as follows:

```shell
CONTAINER_ID=$(docker run -d \
-e FALCONCTL_OPT_CID=<<your CID>> \
-e FALCONCTL_OPT_TRACE=debug \
--privileged --net=host \
-v /var/log:/var/log falcon-sensor)
```

Replace ``<<your CID>>`` with your CrowdStrike Customer ID (CID). This can be found at [https://falcon.crowdstrike.com/hosts/sensor-downloads](https://falcon.crowdstrike.com/hosts/sensor-downloads). 

### Running `falconctl`
`falconctl` can be invoked inside a running sensor container with `docker exec`:

```shell
docker exec -it $CONTAINER_ID falconctl -g --trace
```

## Post-build Actions
Push the image to a registry (like ECR) if the container needs to be accessed outside of the build host.