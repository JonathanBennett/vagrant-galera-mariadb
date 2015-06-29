# -*- mode: ruby -*-
# vi: set ft=ruby :

#cluster configuration
# CLUSTER_SIZE = 3
START_CLUSTER_ID = 1
FROM_IP = "192.168.100.101"
ALL_NODES_IN_CLUSTER = ["192.168.100.101","192.168.100.102","192.168.100.103"]
HOST_PORT_START = 3307
BOX_MEMORY = 2048

#spetial NW settings
START_INTERFACE_ID = 1
HOSTNAME_PREF = 'h'


def get_nodes (count, from_ip, hostname_pref)
    nodes = []
    ip_arr = from_ip.split('.')
    first_ip_part = "#{ip_arr[0]}.#{ip_arr[1]}.#{ip_arr[2]}"
    count.times do |i|
        hostname = "%s%01d" % [hostname_pref, (i+START_INTERFACE_ID)]
        nodes.push([i+START_CLUSTER_ID, hostname, "#{first_ip_part}.#{ip_arr.last.to_i+i}", i+HOST_PORT_START])
    end
    nodes
end

def provision_node(hostaddr, node_addresses)
    setup = <<-SCRIPT
sudo apt-get  -q -y update
sudo apt-get  -q -y install python-software-properties vim curl wget tmux socat
sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db
sudo add-apt-repository 'deb http://ftp.osuosl.org/pub/mariadb/repo/10.0/ubuntu precise main'

sudo apt-get  -q -y update
echo mariadb-galera-server-10.0 mysql-server/root_password password root | debconf-set-selections
echo mariadb-galera-server-10.0 mysql-server/root_password_again password root | debconf-set-selections

LC_ALL=en_US.utf8 DEBIAN_FRONTEND=noninteractive sudo apt-get -o Dpkg::Options::='--force-confnew' -qqy install mariadb-galera-server galera mariadb-client

gpg --keyserver hkp://keys.gnupg.net --recv-keys 1C4CBDCDCD2EFD2A
gpg -a --export CD2EFD2A | sudo apt-key add -
sudo add-apt-repository 'deb http://repo.percona.com/apt precise main'
sudo apt-get  -q -y update
sudo apt-get -qqy install percona-xtrabackup

echo "[mysqld]
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_name=ma_cluster
wsrep_cluster_address="gcomm://#{node_addresses.join(',')}"
wsrep_slave_threads=2
wsrep_sst_method=xtrabackup
wsrep_sst_auth=galera:galera
wsrep_node_address=#{hostaddr}" > /etc/mysql/conf.d/galera.cnf

echo "[client]
port        = 3306
socket      = /var/run/mysqld/mysqld.sock
[mysqld_safe]
socket      = /var/run/mysqld/mysqld.sock
nice        = 0
[mysqld]
user        = mysql
pid-file    = /var/run/mysqld/mysqld.pid
socket      = /var/run/mysqld/mysqld.sock
port        = 3306
basedir     = /usr
datadir     = /var/lib/mysql
tmpdir      = /tmp
lc_messages_dir = /usr/share/mysql
lc_messages = en_US
binlog_format = ROW
default-storage-engine = InnoDB
innodb_autoinc_lock_mode = 2
innodb_doublewrite = 1
query_cache_size = 0
query_cache_type = OFF
bind-address        = 0.0.0.0
max_connections     = 10000
max_connect_errors  = 10000
connect_timeout     = 5
wait_timeout        = 600
max_allowed_packet  = 16M
thread_cache_size       = 128
sort_buffer_size    = 4M
bulk_insert_buffer_size = 16M
tmp_table_size      = 32M
max_heap_table_size = 32M
key_buffer_size     = 128M
table_open_cache    = 400
concurrent_insert   = 2
read_buffer_size    = 2M
read_rnd_buffer_size    = 1M
# * Query Cache Configuration
query_cache_limit       = 128K
log_warnings        = 2
log_error	= /var/log/mysql/error.log
slow_query_log_file = /var/log/mysql/mariadb-slow.log
long_query_time = 10
#log_slow_rate_limit    = 1000
log_slow_verbosity  = query_plan
log_bin         = /var/log/mysql/mariadb-bin
log_bin_index       = /var/log/mysql/mariadb-bin.index
expire_logs_days    = 10
max_binlog_size         = 100M
default_storage_engine  = InnoDB
#innodb_log_file_size   = 50M
innodb_buffer_pool_size = 2G
innodb_log_buffer_size  = 8M
innodb_file_per_table   = 1
innodb_open_files   = 1000
innodb_io_capacity  = 1000
innodb_flush_method = O_DIRECT
[mysqldump]
quick
quote-names
max_allowed_packet  = 16M
[isamchk]
key_buffer      = 16M
!includedir /etc/mysql/conf.d/" > /etc/mysql/my.cnf

