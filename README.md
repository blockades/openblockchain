# OpenBlockChain

### Architecture

The project is split into several services:

- **Cassandra** holds the entire data: blocks, transactions, and visualisations (analysed data).
- **Bitcoin** is used to connect to the blockchain. It's a simple Bitcoin Core node
whose role is to index all the blocks and transactions and make them consumable through a JSON RPC interface.
- **Scanner** connects to the bitcoin service through its APIs and stores all the blocks and transactions
in the Cassandra database.
- **Spark** analyses the blockchain data stored in Cassandra.
- **API** is a REST interface that allows clients to consume the data stored in Cassandra.
- **Frontend** is the web app used to explore the blockchain and visualise analysed data.

Each service contains 1 or more containers and can be scaled independently from each other.

### System requirements

Due to the computing power and disk space requirements, the project was designed to be run on several machines,
preferably in a cloud.

The host machine (from which deployment will be made) requires:

- Ubuntu Wily 15.10 – 64 bit
- Note: it is possible to run the project on other versions of Ubuntu and even other operating systems
such as Windows and OS X, but instructions for those have not been included here.

The cluster machines (where the services will be deployed) require:

- RAM: min. 4GB
- CPU: min. 4 cores
- Disk: min. 200GB

### Prerequisites

These must be installed on the host machine only.

