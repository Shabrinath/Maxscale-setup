# How to Install and Configure MaxScale for MariaDB


A database proxy is an intermediary layer that manages connections between applications and database servers, optimizing traffic flow, enhancing security, and providing failover support. It acts as a single access point, balancing the load across multiple database servers, managing read/write splits, and handling reconnections in case of database node failures.

MaxScale is a proxy specifically designed for MariaDB databases. It helps distribute database traffic, ensuring high availability and scalability. By using MaxScale, you can route read queries to replica nodes and write queries to the master node, which improves performance and reliability in a MariaDB cluster.

Adding a load balancer between your application and database can help manage high traffic by distributing it across multiple database nodes, provide a single endpoint to manage failover scenarios, and allow for different read/write port allocations. If you're working with a MariaDB cluster, MaxScale serves as a reliable database proxy.

This guide covers the steps to manually install, configure, and test MaxScale for a MariaDB replication cluster using CentOS8.

---

## Prerequisites

- **MariaDB database cluster** set up and running (1 master and 1 slave node recommended)
- **Separate machine** for MaxScale installation (to allow MaxScale to continue working even if the primary database node fails)

---

## Step 1: Install MaxScale

1. Add the MariaDB repository on your MaxScale server:
    ```bash
    curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash
    ```

2. Install the MaxScale package:
    ```bash
    yum install maxscale
    ```

---

## Step 2: Configure MaxScale

1. **Create a MaxScale user** with the necessary privileges in MariaDB:

    ```sql
    CREATE USER 'maxscaleuser'@'%' IDENTIFIED BY 'maxscalepassword';
    GRANT SELECT ON mysql.user TO 'maxscaleuser'@'%';
    GRANT SELECT ON mysql.db TO 'maxscaleuser'@'%';
    GRANT SELECT ON mysql.tables_priv TO 'maxscaleuser'@'%';
    GRANT SELECT ON mysql.roles_mapping TO 'maxscaleuser'@'%';
    GRANT SHOW DATABASES ON *.* TO 'maxscaleuser'@'%';
    GRANT REPLICATION CLIENT ON *.* TO 'maxscaleuser'@'%';
    #execute below 3 queries if you face auth error: 
    GRANT SLAVE MONITOR ON *.* TO 'maxscaleuser'@'%';
    GRANT REPLICATION CLIENT ON *.* TO 'maxscaleuser'@'%';
    GRANT SUPER ON *.* TO 'maxscaleuser'@'%';


    -- For MariaDB 10.2.2 to 10.2.10
    GRANT SELECT ON mysql.* TO 'maxscaleuser'@'%';
    ```

2. **Edit the MaxScale configuration file** (`/etc/maxscale.cnf`), defining servers, monitoring settings, services, and listeners:

    ```ini
    # Global parameters
    [maxscale]
    threads = auto
    admin_port=8989
    log_augmentation = 1
    ms_timestamp = 1
    syslog = 1

    # Server definitions
    [server1]
    type=server
    address=192.168.100.126  # Master node
    port=3306
    protocol=MariaDBBackend

    [server2]
    type=server
    address=192.168.100.127  # Slave node
    port=3306
    protocol=MariaDBBackend

    # Monitor for the servers
    [MariaDB-Monitor]
    type=monitor
    module=mariadbmon
    servers=server1,server2
    user=maxscaleuser
    password=maxscalepassword
    monitor_interval=2000

    # Service definitions
    [Read-Only-Service]
    type=service
    router=readconnroute
    servers=server2
    user=maxscaleuser
    password=maxscalepassword
    router_options=slave

    [Read-Write-Service]
    type=service
    router=readwritesplit
    servers=server1
    user=maxscaleuser
    password=maxscalepassword

    # Listener definitions for the services
    [Read-Only-Listener]
    type=listener
    service=Read-Only-Service
    protocol=MariaDBClient
    port=4008

    [Read-Write-Listener]
    type=listener
    service=Read-Write-Service
    protocol=MariaDBClient
    port=4006
    ```

    This configuration defines:
    - **Server1** (Master) and **Server2** (Slave) with corresponding IPs
    - A **monitor** for health checks
    - Separate **read-only** and **read-write** services
    - **Listeners** for each service with dedicated ports (4008 for read-only, 4006 for read-write)

---

## Step 3: Start and Verify MaxScale

1. **Start the MaxScale service**:
    ```bash
    systemctl start maxscale.service
    ```

2. **Verify the service and servers** using `maxctrl`:
    ```bash
    maxctrl list services
    maxctrl list servers
    ```

---

## Step 4: Test the Connection

You can now test MaxScale by connecting to the MariaDB nodes through the MaxScale IP and port.

1. **Testing Read-Write Connection**:
    ```bash
    mysql -h 192.168.100.128 -umaxscaleuser -pmaxscalepassword -P4006 -e 'SELECT @@hostname;'
    ```

   Expected output:
   ```plaintext
   +------------+
   | @@hostname |
   +------------+
   | server1    |
   +------------+
1. **Testing Read-only Connection**:
    ```bash
    mysql -h 192.168.100.128 -umaxscaleuser -pmaxscalepassword -P4008 -e 'SELECT @@hostname;'
    ```

   Expected output:
   ```plaintext
   +------------+
   | @@hostname |
   +------------+
   | server2    |
   +------------+
## Step 5: Access Maxscale GUI
We can access Maxscale GUI on port 8989. 
