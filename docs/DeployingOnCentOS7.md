# Deploying a Mesos Cluster with Calico on CentOS7

In these instructions, we will install Mesos and Calico Networking on baremetal CentOS/Red Hat Enterprise Linux 7 from rpm packages. These packages can be built from the Mesosphere [net-modules repo](https://github.com/mesosphere/net-modules) and will install the following processes:

 * Mesos Master
 * Mesos Agent
 * net-modules
 * Calico

To simplify deployment, ZooKeeper and etcd will run in Docker containers on the Master host.

## Preparation

For the installation of Mesos + Calico, you will need Docker 1.7+. You must also verify your host has a unique fully qualified domain name. Lastly, we recommend disabling the firewall on the Mesos communication ports.

The instruction below will walk you through the steps.

### Prepare External Services

You will need to run one instance of both, etcd and ZooKeeper. Most Mesos cluster will run the backend services in a cluster external to their core Mesos cluster. For simplicity's sake, we will be launching ZooKeeper and etcd on the master host.

In this guide, both services will be run in Docker containers, but that is not mandatory.

Also, keep note of the IP address of the host you deploy on because we will need this later when configuring our Mesos masters and agents.

#### Docker (Optional)

Docker can be used to quickly deploy services, such as etcd, ZooKeeper, and Calico node (a package of Calico's core components). 

Docker is not a mandatory component of the installation, but this guide will walk you through deploying some services with Docker. 

    $ sudo yum -y install docker docker-selinux
    $ sudo systemctl enable docker.service
    $ sudo systemctl start docker.service

##### Verify Docker installation

    $ sudo docker run hello-world

*Optional:* You may also want to create a `docker` group and add your local user to the group.  This means you can drop the `sudo` in the `docker ...` commands that follow.

    $ sudo groupadd docker
    $ sudo usermod -aG docker `whoami`
    $ sudo systemctl restart docker.service

Then log out (`exit`) and log back in to pick up your new group association.  Verify your user has access to Docker without sudo

    $ docker ps

#### ZooKeeper

    $ sudo docker pull jplock/zookeeper:3.4.5
    $ sudo docker run --detach --name zookeeper -p 2181:2181 jplock/zookeeper:3.4.5

#### etcd

`etcd` needs your fully qualified domain name to start correctly.

    $ sudo docker pull quay.io/coreos/etcd:v2.2.0
    $ export FQDN=`hostname -f`
    $ sudo mkdir -p /var/etcd
    $ sudo docker run --detach --name etcd --net host -v /var/etcd:/data quay.io/coreos/etcd:v2.2.0 \
     --advertise-client-urls "http://${FQDN}:2379,http://${FQDN}:4001" \
     --listen-client-urls "http://0.0.0.0:2379,http://0.0.0.0:4001" \
     --data-dir /data

If you have SELinux policy enforced, you must perform the following step:

    $ sudo chcon -Rt svirt_sandbox_file_t /var/etcd

### Prepare Each Host

The following steps should be performed on each Master and Agent in the cluster.

#### Set & verify fully qualified domain name

These instructions assume each host can reach other hosts using their fully qualified domain names (FQDN).  To check the FQDN on a host use

    $ hostname -f

Then attempt to ping that name from other servers.

Also important are that Calico and Mesos have the same view of the (non-fully-qualified) hostname.  In particular, the value returned by

    $ hostname

must be unique for each node in your cluster.  Both Calico and Mesos use this value to identify the host.

#### Build Mesos

The Mesos RPM packages can be built from the Mesosphere [net-modules repo](https://github.com/mesosphere/net-modules).

    $ git clone git@github.com:mesosphere/net-modules.git
    $ cd net-modules
    $ make builder-rpm
    $ ls packages/rpms/RPMS/x86_64

The files you see in the `packages/rpms/RPMS/x86_64` directory are all the packages you need to install Mesos, while additionally including the compiled net-modules library. Transfer these files to each host in your cluster.

#### Install Mesos

If you followed the guide correctly so far, you should have the Mesos rpm packages on each of your master and agent hosts.

Before installing these packages, you must have the Extra Packages for Enterprise Linux (or EPEL) packages installed.

    $ sudo yum -y install epel-release

Now your server should be ready to install the Mesos packages.

    $ sudo yum -y install *.rpm


## Configure Master
### Configure your firewall

You will either need to configure the firewalls on each node in your cluster (recommended) to allow access to the cluster services or disable it completely.  Included in this section is configuration examples for `firewalld`.  If you use a different firewall, check your documentation for how to open the listed ports.

Master node(s) require

| Service Name | Port/protocol     |
|--------------|-------------------|
| zookeeper    | 2181/tcp          |
| mesos-master | 5050/tcp          |
| etcd         | 2379/tcp 4001/tcp |
| marathon     | 8080/tcp          |

Example `firewalld` config

    $ sudo firewall-cmd --zone=public --add-port=2181/tcp --permanent
    $ sudo firewall-cmd --zone=public --add-port=5050/tcp --permanent
    $ sudo firewall-cmd --zone=public --add-port=2379/tcp --permanent
    $ sudo firewall-cmd --zone=public --add-port=4001/tcp --permanent
    $ sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
    $ sudo systemctl restart firewalld

### Set Master Environment Variables

An explanation of the configuration options for Mesos can be found by running `mesos-init-wrapper -h`. 

We will be need to set the correct environment variables for the master. These environment variables are interpreted as command line arguments in the mesos-master application at runtime.

First, you will need set the ZooKeeper URL in `/etc/mesos/zk`. Modify the line to include the IP address of the host with ZooKeeper running.

The value in `/etc/mesos-master/quorum` may need to change depending on how many master hosts you have in your cluster. Mesos recommends that the quorum count is at least 1/2 the number of master hosts running. 

### Run Mesos Master

Run the mesos-master process on your master host.

    $ sudo systemctl enable mesos-master.service
    $ sudo systemctl start mesos-master.service

## Configure Agent
### Configure your firewall

You will either need to configure the firewalls on each node in your cluster (recommended) to allow access to the cluster services or disable it completely.  Included in this section is configuration examples for `firewalld`.  If you use a different firewall, check your documentation for how to open the listed ports.

Agent (compute) nodes require

| Service Name | Port/protocol     |
|--------------|-------------------|
| BIRD (BGP)   | 179/tcp           |
| mesos-agent  | 5051/tcp          |

Example `firewalld` config

    $ sudo firewall-cmd --zone=public --add-port=179/tcp --permanent
    $ sudo firewall-cmd --zone=public --add-port=5051/tcp --permanent
    $ sudo systemctl restart firewalld

### Download the Calico Mesos Plugin

To obtain the Calico files, you will need `wget` installed. If you haven't already done so, download the tool with `yum -y install wget`.

    $ wget https://github.com/projectcalico/calico-mesos/releases/download/v0.1.1/calico_mesos
    $ chmod +x calico_mesos
    $ sudo mkdir /calico
    $ sudo mv calico_mesos /calico/calico_mesos

### Create the modules.json Configuration File

The `modules.json` file loads the library `libmesos_network_isolator.so` with two modules, `com_mesosphere_mesos_NetworkIsolator` and `com_mesosphere_mesos_NetworkHook`, into Mesos.

    $ cat > modules.json <<EOF
    {
      "libraries": [
        {
          "file": "/opt/net-modules/libmesos_network_isolator.so",
          "modules": [
            {
              "name": "com_mesosphere_mesos_NetworkIsolator",
              "parameters": [
                {
                  "key": "isolator_command",
                  "value": "/calico/calico_mesos"
                },
                {
                  "key": "ipam_command",
                  "value": "/calico/calico_mesos"
                }
              ]
            },
            {
              "name": "com_mesosphere_mesos_NetworkHook"
            }
          ]
        }
      ]
    }
    EOF
    $ sudo mv modules.json /calico/

### Run Calico Node

Download `calicoctl`, Calico's command line tool. We will use `calicoctl` to run Calico's core processes packaged up in a Docker container on each agents. In Docker, this container will be called calico-node. 

When doing so, we must point `calicoctl` to our running instance of etcd. Go back and retrieve the IP address of the host where etcd is running. Use that address in the `ECTD_AUTHORITY` variable passed to `calicoctl`.

    $ wget https://github.com/projectcalico/calico-docker/releases/download/v0.9.0/calicoctl
    $ chmod +x calicoctl
    $ sudo ETCD_AUTHORITY=<IP of host with etcd>:4001 ./calicoctl node

### Set Agent Environment Variables

An explanation of the configuration options for Mesos can be found by running `mesos-init-wrapper -h`. 

We will be need to set the correct environment variables for each agent. These environment variables are interpreted as command line arguments in the mesos-slave application at runtime.

Append the following lines to `/etc/default/mesos-slave` on each of your agent hosts. 

    MESOS_RESOURCES="ports(*):[31000-31100]"
    MESOS_MODULES=file:///calico/modules.json
    MESOS_ISOLATION=com_mesosphere_mesos_NetworkIsolator
    MESOS_HOOKS=com_mesosphere_mesos_NetworkHook
    MESOS_EXECUTOR_REGISTRATION_TIMEOUT=5mins
    ETCD_AUTHORITY=<IP of host with etcd running>:4001
    
Next, you will need set the ZooKeeper URL in `/etc/mesos/zk`. Modify the line to include the IP address of the host with ZooKeeper running.

### Run Agents

Run the mesos-slave process on each of your agent hosts.

    $ sudo systemctl enable mesos-slave.service
    $ sudo systemctl start mesos-slave.service

## Test your cluster

You can quickly verify that Mesos is up and running with the expected number of agents by pointing your browser to the master node, port 5050 (e.g. http://mesos-master.mydomain:5050/ ).

Additionally, you can test Calico network functionality by running our test framework.  On each host, download the framework files to `/framework`

    $ sudo mkdir /framework
    $ cd /framework
    $ sudo wget https://raw.githubusercontent.com/mesosphere/net-modules/integration/0.25/framework/calico_executor.py
    $ sudo wget https://raw.githubusercontent.com/mesosphere/net-modules/integration/0.25/framework/calico_framework.py
    $ sudo wget https://raw.githubusercontent.com/mesosphere/net-modules/integration/0.25/framework/calico_utils.py
    $ sudo wget https://raw.githubusercontent.com/mesosphere/net-modules/integration/0.25/framework/constants.py
    $ sudo wget https://raw.githubusercontent.com/mesosphere/net-modules/integration/0.25/framework/tasks.py

Check to see if your system supports the `nc` command. If not, you will need to install it on each host.

    $ sudo yum install nc
    
Now, on your master host run the framework.

    $ sudo python calico_framework.py

The Calico framework launches a series of tasks on your Mesos cluster to verify network connectivity and network isolation are working correctly.

[calico]: http://projectcalico.org
[mesos]: https://mesos.apache.org/
[net-modules]: https://github.com/mesosphere/net-modules
[docker]: https://www.docker.com/