echo "# Automatically generated for Debian scripts. DO NOT TOUCH!
[client]
host     = localhost
user     = debian-sys-maint
password = some_pwd
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = debian-sys-maint
password = some_pwd
socket   = /var/run/mysqld/mysqld.sock
basedir  = /usr" > /etc/mysql/debian.cnf

mysql -u root -proot -e 'GRANT ALL PRIVILEGES on *.* TO "debian-sys-maint"@'localhost' IDENTIFIED BY "some_pwd" WITH GRANT OPTION; FLUSH PRIVILEGES;'
mysql -u root -proot -e 'CREATE DATABASE tetete;'
mysql -uroot -proot -e "INSERT INTO mysql.user (Host,User) values ('%','ha_chk'); FLUSH PRIVILEGES;"

sudo service mysql stop
sleep 5
    SCRIPT
end

def start_cluster()
    ret = <<-SCRIPT
sudo service mysql start --wsrep-new-cluster
sleep 10
mysql -uroot -proot -e 'GRANT ALL ON *.* TO 'galera'@'localhost' IDENTIFIED BY "galera"'
mysql -uroot -proot -e 'GRANT ALL ON *.* TO 'galera'@"%" IDENTIFIED BY "galera"'
    SCRIPT
end

def attachnode()
    ret = <<-SCRIPT
nohup bash -c 'sleep 30;sudo service mysql start;' > /home/vagrant/nohup.out &
    SCRIPT
end

def install_maxscale()
    ret = <<-SCRIPT
wget https://downloads.mariadb.com/software/mariadb-maxscale/configure-maxscale-repo-0.1.2.deb
sudo dpkg -i configure-maxscale-repo-0.1.2.deb
sudo apt-get update
sudo apt-get install -y maxscale

echo "## Example MaxScale.cnf configuration file
#
# Global parameters
#
# Number of worker threads in MaxScale
#
#   threads=<number of threads>
#
# Enabled logfiles. The message log is enabled by default and
# the error log is always enabled.
#
# log_messages=<1|0>
# log_trace=<1|0>
# log_debug=<1|0>
## Example:

[maxscale]
threads=4

## Define a monitor that can be used to determine the state and role of
# the servers.
#
# Currently valid options for all monitors are:
#
#   module=[mysqlmon|galeramon]
#
# List of server names which are being monitored
#
#   servers=<server name 1>,<server name 2>,...,<server name N>
#
# Username for monitor queries, need slave replication and slave client privileges
# Password in plain text format, and monitor's sampling interval in milliseconds.
#
#   user=<username>
#   passwd=<plain txt password>
#   monitor_interval=<sampling interval in milliseconds> (default 10000)
#
# Timeouts for monitor operations in backend servers - optional.
#
#   backend_connect_timeout=<timeout in seconds>
#   backend_write_timeout=<timeout in seconds>
#   backend_read_timeout=<timeout in seconds>
#
## MySQL monitor-specific options:
#
# Enable detection of replication slaves lag via replication_heartbeat
# table - optional.
#
#   detect_replication_lag=[1|0] (default 0)
#
# Allow previous master to be available even in case of stopped or misconfigured
# replication - optional.
#
#   detect_stale_master=[1|0] (default 0)
#
## Galera monitor-specific options:
#
# If disable_master_failback is not set, recovery of previously failed master
# causes mastership to be switched back to it. Enabling the option prevents it.
#
#   disable_master_failback=[0|1] (default 0)
#
## Examples:

