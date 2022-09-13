# Teleport Setup Using Docker
Setup SSH proxy server with open-source software Teleport. Install on self-hosted server

Project Homepage: https://goteleport.com/
Documentation: https://goteleport.com/docs/

## Prerequisites

- Linux Server running Ubuntu 20.04 LTS or newer
- Self Hosted IP of your Linux Server
- Installed Docker and Docker-compose

## Set up Teleport

Create a new file `docker-compose.yml`file, please refer to the teleport documentation: https://goteleport.com/docs/getting-started/docker-compose/.

First, will need to install an authentication and proxy server. This will handle the whole authentication process for all nodes and clients. After the installation, we can connect to the proxy via a web client. 

I will use docker-compose to deploy the teleport proxy and auth server, there is another way which install directly on Linux. However, I prefer the containerized deployment because better flexibility.

**Example Docker-Compose File**:
```yml
version: '3'
services:
  configure:
    image: public.ecr.aws/gravitational/teleport:10.2.1
    container_name: teleport-configure
    entrypoint: /bin/sh
    hostname: <domain or ip>
    command: -c "if [ ! -f /etc/teleport/teleport.yaml ]; then teleport configure > /etc/teleport/teleport.yaml; fi"
    volumes:
      - ./config:/etc/teleport

  teleport:
    image: public.ecr.aws/gravitational/teleport:10.2.1
    container_name: teleport
    entrypoint: /bin/sh
    hostname: <domain or ip>
    command: -c "sleep 1 && /bin/dumb-init teleport start -c /etc/teleport/teleport.yaml"
    ports:
      - "3023:3023"
      - "3024:3024"
      - "3025:3025"
      - "3080:3080"
    volumes:
      - ./config:/etc/teleport
      - ./data:/var/lib/teleport
    depends_on:
      - configure
```
Before you start the container you should change the `hostname` of both containers and ensure you it accessible. Then start the docker container with the following command.
### Start the Teleport Server

```bash
docker-compose up -d
```
You may double check the docker-compose is up using below command.

```bash
docker-compose ps
```
After it create the container successfully, you will notice that it will automatically created the folder : data and config


### Adjust the Config file

Let's tak a look on the configuration file, which is located in `./config/teleport.yaml`

**Example teleport.yaml**:
```yml
version: v2
teleport:
  nodename: <auto generated>
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO
    format:
      output: text
  ca_pin: <auto generated>
  diag_addr: ""
auth_service:
  enabled: "yes"
  listen_addr: 0.0.0.0:3025
  proxy_listener_mode: multiplex
ssh_service:
  enabled: "yes"
  commands:
  - name: hostname
    command: [hostname]
    period: 1m0s
proxy_service:
  enabled: "yes"
  https_keypairs: []
  acme: {}
```

Basically the configuration files is much standard and do not need to do anything.

Now let's go back to the docker-compose file, we will need to some adjustment to remove the configuration and recreate it.

**Example Docker-Compose File**:
```yml
version: '3'
services:
  teleport:
    image: public.ecr.aws/gravitational/teleport:10.2.1
    container_name: teleport
    entrypoint: /bin/sh
    hostname: <<domain name or ip>>
    command: -c "sleep 1 && /bin/dumb-init teleport start -c /etc/teleport/teleport.yaml"
    ports:
      - "3023:3023"
      - "3024:3024"
      - "3025:3025"
      - "3080:3080"
    volumes:
      - ./config:/etc/teleport
      - ./data:/var/lib/teleport
```

After that restart your docker container with the following command.

```bash
docker-compose up -d --force-recreate
```
You can using below command to check the teleport status. 

```bash
docker-compose exec teleport tctl status
```
You may also try to hit the web ui by using url `https://<domain or ip>:3080`

## How to access to teleport

Now, we will create a user on the teleport auth server.

```bash
docker-compose exec teleport tctl users add testuser --roles=editor,access --logins=root
```

With this command, I will add a new user called `testuser` who can log in with the Linux users `root` on the nodes.

This will create a registration token. With the registration token, we can now set up our credentials on the teleport server. Teleport enforces 2FA by default. Install a 2FA like Google Authentication or Authy on your smartphone and scan the QR-Code. Then you can simply enter the 2FA code that is generated on your smartphone.

Now you can simply connect to the docker node with the web interface by accessing `https://<domain or ip>:3080`



