# Technical Documentation

### Build Logic

#### Operating System

Size was a major consideration in the base Docker image we used for our containers. Because of that, we use Alpine Linux, in most of our Docker images, which comes in at a whopping 5 MB by default. Based on comparisons between other base images, we've roughly saved 50-70% of space utilizing Alpine. The average size of each of our containers is about 269 MB due to dependencies.

#### Startup

Docker containers generally have entrypoint scripts to prepare templates, configure files, run services, or anything else you need to run to properly initialize a container and run the process. For our containers, we pull most of our files and certificates from Consul.

Because there is a heirarchy of function to Gluu Server, `wait-for-it` scripts were designed, thanks to contributions from Torstein Krause Johansen (@skybert), to try and make sure the containers don't begin their launch processes until the services superior to the container are fully started. However, there is a time limit, so a container dependent upon another container could fail as the `wait-for-it` "health checks" aren't being met.

### Networking Considerations

The mandatory ports need to be published to the host server are port 80 and 443 (both are NGINX ports). By publishing these ports to the host server, Gluu Server can be accessed publicly.

oxTrust is an OpenID Connect client, so its container is dependent upon oxAuth's `/.well-known/openid-configuration` endpoint, which is only accessible if NGINX is started. So if the oxTrust container cannot navigate to `https://<hostname>/.well-known/openid-configuration`, it will fail to finish initialization. The container will most likely not exit.

### Images/Containers

The Gluu Server Docker containers consist of in-house and 3rd-party containers.

The following images are used heavily by the Gluu Server Docker containers:

- Consul (3rd-party)
- Registrator (3rd-party)
- config-init
- OpenDJ (customized)
- OpenLDAP (customized; an alternative of OpenDJ)
- oxAuth
- oxTrust
- oxPassport
- oxShibboleth
- NGINX (customized)

Refer to the sections below for an overview of each image.

#### Consul

The Gluu Server Docker containers were built to be centralized around a configuration KV store. For our use case, we've used [Consul](https://www.consul.io/) as our KV store. The reasoning behind this decision was to allow the containers to be as modular as possible. Services can be replaced without concern for losing any of the default configuration. This isn't to say that there won't need to be persistence using volumes (see [here](https://github.com/GluuFederation/gluu-docker/blob/master/examples/single-host/docker-compose.yml#L73) and [here](https://github.com/GluuFederation/gluu-docker/blob/master/examples/single-host/docker-compose.yml#L57)) for custom files and long standing data requirements.