[Galera Monitor]
type=monitor
module=galeramon
servers=server1,server2,server3
user=galera
passwd=galera
monitor_interval=10000
#disable_master_failback=

## Filter definition
#
# Type specifies the section
#
#   type=filter
#
# Module specifies which module implements the filter function
#
#   module=[qlafilter|regexfilter|topfilter|teefilter]
#
# Options specify the log file for Query Log Filter
#
#   options=<path to logfile>
#
# Match and replace are used in regexfilter
#
#   match=fetch
#   replace=select
#
# Count and filebase are used with topfilter to specify how many top queries are
# listed and where.
#
#   count=<count>
#   filebase=<path to output file>
#
# Match and service are used by tee filter to specify what queries should be
# duplicated and where the copy should be routed.
#
#   match=insert.*HighScore.*values
#   service=Cassandra
#
## Examples:

[qla]
type=filter
module=qlafilter
options=/tmp/QueryLog

[fetch]
type=filter
module=regexfilter
match=fetch
replace=select


## A series of service definition
#
# Name of router module, currently valid options are
#
#   router=[readconnroute|readwritesplit|debugcli|CLI]
#
# List of server names for use of service - mandatory for readconnroute,
# readwritesplit, and debugcli
#
#   servers=<server name 1>,<server name 2>,...,<server name N>
#
# Username to fetch password information with and password in plaintext
# format - for readconnroute and readwritesplit
#
#   user=<username>
#   passwd=<password in plain text format>
#
# flag for enabling the use of root user - for readconnroute and
# readwritesplite - optional.
#
#   enable_root_user=[0|1] (default 0)
#
# Version string to be used in server handshake. Default value is that of
# MariaDB embedded library's - for readconnroute and readwritesplite - optional.
#
#   version_string=<specific version string>
#
# Filters specify the filters through which the query is transferred and the
# order of their appearance on the list corresponds the order they are
# used. Values refer to names of filters configured in this file - for
# readconnroute and readwritesplit - optional.
#
#   filters=<filter name1|filter name2|...|filter nameN>
#
## Read Connection Router specific router options.
#
# router_options specify the role in which the selected server must be.
#
#   router_options=[master|slave|synced]
#
## Read/Write Split Router specific options.
#
# use_sql_variables_in specifies where sql variable modifications are
# routed - optional.
#
#   use_sql_variables_in=[master|all] (default all)
#
# router_options=slave_selection_criteria specifies the selection criteria for
# slaves both in new session creation and when route target is selected - optional.
#
#   router_options=
#   slave_selection_criteria=[LEAST_CURRENT_OPERATIONS|LEAST_BEHIND_MASTER]
#
# router_options=max_sescmd_history specifies a limit on the number of 'session commands'
# a single session can execute. Please refer to the configuration guide for more details - optional.
#
#   router_options=
#   max_sescmd_history=2500
#
# max_slave_connections specifies how many slaves a router session can
# connect to - optional.
#
#       max_slave_connections=<number, or percentage, of all slaves>
#
# max_slave_replication_lag specifies how much a slave is allowed to be behind
# the master and still become chosen routing target - optional, requires that
# monitor has detect_replication_lag=1 .
#
#       max_slave_replication_lag=<allowed lag in seconds for a slave>
#
# Valid router modules currently are:
#   readwritesplit, readconnroute, debugcli and CLI
#
## Examples:

#[Read Connection Router]
#type=service
#router=readconnroute
#servers=server1,server2,server3
#user=myuser
#passwd=mypwd
#router_options=synced

[RW Split Router]
type=service
router=readwritesplit
servers=server1,server2,server3
user=galera
passwd=galera
#use_sql_variables_in=
#max_slave_connections=100%
#max_slave_replication_lag=21
#router_options=slave_selection_criteria=
#filters=fetch|qla

[Debug Interface]
type=service
router=debugcli

[CLI]
type=service
router=cli

