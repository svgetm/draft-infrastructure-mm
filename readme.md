# Server prepare
- To force install from `deb` packages: comment [cdrom] in `/etc/apt/sources.list`
- Root priveleges needed for most of operations (use `sudo` prefix, or switch to `su`)

## Docker
Requirements:
- 64-bit system; (ok)
- 3.10+ kernel; (ok)
- 1.4+ `iptables`; (nftables section)
- 1.7+ `git`; (git section)
- `ps` executable; (ok)
- 4.9+ `XZ utils`; (ok)
- configured `cgroupfs-mount` (cgroupfs section)

Download `docker-ce` and unpack it to `/usr/bin` directory
```
# tar xzvf docker.tgz
# cp docker/* /usr/bin/
```

#### configure iptables:
> `# apt install nftables` (offline: packages.debian.org: `libnftables0`, `nftables`)

> `# systemctl enable nftables.service` (to stop the firewall `nft flush ruleset`, to prevent starting on boot `systemctl mask nftables.service`, to uninstall `apt purge`)

set `/sbin/iptables` to use the newest `nft` package
```
# update-alternatives --set iptables /usr/sbin/iptables-nft
# update-alternatives --set ip6tables /usr/sbin/ip6tables-nft
# update-alternatives --set arpables /usr/sbin/arptables-nft
# update-alternatives --set ebtables /usr/sbin/ebtables-nft
```

create a link to `/sbin/iptables` from `/usr/bin/iptables` to be used by `docker` daemon
> `# ln /sbin/iptables /usr/bin/iptables`

### install git
> `# apt install git` (offline: packages.debian.org: `git-man`, `liberror-perl`)

### configure cgroupfs
> `# apt install cgroupfs-mount` (offline: packages.debian.org: `cgroupfs-mount`)

### add apparmor support
create a link to `/sbin/apparmor_parser` from `/usr/bin/apparmor_parser` to be used by `docker` daemon
> `# ln /sbin/apparmor_parser /usr/bin/apparmor_parser`

### Launch docker daemon
> `# dockerd &`

`ps` should show `dockerd` running

## Docker-compose
Download `docker-compose` from `https://github.com/docker/compose/releases/download/1.25.0/docker-compose-Linux-x86_64` (version 1.25.0 is the newest for the time this manual is written)
```
# cp docker-compose-Linux-86_64 /usr/local/bin/docker-compose
# chmod +x /usr/local/bin/docker-compose
# ln /usr/local/bin/docker-compose /usr/bin/docker-compose
```
To check the application execute `docker-compose --version`

## Nexus
Nexus data should be available explicitly, so it should be inside the host dir. Nexus sv-noda port is 8010, so that it should be connected to 8081 port of the container. Current build's admin password is `admin1234`

- pull the image of `sonatype/nexus` (offline: On the online computer pull the image of `sonatype/nexus` and execute `# docker save --output _file_name_.tar image_repository`, copy the tarball to an offline server and execute `# docker load < _file_name_.tar`)

```
# mkdir /some/dir/nexus-data && chown -R 200 /some/dir/nexus-data
# mkdir /some/dir/sonatype-work && chown -R 200 /some/dir/sonatype-work
```

and use docker-compose.yml
```
nexus:
    volumes:
        - /home/c68srv/docker/data/nexus3/nexus-data:/nexus-data
    ports:
        - "8011:8081"
    restart: always
    image: sonatype/nexus3
    container_name: nexus3
```

> `# docker-compose -f compose-nexus.yml up -d` creates a container and starts it as a daemon

To restart service use `# docker-compose -f compose-nexus.yml stop` and `start` respectively

## Nexus2 (for maven repositories)
`Nexus2` is used to host maven packages for java developers in a raw filesystem way. `Nexus3` is hosting repositories in BLOBs, which makes it hard to use for developer needs (for example, adding a batch of packages, downloaded from internet). Current deployed `Nexus2` version is `2.14.15-01`. `Nexus2` is deployed the same way as `Nexus3`, with one difference: chown must contain group policies as well as user, so thtat the command will look like `# chown -R 777 /path/to/sonatype-work`. before pasting storage contents `cuba` repositories must be created explicitly, referring to `cuba platform` manual. the 777 rights are needed to share stored repositories by http for anonymous users. Currently `sonatype-work` directory is set to relative path

## Gitlab
gitlab should start at `3033` port and the working host should be set in the container's config. if we set `external_url` explicitly adding port to the address - this port should be given in the `ports` section. volumes attached as shown in official example file.

`docker-compose.yml` file
```
gitlab:
  image: 'gitlab/gitlab-ce:latest'
  restart: always
  hostname: 'sv-noda.risde.ru'
  container_name: gitlab
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'http://sv-noda.risde.ru:3033'
      gitlab_rails['gitlab_shell_ssh_port'] = 2224
  ports:
    - '3033:3033'
    - '30443:443'
    - '2224:22'
  volumes:
    - '/home/c68srv/docker/data/gitlab/config:/etc/gitlab'
    - '/home/c68srv/docker/data/gitlab/logs:/var/log/gitlab'
    - '/home/c68srv/docker/data/gitlab/data:/var/opt/gitlab'
```

## Postgres
PostgreSQL RDBMS is needed in this system for mattermost chat to store it's data. current version is `10.11`, pulled from `docker hub`

`docker-compose.yml`
```
version: '3.1'

services:
 db:
  image: postgres:10.11
  restart: always
  ports:
   - 5433:5432
  environment:
   POSTGRES_USER: postgres
   POSTGRES_PASSWORD: postgres
  volumes:
   - /home/c68srv/docker/data/postgres/data:/var/lib/postgresql/data
  container_name: postgres
```

## Mattermost
First of all, backup an existing mattermost database and restore it on a new server. if there is no existing database - skip this step.
`volumes` folder contains user-uploaded files, be careful with it, please
In `docker-compose.yml` change db credentials to the current postgres location, user, password
In `volumes/app/mattermost/config/config.json` change db credentials as well.
At this point `mattermost` is deployed from existing images, located on the old seerver. online self-hosted version drops with an error.

## Tuleap
Requirements:
 - docker
 - docker-compose
 - Nexus3

Create blobstore: yum
Create proxy yum repository `(Repositories > Create repository > yum (proxy))`:
- Tuleap
name: Tuleap
remote storage: https://ci.tuleap.net/yum/tuleap/rhel/6/dev/
blob store: yum
- Centos
name: centos
remote storage: http://mirror.centos.org/centos/
blob store: yum
- Epel
name: epel
remote storage: http://mirror.linux-ia64.org/epel/6/
blob store: yum
- Remirepo
name: remirepo
remote storage: http://rpms.remirepo.net/enterprise/6/safe/
blob store: yum

Make sure to remember repository names, they are case-sensitive
git clone https://gogs.terekhov.lv/l.terekhov/tuleap_closenetwork.git
cd docker-tuleap/repos
change all repos to your current nexus address