**1. Install Docker Engine**

  Docker Engine includes the docker client. `docker` will be used to communicate with the docker daemons which
  run on every machine in the cluster.

  Update package information. Ensure that APT works with the https method, and that CA certificates are installed:

  ```
  $ sudo apt-get update
  $ sudo apt-get install apt-transport-https ca-certificates
  ```

  Add the official GPG key and APT repository:

  ```
  $ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
  $ sudo echo "deb https://apt.dockerproject.org/repo ubuntu-wily main" > /etc/apt/sources.list.d/docker.list
  ```

  Update the APT package index and install Docker:

  ```
  $ sudo apt-get update
  $ sudo apt-get install docker-engine
  ```

  Start the docker daemon and verify it's installed correctly:

  ```
  $ sudo service docker start
  $ sudo docker run hello-world
  ```

  Always up-to-date instructions are [available at docker.com](https://docs.docker.com/engine/installation/linux/ubuntulinux/).

**2. Install Docker Machine**

  `docker-machine` is used to provision machines for the cluster.

  Download the Docker Machine 0.8 binary and extract it to your PATH:

  ```
  $ sudo curl -L https://github.com/docker/machine/releases/download/v0.8.0/docker-machine-`uname -s`-`uname -m` > /usr/local/bin/docker-machine
  $ sudo chmod +x /usr/local/bin/docker-machine
  ```

  Check the installation by displaying the Machine version:

  ```
  $ docker-machine version
  ```

  Always up-to-date instructions are [available at docker.com](https://docs.docker.com/machine/install-machine/).

**3. Install Docker Compose**

  Docker Compose is used to manage the services.

  ```
  $ sudo curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
  $ sudo chmod +x /usr/local/bin/docker-compose
  ```

  Check the installation by displaying the Compose version:

  ```
  $ docker-compose --version
  ```

  Always up-to-date instructions are [available at docker.com](https://docs.docker.com/compose/install/).

**4. Install VirtualBox**

  Add the repository and the secure key:

  ```
  $ sudo echo "deb http://download.virtualbox.org/virtualbox/debian wily contrib" > /etc/apt/sources.list
  $ wget -q https://www.virtualbox.org/download/oracle_vbox.asc -O- | sudo apt-key add -
  ```

  Install VirtualBox:

  ```
  $ sudo apt-get update
  $ sudo apt-get install virtualbox-5.0
  ```

  Always up-to-date instructions are [available at virtualbox.org](https://www.virtualbox.org/wiki/Linux_Downloads).

### Cluster Setup

The cluster can be deployed locally, using VirtualBox, or in the cloud, using DigitalOcean or Scaleway.

**1. Create the discovery service**

  *a) Using VirtualBox*

  Create a machine to host a Consul discovery service:

  ```
  $ docker-machine create -d virtualbox opb-consul
  ```

  Point your docker client to the new machine:

  ```
  $ eval $(docker-machine env opb-consul)
  ```

  Create and start the service:

  ```
  $ docker run --name consul --restart=always -p 8400:8400 -p 8500:8500 \
    -p 53:53/udp -d progrium/consul -server -bootstrap -ui-dir /ui
  ```

  *b) Using DigitalOcean*

  Create a machine to host a Consul discovery service:

  ```
  $ docker-machine create -d digitalocean \
    --digitalocean-access-token=<DO-TOKEN> \
    opb-consul
  ```

  Point your docker client to the new machine:

  ```
  $ eval $(docker-machine env opb-consul)
  ```

  Create and start the service:

  ```
  $ docker run --name consul --restart=always -p 8400:8400 -p 8500:8500 \
    -p 53:53/udp -d progrium/consul -server -bootstrap-expect 1 -ui-dir /ui
  ```

  *c) Using Scaleway (recommended)*

  Create a machine to host a Consul discovery service:

  ```
  $ docker-machine create -d scaleway \
    --scaleway-commercial-type=C2S --scaleway-name=opb-consul \
    --scaleway-organization=<SCALEWAY-ACCESS-KEY> --scaleway-token=<SCALEWAY-SECRET-KEY> \
    opb-consul
  ```

  Point your docker client to the new machine:

  ```
  $ eval $(docker-machine env opb-consul)
  ```

  Create and start the service:

  ```
  $ docker run --name consul --restart=always -p 8400:8400 -p 8500:8500 \
    -p 53:53/udp -d progrium/consul -server -bootstrap-expect 1 -ui-dir /ui
  ```

**2. Launch the Docker Swarm (cluster)**

  *a) Using VirtualBox*

  You can create the cluster locally, using VirtualBox:

  ```
  $ docker-machine create -d virtualbox \
    --virtualbox-memory=8192 --virtualbox-cpu-count=4 --virtualbox-disk-size=160000 \
    --swarm --swarm-master \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb

  $ docker-machine create -d virtualbox \
    --virtualbox-memory=8192 --virtualbox-cpu-count=4 --virtualbox-disk-size=160000 \
    --swarm \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb-01

  $ docker-machine create -d virtualbox \
    --virtualbox-memory=8192 --virtualbox-cpu-count=4 --virtualbox-disk-size=160000 \
    --swarm \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb-02
  ```

  *b) Using DigitalOcean*

  Or you can provision the machines from DigitalOcean:

  ```
  $ docker-machine create -d digitalocean \
    --digitalocean-access-token=<DO-TOKEN> --digitalocean-size=16gb \
    --swarm --swarm-master \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb

  $ docker-machine create -d digitalocean \
    --digitalocean-access-token=<DO-TOKEN> --digitalocean-size=16gb \
    --swarm \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb-01

  $ docker-machine create -d digitalocean \
    --digitalocean-access-token=<DO-TOKEN> --digitalocean-size=16gb \
    --swarm \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb-02
  ```

  *c) Using Scaleway (recommended)*

  Or you can provision the machines from Scaleway, which provides affordable high-end servers that are perfect
  for this project. You'll need to [install the driver](https://github.com/scaleway/docker-machine-driver-scaleway)
  first. Then:

  ```
  $ docker-machine create -d scaleway \
    --scaleway-commercial-type=C2L --scaleway-name=opb \
    --scaleway-organization=a332ff5e-eaaa-4a0c-87b5-dd3d67ae3d91 --scaleway-token=faaae837-4f9b-42f1-9154-54fd38950785 \
    --swarm --swarm-master \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb

  $ docker-machine create -d scaleway \
    --scaleway-commercial-type=C2L --scaleway-name=opb-01 \
    --scaleway-organization=a332ff5e-eaaa-4a0c-87b5-dd3d67ae3d91 --scaleway-token=faaae837-4f9b-42f1-9154-54fd38950785 \
    --swarm \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb-01

  $ docker-machine create -d scaleway \
    --scaleway-commercial-type=C2L --scaleway-name=opb-02 \
    --scaleway-organization=a332ff5e-eaaa-4a0c-87b5-dd3d67ae3d91 --scaleway-token=faaae837-4f9b-42f1-9154-54fd38950785 \
    --swarm \
    --swarm-discovery consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-store=consul://`docker-machine ip opb-consul`:8500 \
    --engine-opt cluster-advertise=eth0:2376 \
    opb-02
  ```

  By default the Scaleway machines have a 50GB disk. However, Scaleway also attaches a 250GB
  SSD, which needs to be mounted manually:

  ```
  $ printf "opb\nopb-01\nopb-02" | \
    xargs -n 1 -I CONT_NAME docker-machine ssh CONT_NAME \
    "echo ; echo ; echo CONT_NAME ;"`
    `"echo 'Formatting...' ;"`
    `"yes | mkfs -t ext4 /dev/sda ;"`
    `"echo 'Mounting...' ;"`
    `"mkdir -p /openblockchain ;"`
    `"mount /dev/sda /openblockchain ;"`
    `"echo 'Adding to fstab...' ;"`
    `"echo '/dev/sda /openblockchain auto defaults,nobootwait,errors=remount-ro 0 2' >> /etc/fstab ;"`
    `"echo 'Done' ;"
  ```

