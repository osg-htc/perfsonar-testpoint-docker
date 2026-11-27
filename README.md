# perfSONAR 5.X Testpoint docker container

## OSG image (hub.opensciencegrid.org/osg-htc/perfsonar-testpoint)

The OSG image is the upstream systemd testpoint with a small layer on top (`Dockerfile.patched`, files in `osg/`). Deployment instructions are at https://osg-htc.org/networking/.

- **Exporter access.** `/node_exporter`, `/perfsonar_host_exporter` and `/metrics` are restricted to loopback plus the networks in `/etc/apache2/osg/exporter-allow.conf` (default: AGLT2, where the OSG/WLCG central perfSONAR monitoring runs, and CERN). The rules are in `/etc/apache2/conf-enabled/osg-exporter-acl.conf`, so they apply even where a host-side `default-ssl.conf` is mounted. To use your own list, bind-mount a file of `Require ip ...` lines over `/etc/apache2/osg/exporter-allow.conf` and reload Apache. Everything else, including pScheduler, stays open.
- **Per-instance secrets.** At container start, `osg-instance-secrets.service` gives each container its own pScheduler database password and snakeoil TLS key, replacing the ones generated when the image was built (which every pulled image shares). A recreated container gets new ones; a restarted container keeps its own.
- **No port 80.** Apache listens on 443 only.

## Upstream

Please submit problems to:
  https://github.com/perfsonar/perfsonar-testpoint-docker/issues

The docker container runs all perfSONAR 5.x Services in the "Testpoint" bundle, as described at:
http://docs.perfsonar.net/install_options.html

This can be used to run perfSONAR 5.x Testpoint services on any OS that supports docker.

## Running the Docker Container

### Systemd-based Version (Recommended)

We recommend using the systemd-based version of the Docker container due to its better stability. However, as this version requires the host to support [cgroups v2](https://docs.kernel.org/admin-guide/cgroup-v2.html), a supervisord-based version is also provided.

To run the systemd-based version, follow these steps:

Docker version required >= 20.0.0
```bash
docker pull perfsonar/testpoint:systemd  
docker run -td --name perfsonar-testpoint --net=host --tmpfs /run --tmpfs /run/lock --tmpfs /tmp -v /sys/fs/cgroup:/sys/fs/cgroup:rw -v ./compose/psconfig:/etc/perfsonar/psconfig --cgroupns host perfsonar/testpoint:systemd
```

Or, build and run it using [docker compose](https://docs.docker.com/compose/) >= 2.16.0:
```bash
docker compose -f docker-compose.systemd.yml build 
docker compose -f docker-compose.systemd.yml up -d
```

### Supervisord-based Version

If you prefer to use the supervisord-based version of the Docker container, you can follow these steps:

```bash
docker pull perfsonar/testpoint
docker run -d --name perfsonar-testpoint --net=host -v ./compose/psconfig:/etc/perfsonar/psconfig perfsonar/testpoint
```

Or, build and run it using docker compose:
```bash
docker compose -f docker-compose.yml build
docker compose -f docker-compose.yml up -d
```

## Lookup Service Registration

To register your perfSONAR testpoint, start a container shell, and edit the file
`/etc/perfsonar/lsregistrationdaemon.conf` with the location and administrator information for your site.

If this host will be part of a centrally configured mesh, also edit the file 
`/etc/perfsonar/meshconfig-agent.conf`, and update the **configuration_url**.

```bash
docker exec -it perfsonar-testpoint bash
```

After editing the configuration files, exit the container and restart it.
```bash
docker restart perfsonar-testpoint
```

If you want to persist these settings even after the container is removed, you can commit the running container.
```bash
docker commit -m "added config settings" CONTAINER_ID perfsonar/testpoint
```

## Testing

Test the perfSONAR tools from another host with pscheduler and owamp installed:
```bash
owping hostname

pscheduler task clock --source hostname --dest localhost
pscheduler task throughput --dest hostname
```

## Troubleshooting

To get a shell in the Docker container on your host, run `docker ps -a` to get your container ID, 
and then run:
```bash
docker exec -it CONTAINER_ID bash
```

## Notes:
The perfSONAR hostname/IP is assumed to be the same as the base host. To use a different
name/IP for the perfSONAR container, see: https://docs.docker.com/articles/networking/
It also assumes the base host is running NTP, and not running httpd, postgres, or anything else 
listening on the list of ports below.

## Security:
Make sure the following ports are allowed by the base host:

pScheduler: 443
owamp:861, 8760-9960 (tcp and udp)
twamp: 862, 18760-19960 (tcp and udp)
simplestream: 5890-5900
nuttcp: 5000, 5101
iperf2: 5001
iperf3: 5201
ntp: 123 (udp)

See: http://www.perfsonar.net/deploy/security-considerations/


