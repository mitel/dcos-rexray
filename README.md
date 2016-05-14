# dcos-rexray

DCOS with EMC {code} RexRay & DVDI for persistent storage

## Overview
This walkthrough is meant to be used for those who want to play with DCOS 1.7 (not the Enterprise Ed). 
Everything described here is for demo purposes and purely experimental. 

Tested on AWS with DCOS Community Edition, one Mesos Master.

DCOS 1.7 comes with Docker 1.7.1 on CoreOS. According to Mesosphere there is no support for Docker containers with external volumes for DCOS installations running Docker older than 1.8.
I have experimented with modifying the original DCOS CloudFormation template to use the latest stable CoreOS which comes with Docker 1.9.1. Just to be able to use Docker/rexray/DVDI until a new DCOS version comes out.

## Install the components

To ssh on each Mesos agent and on the master(s) I took the Mesos you may want to check this link:

https://docs.mesosphere.com/administration/sshcluster/ 

To ssh on agents:

`dcos node ssh --master-proxy --mesos-id=<mesos-id>`

To ssh on the master:

`dcos node ssh --master-proxy --leader`


#### Install rexray on Mesos agents

`curl -sSL https://dl.bintray.com/emccode/rexray/install | sh -s stable`

This walkthrough should work with rexray 0.3.3. 

Create and edit the config.yml:

`sudo vi /etc/rexray/config.yml`

```yml
rexray:
  storageDrivers:
  - ec2
  logLevel: info
  volume:
    remove:
      disable: true
    create:
      disable: false
    mount:
      preempt: true
    unmount:
      ignoreUsedCount: false
    path:
      disableCache: true
linux:
  volume:
    rootPath: /data
    fileMode: 0700
```

Check the config options here: http://rexray.readthedocs.io/en/stable/user-guide/config/#advanced-configuration

Test it. You should see the available EBS volumes:

`rexray volume ls`

Start the rexray service:

`sudo rexray start`

#### Install DVDCLI on the Mesos agents
More details about DVDCLI may be found here: https://github.com/emccode/dvdcli

DVDCLI stands for Docker Volume Driver CLI and it is exactly that: a CLI implementation of the Docker Volume Driver.

`curl -sSL https://dl.bintray.com/emccode/dvdcli/install | sh -s stable`

Test it:

`sudo dvdcli mount --volumedriver=rexray --volumename=test1`

rexray will mount the volume here: `/var/lib/rexray/volumes/test1`

Check the EC2 dashboard - you should see the volume in use by one of the Mesos Agents. 

This will unmount the volume:

`sudo dvdcli unmount --volumedriver=rexray --volumename=test1`

#### Install DVDI on the Mesos agents
DVDI stands for Docker Volume Driver Interface. To understand it better, I recommend checking this material: https://github.com/emccode/mesos-module-dvdi/blob/master/DVDI%20Isolator%20Overview%20and%20Roadmap.md 

DCOS 1.7 comes with Mesos 0.28.1 hence this command should get you the binaries:

`curl -L -O https://github.com/emccode/mesos-module-dvdi/releases/download/v0.4.2/libmesos_dvdi_isolator-0.28.1.so`

For me it didn't work. First there is a problem with calling the DVDCLI on CoreOS, for which I opened an issue (https://github.com/emccode/mesos-module-dvdi/issues/99). I also got a `invalid ELF header` when trying to use the above binaries. Then I decided to build it myself. My build is publicly available from an S3 location: 

`curl -O https://s3.amazonaws.com/dcos-mitel/libmesos_dvdi_isolator-0.28.1.so`

If you want to build it yourself, here is how I did it:

```bash
# clone the git repo:
git clone https://github.com/emccode/mesos-module-dvdi.git
cd mesos-module-dvdi

# edit and change /usr/bin to /opt/bin for the dvdcli executable
# that's because dvdcli installs by default in /opt/bin on CoreOS
vi isolator/isolator/docker_volume_driver_isolator.hpp

# build the docker image
docker build -t mitelone/mesos-module-dvdi-dev:0.28.1 - < Dockerfile-mesos-module-dvdi-dev

# mount the isolator folder and run the build inside a Docker container
cd isolator
docker run -ti --volume=$(pwd):/isolator mitelone/mesos-module-dvdi-dev:0.28.1

# you will find the build here:
cd /build/.libs
```

After downloading or building the .so file, copy it somewhere, eg: /home/core/dvdi/

Add the library to the list in mesos-slave-modules.json, following the json syntax:

`sudo vi /opt/mesosphere/etc/mesos-slave-modules.json` 

```json
{
  "libraries": [
    { ... },
    {
      "file": "/home/core/dvdi/libmesos_dvdi_isolator-0.28.1.so",
      "modules": [
        {
          "name": "com_emccode_mesos_DockerVolumeDriverIsolator",
          "parameters": [
            {
              "key": "isolator_command",
              "value": "/emc/dvdi_isolator"
            }
          ]
        }
      ]
    }
  ]
}
```

Add `com_emccode_mesos_DockerVolumeDriverIsolator` to the existing `MESOS_ISOLATION` list:

`sudo vi /opt/mesosphere/etc/mesos-slave-common`

It should look like this:

`MESOS_ISOLATION=cgroups/cpu,cgroups/mem,posix/disk,com_emccode_mesos_DockerVolumeDriverIsolator`

Now it should make some more sense to you why DVDI/rexray is so important - storage becomes a Mesos-managed resource along with CPU and Memory.

Stop the Mesos Agent:

`sudo systemctl stop dcos-mesos-slave.service`

After all these modifications, starting the service via systemctl fails for some reason I could not identify yet, hence I had to start it manually once, like this:

`sudo /opt/mesosphere/packages/mesos--0335ca0d3700ea88ad8b808f3b1b84d747ed07f0/bin/mesos-slave --modules=file:///opt/mesosphere/etc/mesos-slave-modules.json --isolation=cgroups/cpu,cgroups/mem,posix/disk,com_emccode_mesos_DockerVolumeDriverIsolator --master=zk://leader.mesos:2181/mesos --containerizers=docker,mesos --log_dir=/var/log/mesos`

It should start succesfully (check your `mesos-slave` executable location, might be different than the one above), then you can CTRL-C and then start it under systemd:

`sudo systemctl start dcos-mesos-slave.service`

Verify the service status:

`sudo systemctl status dcos-mesos-slave.service`

Now exit the Mesos agent session and ssh on the master to setup Marathon:
	
`dcos node ssh --master-proxy --leader`

Add `external_volumes` to `--enable_features` list:

`sudo vi /etc/systemd/system/dcos-marathon.service`

It should look like this now:

`--enable_features "vips,task_killing,external_volumes"`

Restart Marathon:

`sudo systemctl daemon-reload`

`sudo systemctl restart dcos-marathon.service`

#### Even more tweaking
After running the above steps you should be able to run stateful apps using the Mesos containerizer using the example here:

https://mesosphere.github.io/marathon/docs/external-volumes.html 

..but not yet the Docker containerizer due to the Docker version used on the CoreOS version used by DCOS 1.7.
Quite frustrating :) So I modified the CloudFormation template to use a newer CoreOS version that comes with Docker 1.9.1.
Check it out and/or use it from

https://s3.amazonaws.com/dcos-mitel/dcos_single_master_23.04_coreos_899.17.0.json 

You can provide this link when deploying DCOS on AWS (US-East-1 region only).

Have fun!