**3. Verify the swarm**

  Point your Docker environment to the machine running the swarm master:

  ```
  $ eval $(docker-machine env --swarm opb)
  ```

  Print cluster info:

  ```
  $ docker info
  ```

  You can also list the IPs that Consul has "discovered":

  ```
  $ docker run --rm swarm list consul://`docker-machine ip opb-consul`:8500
  ```

### Launch the services

**1. Create the default network**

  Point your Docker environment to the machine running the swarm master:

  ```
  $ eval $(docker-machine env --swarm opb)
  ```

  Create an overlay network that allows containers to "see" each other no matter what node they are on:

  ```
  $ docker network create --driver overlay --subnet 10.0.9.0/24 opbnet
  ```

  List all networks, `opbnet` should be shown as `overlay`:

  ```
  $ docker network ls
  ```

**2. Up!**

  From the root folder (which contains `docker-compose.yml`), run:

  ```
  $ docker-compose up -d --build
  ```

  Docker will build images and deploy all the services. By default, it'll launch 3 instances of Cassandra,
  a Bitcoin node, 1 Spark master, 2 Spark workers, a scanner and one Spark job submitter.

**3. Check the services**

  Show all containers and their status (they should all be "Up"):

  ```
  $ docker-compose ps
  ```

  Show the Cassandra cluster status (you should see 3 nodes):

  ```
  $ docker-compose exec cassandra-seed nodetool status
  ```

  In the mean time, the bitcoin server should've already scanned a few hundred blocks. Let's see it working live:

  ```
  $ docker-compose logs -f --tail 100 bitcoin
  ```

  You should see something like this, which means that it downloaded all the blocks from 0 to 187591 (in this example):

  ```
  bitcoin_1    | 2016-08-25 15:03:16 UpdateTip: new best=00000000000006edbd5920f77c4aeced0b8d84e25bb5123547bc6125bffcc830  height=187589  log2_work=68.352088  tx=4696226  date=2012-07-05 03:45:13 progress=0.015804  cache=48.7MiB(122719tx)
  bitcoin_1    | 2016-08-25 15:03:16 UpdateTip: new best=00000000000003f07c1227f986f4687d291af311a346f66247c504b332510931  height=187590  log2_work=68.352117  tx=4696509  date=2012-07-05 03:52:33 progress=0.015805  cache=48.7MiB(122815tx)
  bitcoin_1    | 2016-08-25 15:03:16 UpdateTip: new best=00000000000002b34a755d25a5fee4c0ad0c018cf94f4b0afef6aabe823d304a  height=187591  log2_work=68.352146  tx=4696593  date=2012-07-05 04:01:11 progress=0.015805  cache=48.7MiB(122839tx)
  ```

  Now let's see the Spark dashboard. Run the following command to get the IPs of machines:

  ```
  $ docker-machine ip opb
  $ docker-machine ip opb-01
  $ docker-machine ip opb-02
  ```

  And you'll be able to access the dashboard of the Spark master and workers at:

  ```
  <opb-ip>:8080     # master
  <opb-01-ip>:8081  # worker 1
  <opb-02-ip>:8081  # worker 2
  ```

  # TODO

  ---

  Let's scale the scanner to use 5 containers:

  ```
  $ docker-compose scale scanner=5
  $ vim docker-compose.yml # replace --scale=1 with --scale=5
  $ docker-compose up -d
  ```

  Docker will create 5 scanner containers. Having `--scale=5`, each container will start scanning
  a fifth of the blockchain already downloaded. For example, if by the time you rescale the scanner the bitcoin
  server downloads 1000 blocks, each scanner instance will start indexing 200 blocks. Which 200 blocks of those 1000
  is chosen at random (sequencially). After they finish scanning, each scanner will pause for 1 hour and restart.

  There's a small chance that the scanners will only scan the lower part of the blockchain, which doesn't contain
  any OP_RETURN transactions. If this happens, please restart the scanners:

  ```
  $ docker-compose restart scanner
  ```

  To see what each container is doing, watch the logs:

  ```
  $ docker-compose logs -f --tail 100 scanner  # all containers
  $ docker logs -f --tail 100 opb_scanner_3    # only container 3
  ```

  ---

  Now let's see what the bitcoin server has been doing:

  ```
  $ docker-compose logs -f --tail 100 bitcoin
  ```

  You should see something like this, which means that it downloaded all the blocks from 0 to 187591 (in this example):

  ```
  bitcoin_1    | 2016-08-25 15:03:16 UpdateTip: new best=00000000000006edbd5920f77c4aeced0b8d84e25bb5123547bc6125bffcc830  height=187589  log2_work=68.352088  tx=4696226  date=2012-07-05 03:45:13 progress=0.015804  cache=48.7MiB(122719tx)
  bitcoin_1    | 2016-08-25 15:03:16 UpdateTip: new best=00000000000003f07c1227f986f4687d291af311a346f66247c504b332510931  height=187590  log2_work=68.352117  tx=4696509  date=2012-07-05 03:52:33 progress=0.015805  cache=48.7MiB(122815tx)
  bitcoin_1    | 2016-08-25 15:03:16 UpdateTip: new best=00000000000002b34a755d25a5fee4c0ad0c018cf94f4b0afef6aabe823d304a  height=187591  log2_work=68.352146  tx=4696593  date=2012-07-05 04:01:11 progress=0.015805  cache=48.7MiB(122839tx)
  ```

  Let the bitcoin server run for a few hours until it reaches block `330,000`. After that, restart the scanner
  and make sure it scans blocks above `300,000`. Doing so will ensure that Cassandra has OP_RETURN transactions
  which Spark can analyse.

2. Spark

  Let's move on to Spark:

  ```
  $ cd spark/
  ```

  Launch the master and 2 workers:

  ```
  $ docker-compose up -d
  ```

  Start the

3. API

  ```
  $ cd api/
  $ docker-compose up -d
  ```

4. Frontend

  ```
  $ cd frontend/
  $ docker-compose up -d
  ```

### next..

- download the blockchain
- run spark job
- demo the api
- demo the frontend
- provide some demo visualizations that can be imported into cassandra (without having to run scanner/bitcoin)
- provide a tool to see how much of the blockchain has been scanned (and which parts?)
