# Docker deployments of OFBiz

As part of OFBIZ-12862, OFBIZ-12757, OFBIZ-12786 and OFBIZ-12798, docker deployments are being carried on VM ofbiz-ec2-or.apache.org for the demo-stable, demo-trunk and demo-next sites.

Work under OFBIZ-12757 also created 3 experimental sites:
* exp1.ofbiz.apache.org
* exp2.ofbiz.apache.org
* exp3.ofbiz.apache.org

These sites may be disabled at any time, but the hostnames will be left configured to enable rapid experimentation with demo sites in the future.

Files in this subdirectory of the ofbiz-tools repository reflect files which should be created on the root filesystem of ofbiz-ec2-or.apache.org with the following additions and/or settings:
* /etc/cron.d/ofbizdocker
  * Owned by root with permissions 0644
* /home/ofbizdocker/pull-and-restart.sh
  * Owned by ofbizdocker user with permissions 0755
* /home/ofbizdocker/ofbiz-framework
  * Git clone of https://github.com/apache/ofbiz-framework with the experimental-docker branch checked otu.


## How do the Docker deployments work

At 02:35h UTC each day, the cronttab defined by `/etc/cron.d/ofbizdocker` will execute script `pull-and-restart.sh`.

>**Note that the plugins are only updated when a change in framework has been done.**

The `pull-and-restart.sh` script does the following:
* For each directory in /home/ofbizdocker/[demo-stable, demo-next, demo-trunk, exp*]
  * Change to the directory.
  * Run `docker compose pull` to pull the latest container images needed to support the docker compose application.
  * Run `docker compose down --volumes` to shutdown and remove any existing containers and volumes for the docker compose application.
  * Run `docker compose up -d` to start the containers for the docker compose application.

New docker images are built and pushed to the GitHub Container Repository for each commit or tag on a branch. Depending on whether the build is triggered by a commit or a tag, the build will have '-snapshot' appended to the image tag.

Any data that might have accumulated in the demo site is lost each time the refresh process is run as we also create a new database container for each site. One exception to deleting the data is that we do persist log files between refreshes, see framework/base/config/log4j2.xml for details. For access log files we use the Tomcat default format and there are rotable.

The `demo-trunk` application listens on AJP port 8009.
The `demo-stable` application listens on AJP port 18009.
The `demo-next` application listens on AJP port 28009.

If in use, the `exp1` application listens on AJP port 38009, the `exp2` application listens on AJP port 48009, and the `exp3` application listens on AJP port 58009. The Apache server on ofbiz-ec2-or.apache.org has been configured to reverse-proxy to these applications for hostnames exp1.ofbiz.apache.org, exp2.ofbiz.apache.org and exp3.ofbiz.apache.org respectively.


## Default user namespace remapping

The docker daemon on ofbiz-ec2-or.apache.org has been configured to use default user namespace remapping. This means that the UIDs of processes running within containers are mapped to a range of 'high' non-existing UIDs on the host system. Since the UIDs are non-existant, processes with those UIDs will have no priviledges on the host system.

See the `dockremap` entry in file /etc/subuid to see the range of UIDs that will be used for remapping.

## Cleaning Docker disk space

A weekly cron job, /etc/cron.weekly/docker-system-prune, runs the `docker system prune --force` command to clean up unused container images.

## Viewing logs
For each demo instance (stable, next and trunk), the logs are respectively accessible under /home/ofbizdocker/[demo-stable|demo-next|demo-trunk]/logs

## Letsencrypt certificate update
It was a time when every 3 months we needed to manually update our Letsencrypt certificate. It was automated before, it's now again, so no worries. Anyway, if necessary it's quite easy to do so. Simply connect to the demo VM and run

    sudo certbot renew

I got this message today (2020-04-17):

    Processing /etc/letsencrypt/renewal/ofbiz-ec2-or.apache.org.conf
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    Cert not yet due for renewal
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    The following certs are not due for renewal yet:
      /etc/letsencrypt/live/ofbiz-ec2-or.apache.org/fullchain.pem expires on 2020-06-08 (skipped)
    No renewals were attempted.
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

and always since: it's OK. Nothing to do, it's automated. :)
In case you get an issue, simply reboot the VM and restart the demos. That happened once: https://issues.apache.org/jira/browse/INFRA-23637
