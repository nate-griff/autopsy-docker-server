# Autopsy multiuser "server"

This respository contains all of the necessary files to run a multiuser server for Autopsy. 

## Installation
- Start by cloning the repository to your local machine. 
- Edit the .env file to include the correct values for your environment.
- Run `docker compose up --build -d` to start the local development stack.
- To publish the Samba share as well, run `docker compose --profile samba up --build -d`.
- If you started the Samba profile, stop the stack with `docker compose --profile samba down`.

## Local development notes
- On Windows hosts, the SMB ports used by Samba (`139` and `445`) are typically already owned by the OS, so the default local development stack does **not** start the Samba container.
- The PostgreSQL host port defaults to `15432` to avoid collisions with a locally installed PostgreSQL server.
- The Autopsy-facing ZooKeeper endpoint defaults to port `9983`, matching the official Autopsy Solr 8 guidance. In this Docker setup that host port maps to the standalone ZooKeeper service running inside the stack.
- If you previously started the stack before the PostgreSQL auth fix, recreate the volumes once with `docker compose down -v` before bringing it back up.
- If you are deploying this onto a dedicated Linux server, you can keep the default Samba ports and enable the `samba` profile.

## Memory tuning
The Autopsy UI values are not persisted back into this Docker stack. To make memory changes stick across restarts, edit the `.env` file and recreate the affected services.

- `SOLR_CONTAINER_MEMORY` controls the Solr container memory ceiling. If Autopsy keeps reporting a 4 GB maximum, this is the setting to raise.
- `SOLR_JAVA_MEM` is passed directly to Solr as its JVM heap setting. Example: `-Xms2g -Xmx8g`
- `ACTIVEMQ_CONTAINER_MEMORY` controls the ActiveMQ container memory ceiling.
- `ACTIVEMQ_OPTS_MEMORY` is passed to the ActiveMQ JVM. Example: `-Xms512m -Xmx2g`
- PostgreSQL is not JVM-based, so its memory and connection tuning must be handled separately from these JVM settings.

After changing memory values in `.env`, rebuild and recreate the Java services:
```bash
docker compose up --build -d --force-recreate solr activemq
```

## LEAPP packages
LEAPP modules are not a good fit for this Linux container stack if they depend on Windows binaries or Windows-only runtime components. In practice, treat them as unsupported here unless you have a separate Windows-side LEAPP workflow.

If you need LEAPP, the safer assumption is:
- run LEAPP-capable tooling on the Windows Autopsy client side, or
- use a separate Windows host for that processing rather than trying to add it to these containers.

## Setup
Upload the Autopsy Solr config set to ZooKeeper before creating cases in Autopsy:
```bash
docker exec -it autopsy-solr /bin/sh -lc 'bin/solr zk upconfig -z zookeeper:2181 -n AutopsyConfig -d "/tmp/SOLR_${SOLR_VERSION}_AutopsyService/solr-${SOLR_VERSION}/server/solr/configsets/AutopsyConfig/conf"'
```

Verify that the config set is present:
```bash
docker exec -it autopsy-solr /bin/sh -lc 'bin/solr zk ls /configs -z zookeeper:2181'
```

Configure the Autopsy client with:
- **Solr 8 Service:** `<server-ip>:8983`
- **ZooKeeper:** `<server-ip>:9983`

## Firewall (UFW)
If you are running this on an Ubuntu host with UFW enabled, allow the exposed service ports before connecting from Autopsy clients.

Default stack ports:
```bash
sudo ufw allow 8983/tcp comment 'Autopsy Solr'
sudo ufw allow 9983/tcp comment 'Autopsy Zookeeper'
sudo ufw allow 61616/tcp comment 'Autopsy ActiveMQ'
sudo ufw allow 15432/tcp comment 'Autopsy PostgreSQL'
```

Optional Samba profile ports:
```bash
sudo ufw allow 137/udp comment 'Autopsy Samba NMBD'
sudo ufw allow 138/udp comment 'Autopsy Samba Datagram'
sudo ufw allow 139/tcp comment 'Autopsy Samba NetBIOS'
sudo ufw allow 445/tcp comment 'Autopsy Samba SMB'
```

If you change any `*_HOST_PORT` values in `.env`, update the UFW rules to match.

## Usage
Start by mapping the samba share to a network drive on you computer.
The default SMB share name is `Autopsy`, so the share path will typically look like `\\<server-ip>\Autopsy`.

Then, open Autopsy and go to `Tools (toolbar at the top) > Options > Multi-user` and configure the server settings.
- Use the Solr 8 service on port `8983` and the ZooKeeper service on port `9983`.

![example config](image.png)

Replace the IP with your server ip/dns

## Credits
Inspired by:
- [CptOfEvilMinions/Autopsy-Automation](https://github.com/CptOfEvilMinions/Autopsy-Automation/tree/main)
- [DACC4/autopsy-server](https://github.com/DACC4/autopsy-server)
- [Getting started with Autopsy multi-user cluster](https://holdmybeersecurity.com/2021/05/11/getting-started-with-autopsy-multi-user-cluster/)
