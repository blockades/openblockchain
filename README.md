# OpenBlockChain

### Architecture

The project is split into several services:

- **Cassandra** holds the entire data: blocks, transactions, and visualisations (analysed data).
- **Bitcoin** is used to connect to the blockchain. It's a simple Bitcoin Core node
whose role is to index all the blocks and transactions and make them consumable through a JSON RPC interface.
- **Scanner** connects to the bitcoin service through its APIs and stores all the blocks and transactions
in the Cassandra database.
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

1. Install Docker Engine

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

2. Install Docker Machine

  `docker-machine` is used to provision machines for the cluster.

  Download the Docker Machine 0.7 binary and extract it to your PATH:

  ```
  $ sudo curl -L https://github.com/docker/machine/releases/download/v0.7.0/docker-machine-`uname -s`-`uname -m` > /usr/local/bin/docker-machine
  $ sudo chmod +x /usr/local/bin/docker-machine
  ```

  Check the installation by displaying the Machine version:

  ```
  $ docker-machine version
  ```

  Always up-to-date instructions are [available at docker.com](https://docs.docker.com/machine/install-machine/).

3. Install Docker Compose

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

### Cluster Setup

1. Create a Docker Swarm

  Create a VirtualBox machine called local on your system:

  ```
  $ docker-machine create -d virtualbox local
  ```

  Load the master machine configuration into your shell:

  ```
  $ eval "$(docker-machine env local)"
  ```

  Generate a discovery token using the Docker Swarm image:

  ```
  $ docker pull swarm
  $ docker run --rm swarm create # output of this command = the discovery token
  ```

  You can now stop `local`:

  ```
  $ docker-machine stop local
  ```

2. Launch the Docker Swarm (cluster)

  *a) Using VirtualBox*

  You can create the cluster locally, using [VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads):

  ```
  $ docker-machine create -d virtualbox --swarm --swarm-master --swarm-discovery token://<TOKEN-FROM-ABOVE> opb-master
  $ docker-machine create -d virtualbox --swarm --swarm-discovery token://<TOKEN-FROM-ABOVE> opb-agent-00
  $ docker-machine create -d virtualbox --swarm --swarm-discovery token://<TOKEN-FROM-ABOVE> opb-agent-01
  ```

  ```
  TODO: maybe specify RAM above?
  ```

  Note: an internet connection is still required (for Swarm's node discovery service).

  *b) Using DigitalOcean*

  Or you can provision the machines from DigitalOcean:

  ```
  TODO
  ```

  *c) Using Scaleway*

  Or you can provision the machines from Scaleway:

  ```
  TODO
  ```

3. Verify the swarm

  Point your Docker environment to the machine running the swarm master:

  ```
  $ eval $(docker-machine env --swarm opb-master)
  ```

  Print info:

  ```
  $ docker info
  ```

### Launch services

Point your Docker environment to the machine running the swarm master:

```
$ eval $(docker-machine env --swarm opb-master)
```

Create a default network:

```
$ docker network create opbnet
```

1. Cassandra, Bitcoin, Scanner

  ```
  $ cd scanner/
  ```

  ```
  TODO: dc up -d --build
  ```

2. Spark

  ```
  $ cd spark/
  ```

  ```
  TODO: dc up -d --build
  ```

3. API

  ```
  $ cd api/
  ```

  ```
  TODO: dc up -d --build
  ```

4. Frontend

  ```
  $ cd frontend/
  ```

  ```
  TODO: dc up -d --build
  ```

### next..

- download the blockchain
- run spark job
- demo the api
- demo the frontend
