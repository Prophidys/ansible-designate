# Introduction

Designate is the DNSaaS (DNS as a Service) component for Openstack, it manages domains and creates records for these domains through an API. It can also handle events sent by Nova (compute) and Neutron (network) to automatically create one or more records when a VM is created.

In this blog post we will see how Designate works and learn some tricks to manage a production environment. For all tests we use CentOS 7.1 and the RDO packages. We will use a standalone Designate, not linked to Nova or Neutron event (record on the fly for each VM creation).

In designate you can use many backend as an Authoritative DNS, in our case we will use PowerDNS with MariaDB. We will also use Keystone for the authentication part, Designate can run without auth but the CLI doesn’t work well with no auth.



# Base Architecture

As an example architecture we will use 3 servers : 
* 1 as a Controller with all designate services
* 2 as Public Authoritative DNS.



The controller will work as a hidden master to populate and sync domains for the public DNS.

Services Descriptions

**designate-api :**  
It’s the user endpoint to create or deletes domains and records.

**designate-central:**  
    As mentioned in the name it’s the central point of Designate, he will manage the API requests through RabbitMQ and ensure the coherence of the database.

**designate-mdns:**  
    This service is the hidden DNS part of designate, he send AXFR notifications to all DNS declared in the config file and allow zone transfer from them (he simply read the database when a transfer is asked and send it via AXFR).

**designate-pool-manager:**  
    The Pool Manager is the service in charge of the data coherence, he manage the states of all Public DNS by creating the domain on each slaves. In the PowerDNS case it’s a mysql insert directly done in the slave database.



# Installation

 The installation is really simple in our case and to be more easy an ansible playbook is avaible here : https://github.com/Prophidys/ansible-designate all you have to do is bootstrap 3 new CentOS 7.1 and the script will bootstrap a fully working designate.

You have to update the “hosts_example” file with the right IPs/roles and launch the command :

    ansible-playbook -i hosts_example site.yml

All your designate should be up and running. But we need to fulfil the database to start delivering some DNS record.

On the NS-Master we have to declare public NS before adding the first domain and adding some records (don’t forget the dot “.” at the end of each domain) :

    # source designaterc
    # designate server-create --name ns1.example.org.
    # designate server-create --name ns2.example.org.
    # designate domain-create --name example.org. --email admin@example.org
    # designate record-create --type A --name ns1.example.org. --data <IP1> <Domain UUID>
    # designate record-create --type A --name ns2.example.org. --data <IP2> <Domain UUID>

And we can check if everything is working correctly by using dig (bind-utils package on CentOS) :

    # dig NS example.org @<IP1>
    ns1.example.org.
    ns2.example.org.

# Monitoring

In public authoritative DNS platform one important thing to monitor is the availability of the UDP port 53 to answer DNS request and of course the synchronisation between the hidden master and the public DNS.

If your API stop working correctly that can broke some new deployment but all records already register remain in place and keep working. 

# Scalability

You have multiple way to scale. For this part I will just talk about designate (not in an exhaustive way) and not problems we can face with tooling like RabbitMQ or MariaDB.

* **First case, you have too many DNS request on your public nodes**

Good for you it’s the easiest case, you just have to add a new public DNS node to take a part of the load. (See the section of Adding a new slave).

* **Second case, you have too many domain on the same pool**

For this case you can add a new pool with new public DNS Server to divide the domain list length between the pools.

* **Third case, you have too many update through the API**

This case can happen when you have a huge cloud or huge panel of customers who want to update records at the same time. It’s not easy to deal with this kind of situation. You can upgrade your pool number but the limiting point will be MariaDB and RabbitMQ.

### Adding a new Slave

The first step to add a new Public Authoritative DNS into our pool is to bootstrap a new server with MySQL and PowerDNS like other existing slaves.

After that we will add the server to the Pool Manager configuration by editing /etc/designate/designate.conf to look like this :

    [pool:794ccc2c-d751-44fe-b57f-8894c9f5c842]
    nameservers = 9A9AEA42-2028-49EF-A2D2-07E285068590,8CEBA47B-9A32-45A1-AB9F-8C7715E598DF,<UUID of new one>
    targets = 9A9AEA42-2028-49EF-A2D2-07E285068590,8CEBA47B-9A32-45A1-AB9F-8C7715E598DF,<UUID of new one>
     
    [pool_nameserver:9A9AEA42-2028-49EF-A2D2-07E285068590]
    host = 10.0.0.50
    port = 53
    [pool_nameserver:8CEBA47B-9A32-45A1-AB9F-8C7715E598DF]
    host = 10.0.0.49
    port = 53
    [pool_nameserver:<UUID of new one>]
    host = 10.0.0.47
    port = 53

    [pool_target:9A9AEA42-2028-49EF-A2D2-07E285068590]
    options = connection: mysql://pdns:XqMZFSxRBNEdSt8@10.0.0.50/pdns
    masters = 10.0.0.51:5354
    type = powerdns
    host = 10.0.0.50
    port = 53
    [pool_target:8CEBA47B-9A32-45A1-AB9F-8C7715E598DF]
    options = connection: mysql://pdns:XqMZFSxRBNEdSt8@10.0.0.49/pdns
    masters = 10.0.0.51:5354
    type = powerdns
    host = 10.0.0.49
    port = 53
    [pool_target:<UUID of new one>]
    options = connection: mysql://pdns:XqMZFSxRBNEdSt8@10.0.0.47/pdns
    masters = 10.0.0.51:5354
    type = powerdns
    host = 10.0.0.47
    port = 53

Now we can restart pool-manager on the NS-Master to load the new configuration :

    # systemctl restart designate-pool-manager

The new slave database need to be populated (still on the NS-Master) :

    # designate-manage powerdns sync <UUID of new one>

PowerDNS need to be restarted on the new slave node to be on the freshly installed DB.

We will check if everything is working correctly by syncing all the domains of the pool to the new server :

    # designate sync-all

And we can test if all domains are synced :

    # dig +short example.org @<IP of the new slave>

If everything look good we can add the server to the NS records

    # designate server-create “ns3.example.org.”
    # designate record-create --type A --name ns3.example.org. --data <IP> <Domain UUID>

Sync this modification to all nodes :

    # designate sync-all

Check if the modifications is propagated :

    # dig NS example.org +short @10.0.0.49

All theses steps have been package in one ansible playbook to help. Just add a new category to the hosts file with the title [new-slave] and your new slave IP. After that just launch the right playbook :

    # ansible-playbook -i hosts new-slave.yml

### Adding a new Pool

The need to add a new pool can be useful for scalability reason but also if you want to create a different type of service like a VIP zone with better performance / more availability or a particular configuration like internal TLD .local or .dev.

As seen before in the configuration it’s pretty easy, just boostrap N new slaves and declare a new pool in your designate.conf file pointing on theses new slaves.

Unfortunately I wasn’t able to test this kind of features because I didn’t find how to specify a pool-id in the CLI to create a domain on a particular pool.

### Restore a crashed Server

All the data were kept in the master DB so if one of your slave disappear you just have to recreate a new one and sync all your data via the “designate sync-all”. Of course it should get the same IP to avoid all propagation problems.

If you lose your master be sure to have a backup of the DB, if you have a backup it will not be a problem to recreate a new one and inject the DB. Until the reopening of the all API you won’t be able to create, update or delete any record, but all existing records will continue to be served by the slaves.

# Conclusions

The main limitation I found in designate is the lack of automatisation in the creation of pool / ns-slave. All configuration is written in a flat file. It could be nice to just do “designate pool-create” and get all thing in DB like many others services in openstack. I hope it will be available soon :-)