## Listener definitions for the services
#
# Type specifies section as listener one
#
# type=listener
#
# Service links the section to one of the service names used in this configuration
#
#   service=<name of service section>
#
# Protocol is client protocol library name.
#
#   protocol=[MySQLClient|telnetd|HTTPD|maxscaled]
#
# Port and address specify which port the service listens and the address limits
# listening to a specific network interface only. Address is optional.
#
#   port=<Listening port>
#   address=<Address to bind to>
#
# Socket is alternative for address. The specified socket path must be writable
# by the Unix user MaxScale runs as.
#
#   socket=<Listening socket>
#
## Examples:

#[Read Connection Listener]
#type=listener
#service=Read Connection Router
#protocol=MySQLClient
#address=192.168.100.102
#port=4008
#socket=/tmp/readconn.sock

[RW Split Listener]
type=listener
service=RW Split Router
protocol=MySQLClient
port=4009
#socket=/tmp/rwsplit.sock

[Debug Listener]
type=listener
service=Debug Interface
protocol=telnetd
#address=127.0.0.1
port=4442

[CLI Listener]
type=listener
service=CLI
protocol=maxscaled
#address=localhost
port=6603

## Definition of the servers
#
# Type specifies the section as server one
#
#   type=server
#
# The IP address or hostname of the machine running the database server that is
# being defined. MaxScale will use this address to connect to the backend
# database server.
#
#   address=<IP|hostname>
#
# The port on which the database listens for incoming connections. MaxScale
# will use this port to connect to the database server.
#
#   port=<port>
#
# The name for the protocol module to use to connect MaxScale to the database.
# Currently the only backend protocol supported is the MySQLBackend module.
#
#   protocol=MySQLBackend
#
## Examples:

[server1]
type=server
address=192.168.100.101
port=3306
protocol=MySQLBackend

[server2]
type=server
address=192.168.100.102
port=3306
protocol=MySQLBackend

[server3]
type=server
address=192.168.100.103
port=3306
protocol=MySQLBackend" > /usr/local/mariadb-maxscale/etc/MaxScale.cnf

sudo service maxscale start

    SCRIPT
end

Vagrant.configure("2") do |config|
    config.vm.box = "hashicorp/precise64"
    config.ssh.username = "vagrant"
#    config.ssh.password = "vagrant"
    config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"
    config.vm.provider "virtualbox" do |v|
      v.memory = BOX_MEMORY
      v.cpus = 1
    end
    if Vagrant.has_plugin?("vagrant-cachier")
        config.cache.scope = :box
        config.cache.enable :apt
    end

    cluster_nodes = get_nodes(CLUSTER_SIZE, FROM_IP, HOSTNAME_PREF)

    cluster_nodes.each do |in_cluster_position, hostname, hostaddr, hostport|
        config.vm.define hostname do |box|
            box.vm.hostname = "#{hostname}"
            #box.vm.network :private_network, ip: "#{hostaddr}", :netmask => "255.255.0.0"
            #box.vm.network :private_network, ip: "#{hostaddr}", :netmask => "255.255.0.0",  virtualbox__intnet: true
            box.vm.network :public_network, ip: "#{hostaddr}"

            box.vm.network "forwarded_port", guest:3306, host:"#{hostport}", auto_correct:true

            box.vm.provision :shell, :inline => provision_node(hostaddr, ALL_NODES_IN_CLUSTER)

            if in_cluster_position == 1
                box.vm.provision :shell, :inline => start_cluster()
            else
                box.vm.provision :shell, :inline => attachnode()
            end
        end
    end

    config.vm.define "maxscale" do |maxscale|
        maxscale.vm.provider "virtualbox" do |v|
            v.memory = BOX_MEMORY
            v.cpus = 1
        end
        if Vagrant.has_plugin?("vagrant-cachier")
            maxscale.cache.scope = :box
            maxscale.cache.enable :apt
        end

        maxscale.vm.hostname = "maxscale"

        maxscale.vm.network :public_network, ip:"192.168.100.104"
        maxscale.vm.network "forwarded_port", guest:4009, host:4009, auto_correct:true
        maxscale.vm.provision :shell, :inline => install_maxscale()
	end

end