That being said, Consul stores all of its configuration on the disk. Using our `config-init` container, you can generate, dump, or load the cluster-wide configuration into/from Consul. Please see [the documentation](#variable-explanation) below on how to achieve this.

#### Registrator

Due to the design of Docker networking where container IP gets recycled dynamically, [Registrator](http://gliderlabs.github.io/registrator/latest/) container is used for registering and deregistering oxAuth, oxTrust, oxShibboleth, and oxPassport container's IP. With the help of Registrator, the NGINX container will route the traffic to available oxAuth/oxTrust/oxShibboleth/oxPassport backend.

#### config-init

[config-init](https://github.com/GluuFederation/docker-config-init/tree/3.1.3) is a special container that is not daemonized nor executing a long-running process. The purpose of this container is to generate the initial configuration, dump the existing configuration (for backup), or even load (restore) the configuration.

The following commands are supported by the container:

- [generate](https://github.com/GluuFederation/docker-config-init/tree/3.1.3#generate-command): The generate command will generate all the initial configuration files for the Gluu Server components. The following are required to launch unless otherwise marked:

    - `--email`: The email address of the administrator usually. Used for certificate creation.
    - `--domain`: The domain name where the Gluu Server resides. Used for certificate creation.
    - `--country-code`: The country where the organization is located. User for certificate creation.
    - `--state`: The state where the organization is located. Used for certificate creation.
    - `--city`: The city where the organization is located. Used for certificate creation.
    - `--org-name`: The organization using the Gluu Server. Used for certificate creation.
    - `--kv-host`: The IP address or hostname of the KV store (Consul). Default is `localhost`.
    - (optional) `--kv-port`: The port used to access consul. Default is `8500`.
    - `--admin-pw`: The administrator password for oxTrust and LDAP
    - `--ldap-type`: Either OpenDJ or OpenLDAP. If you're looking to use LDAP replication, we recommend OpenDJ.

- [dump](https://github.com/GluuFederation/docker-config-init/tree/3.1.3#dump-command): The dump command will dump all configuration from inside Consul's KV store into the `/opt/config-init/db/config.json` file inside the container. The following is required to launch unless otherwise marked:

    - `--kv-host`: The IP address or hostname of the KV store (Consul). Default is `localhost`.
    - (optional) `--kv-port`: The port used to access Consul. Default is `8500`.

    Please note that to dump this file into the host, you'll need to map a mounted volume to the `/opt/config-init/db` directory. See this example on how to dump the config into the `/path/to/host/volume/config.json` file:

        docker run \
            --rm \
            --network container:consul \
            -v /path/to/host/volume:/opt/config-init/db \
            gluufederation/config-init:$GLUU_VERSION \
            dump \
            --kv-host <consul address>

- [load](https://github.com/GluuFederation/docker-config-init/tree/3.1.3#load-command): The load command will load a `config.json` into the Consul KV store. The following are required to launch unless otherwise marked:

    - `--kv-host`: The IP address or hostname of the KV store (Consul). Default is `localhost`.
    - (optional) `--kv-port`: The port used to access Consul. Default is `8500`.

    Please note that to load this file from the host, you'll need to map a mounted volume to the `/opt/config-init/db` directory. For example on how to load the config from `/path/to/host/volume/config.json` file:

        docker run \
            --rm \
            --network container:consul \
            -v /path/to/host/volume:/opt/config-init/db \
            gluufederation/config-init:$GLUU_VERSION \
            load \
            --kv-host <consul address>

#### OpenDJ

The following variables are used by the container:

- `GLUU_KV_HOST`: Consul's hostname or IP address
- `GLUU_KV_PORT`: Consul's port number
- `GLUU_LDAP_INIT`: whether to import initial LDAP entries (possible values are true or false)
- `GLUU_LDAP_INIT_HOST`: LDAP hostname for initial configuration (only usable when `GLUU_LDAP_INIT` set to true)
- `GLUU_LDAP_INIT_PORT`: LDAP port for initial configuration (only usable when `GLUU_LDAP_INIT` set to true)
- `GLUU_CACHE_TYPE`: supported values are `IN_MEMORY` and `REDIS`, default is `IN_MEMORY`
- `GLUU_REDIS_URL`: URL of redis service, format is host:port (optional)
- `GLUU_REDIS_TYPE`: redis service type, either `STANDALONE` or `CLUSTER` (optional)
- `GLUU_LDAP_ADDR_INTERFACE`: interface name where the IP will be guessed and registered as OpenDJ host, e.g. `eth0` (will be ignored if `GLUU_LDAP_ADVERTISE_ADDR` is used)
- `GLUU_LDAP_ADVERTISE_ADDR`: the hostname/IP address used as the host of OpenDJ server

#### OpenLDAP

An alternative of OpenDJ image. To use this container, make sure to choose `openldap` when running `config-init` container.
For example:

    docker run \
        --rm \
        gluufederation/config-init:3.1.3_dev generate --ldap-type=openldap

The following variables are used by the container:

- `GLUU_KV_HOST`: Consul's hostname or IP address
- `GLUU_KV_PORT`: Consul's port number
- `GLUU_LDAP_INIT`: whether to import initial LDAP entries (possible values are true or false)
- `GLUU_LDAP_INIT_HOST`: LDAP hostname for initial configuration (only usable when `GLUU_LDAP_INIT` set to true)
- `GLUU_LDAP_INIT_PORT`: LDAP port number for initial configuration (only usable when `GLUU_LDAP_INIT` set to true)
- `GLUU_CACHE_TYPE`: supported values are `IN_MEMORY` and `REDIS`, default is `IN_MEMORY`
- `GLUU_REDIS_URL`: URL of redis service, format is `host:port` (optional)
- `GLUU_REDIS_TYPE`: redis service type, either `STANDALONE` or `CLUSTER` (optional)
- `GLUU_LDAP_ADDR_INTERFACE`: interface name where the IP will be guessed and registered as OpenLDAP host, e.g. `eth0`


#### oxAuth

Variables used by the container:

- `GLUU_LDAP_URL`: The LDAP database's IP address or hostname. Default is `localhost:1636`. Multiple URLs can be used using comma-separated values (i.e. `192.168.100.1:1636,192.168.100.2:1636`).
- `GLUU_KV_HOST`: The IP address or hostname of the KV store (Consul). Default is `localhost`.
- `GLUU_KV_PORT`: The port used to access Consul. Default is `8500`.
- `GLUU_MAX_RAM_FRACTION`: Used in conjunction with Docker memory limitations (`docker run -m <mem>`) to identify the fraction of the maximum amount of heap memory you want the JVM to use.
- `GLUU_DEBUG_PORT`: port of remote debugging (if omitted, remote debugging will be disabled).

#### oxTrust

The following variables are used by the container:

- `GLUU_LDAP_URL`: The LDAP database's IP address or hostname. Default is `localhost:1636`. Multiple URLs can be used using comma-separated values (i.e. `192.168.100.1:1636,192.168.100.2:1636`).
- `GLUU_KV_HOST`: The IP address or hostname of the KV store (Consul). Default is `localhost`.
- `GLUU_KV_PORT`: The port used to access Consul. Default is `8500`.
- `GLUU_MAX_RAM_FRACTION`: Used in conjunction with Docker memory limitations (`docker run -m <mem>`) to identify the fraction of the maximum amount of heap memory you want the JVM to use.
- `GLUU_OXAUTH_BACKEND`: the oxAuth backend address, default is `localhost:8081` (used in `wait-for-it` script)
- `GLUU_SHIB_SOURCE_DIR`: absolute path to directory to copy Shibboleth config from (default is `/opt/shibboleth-idp`)
- `GLUU_SHIB_TARGET_DIR`: absolute path to directory to copy Shibboleth config to (default is `/opt/shared-shibboleth-idp`)

#### oxPassport

To add new strategies to oxPassport, refer to the official docs [here](https://gluu.org/docs/ce/authn-guide/passport/#setup-passportjs-with-gluu). A manual restart of the oxPassport container is needed.

The following variables are used by the container:

- `GLUU_KV_HOST`: The IP address or hostname of the KV store (Consul). Default is `localhost`.
- `GLUU_KV_PORT`: The port used to access Consul. Default is `8500`.
- `GLUU_LDAP_URL`: The IP address or hostname of the LDAP database. Default is `localhost:1636`. Multiple URLs can be used using comma-separated values (i.e. `192.168.100.1:1636,192.168.100.2:1636`).
- `GLUU_OXAUTH_BACKEND`: the address of oxAuth backend, default is `localhost:8081` (used in `wait-for-it` script)
- `GLUU_OXTRUST_BACKEND`: the address of oxTrust backend, default is `localhost:8082` (used in `wait-for-it` script)

#### oxShibboleth

Mounting the volume from host to container, as seen in the `-v $PWD/shared-shibboleth-idp:/opt/shared-shibboleth-idp` option, is required to ensure oxShibboleth can load the configuration correctly. This can [also be seen here](https://github.com/GluuFederation/gluu-docker/blob/master/examples/single-host/docker-compose.yml#L114) in the standalone docker-compose yaml file or [here](https://github.com/GluuFederation/gluu-docker/blob/master/examples/multi-hosts/web.yml#L88) in the multi-host docker-compose yaml file.

By design, each time a Trust Relationship entry is added/updated/deleted via the oxTrust GUI, some Shibboleth-related files will be generated/modified by oxTrust and saved to the `/opt/shibboleth-idp` directory inside the oxTrust container. A background job in oxTrust container ensures those files are copied to the `/opt/shared-shibboleth-idp` directory (and also inside the oxTrust container, which must be mounted from container to host).

After those Shibboleth-related files are copied to `/opt/shared-shibboleth`, a background job in oxShibboleth copies them to the `/opt/shibboleth-idp` directory inside oxShibboleth container. To ensure files are synchronized between oxTrust and oxShibboleth, both containers must use the same mounted volume, `/opt/shared-shibboleth-idp`.

The `/opt/shibboleth-idp` directory is not mounted directly into the container, as there are two known issues with this approach. First, the oxShibboleth container has its own default `/opt/shibboleth-idp` directory requirements to start the app itself. By mounting `/opt/shibboleth-idp` directly from the host, the directory will be replaced and the oxShibboleth app won't run correctly. Secondly, oxTrust renames the metadata file, which unfortunately didn't work as expected in the mounted volume.

The following variables are used by the container:

- `GLUU_KV_HOST`: The IP address or hostname of the KV store (Consul). Default is `localhost`.
- `GLUU_KV_PORT`: The port used to access Consul. Default is 8500.
- `GLUU_MAX_RAM_FRACTION`: Used in conjunction with Docker memory limitations (`docker run -m <mem>`) to identify the fraction of the maximum amount of heap memory you want the JVM to use.
- `GLUU_LDAP_URL`: The LDAP database's IP address or hostname. Default is `localhost:1636`. Multiple URLs can be used using comma-separated values (i.e. `192.168.100.1:1636,192.168.100.2:1636`).
- `GLUU_SHIB_SOURCE_DIR`: absolute path to directory to copy Shibboleth config from (default is `/opt/shared-shibboleth-idp`)
- `GLUU_SHIB_TARGET_DIR`: absolute path to directory to copy Shibboleth config to (default is `/opt/shibboleth-idp`)

#### NGINX

We built a customized NGINX image, based on the official open source version and containing a [consul-template](https://github.com/hashicorp/consul-template) and an NGINX server itself, to satisfy our requirements:

- Dynamically updating the `upstream` directive to point to available oxAuth/oxTrust/oxShibboleth/oxPassport containers
- Restarting the NGINX process when its configuration is changed (without restarting the container)

The following variables are used by the container:

- `GLUU_KV_HOST`: The IP address or hostname of the KV store (Consul). Default is `localhost`.
- `GLUU_KV_PORT`: The port used to access Consul. Default is `8500`.