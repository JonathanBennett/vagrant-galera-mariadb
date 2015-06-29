# MariaDB Galera Cluster using Vagrant including MaxScale router

This repo is a fork of the work by @travelping - so thanks for the helpful base. This builds on that and adds a MaxScale router VM alongside the 3 VMs already created and exposes a port on 4009 for a R/W Split Router.

## Configuration

The following settings in the Vagrantfile must be configured:

**The nodes count in the Cluster: (Defunt in this repo)**

`CLUSTER_SIZE = 3`

**IP address of the first node in the Cluster:**

`FROM_IP = "192.168.100.101"`

**All nodes in the Cluster:**

`ALL_NODES_IN_CLUSTER = ["192.168.100.101","192.168.100.102","192.168.100.103"]`

The first time that you run the cluster, a new user `galera` with all privileges will be created in MariaDB with a `galera` password. Default MariaDB port: `3306` will be opened on every node in the Cluster. With default configuration and after `vagrant up` you have:

```

h1 on 192.168.100.101

h2 on 192.168.100.102

h3 on 192.168.100.103
```


## Usage

To run the Cluster:

`vagrant up --no-parallel`

To establish the SSH connection with first host type:

`vagrant ssh h1`

To establish the SQL connection with first host type:

`mysql -ugalera -pgalera -h192.168.100.101 -P3306`
