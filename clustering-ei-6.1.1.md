# Clustering WSO2 EI 6.1.1

This section describes how to set up a WSO2 ESB worker/manager separated cluster and how to configure this cluster with different third-party load balancers. The following sections give you information and instructions on how to set up your cluster.

* Worker/manager separated clustering deployment pattern
* Configuring the load balancer
* Setting up the databases
* Configuring the manager node
* Configuring the worker node
* Testing the cluster

> Important: When configuring your WSO2 products for clustering, it is necessary to use a specific IP address and not localhost or 
> host names in your configurations. So, keep this in mind when hosting WSO2 products in your production environment.

> See Setting up a Cluster in AWS Mode for information on clustering WSO2 products that are deployed on Amazon EC2 instances. The
> instructions in that topic only include the configurations done to the $PRODUCT_HOME/conf/axis2/axis2.xml file, so the configuration
> changes done to other configuration files must be done in addition to the steps in that topic.

### Worker/manager separated clustering deployment pattern

In this pattern there are three WSO2 ESB nodes; 1 node acts as the manager node and 2 nodes act as worker nodes for high availability and serving service requests. In this pattern, we allow access to the admin console through an external load balancer. Additionally, service requests are directed to worker nodes through this load balancer. The following image depicts the sample pattern this clustering deployment scenario will follow.
![Alt text](http://ziben.com.br/coisitas/images/ClusterESB.png)

Here, we use two nodes as well-known members, one is the manager node and the other is one of the worker nodes. It is always recommended to use at least two well-known members to prevent restarting all the nodes in the cluster in case a well known member is shut down.

See Worker/Manager separated clustering patterns for a wider variety of options if you prefer to use a different clustering deployment pattern.

### Configuring the load balancer

The load balancer automatically distributes incoming traffic across multiple WSO2 product instances. It enables you to achieve greater levels of fault tolerance in your cluster, and provides the required balancing of load needed to distribute traffic.

> About clustering without a load balancer
> The configurations in this subsection are not required if your clustering setup does not have a load balancer. If you follow the rest of > the configurations in this topic while excluding this section, you will be able to set up your cluster without the load balancer.`

### Things to keep in mind

The configuration steps in this document are written assuming that default 80 and 443 ports are used and exposed by the 3rd party load balancer for this ESB cluster. If any other ports are used instead of the default ones, please replace 80 and 443 values with the corresponding ports in the relevant places.

### So with the above in mind, please note the following:

> Load balancer ports are HTTP 80 and HTTPS 443 as indicated in the deployment pattern above.
> Direct the HTTP requests to the worker nodes using http://xxx.xxx.xxx.xx3/<service> via HTTP 80 port.
> Direct the HTTPS requests to the worker nodes using https://xxx.xxx.xxx.xx3/<service> via HTTPS 443 port.
> Access the management console as https://xxx.xxx.xxx.xx2/carbon via HTTPS 443 port
> In a WSO2 ESB cluster, the worker nodes address service requests on the PassThrough Transport ports (8280 and 8243) and can access the Management Console using the HTTPS 9443 port.
        
> Tip: We recommend that you use NGINX Plus as your load balancer of choice.

### Use the following steps to configure NGINX as the load balancer for WSO2 products.

* Install NGINX Plus in a server configured in your cluster.
* Configure NGINX Plus to direct the HTTP requests to the two worker nodes via the HTTP 80 port using the http://esb.wso2.com/<service>. To do this, create a VHost file (esb.http.conf) in the /etc/nginx/conf.d directory and add the following configurations into it.

```
upstream wso2.esb.com {
        server xxx.xxx.xxx.xx3:8280;
        server xxx.xxx.xxx.xx4:8280;
}
 
server {
        listen 80;
        server_name esb.wso2.com;
        location / {
               proxy_set_header X-Forwarded-Host $host;
               proxy_set_header X-Forwarded-Server $host;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header Host $http_host;
               proxy_read_timeout 5m;
               proxy_send_timeout 5m;
               proxy_pass http://wso2.esb.com;
        }
}
```

Configure NGINX Plus to direct the HTTPS requests to the two worker nodes via the HTTPS 443 port using https://esb.wso2.com/<service>. To do this, create a VHost file (esb.https.conf) in the /etc/nginx/conf.d directory and add the following configurations into it.
```
upstream ssl.wso2.esb.com {
    server xxx.xxx.xxx.xx3:8243;
    server xxx.xxx.xxx.xx4:8243;
  
            sticky learn create=$upstream_cookie_jsessionid
            lookup=$cookie_jsessionid
            zone=client_sessions:1m;
}
 
server {
listen 443;
    server_name esb.wso2.com;
    ssl on;
    ssl_certificate /etc/nginx/ssl/wrk.crt;
    ssl_certificate_key /etc/nginx/ssl/wrk.key;
    location / {
               proxy_set_header X-Forwarded-Host $host;
               proxy_set_header X-Forwarded-Server $host;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header Host $http_host;
               proxy_read_timeout 5m;
               proxy_send_timeout 5m;
        proxy_pass https://ssl.wso2.esb.com;
        }
}
```

Configure NGINX Plus to access the Management Console as https://mgt.esb.wso2.com/carbon via HTTPS 443 port. This is to direct requests to the manager node. To do this, create a VHost file (mgt.esb.https.conf) in the /etc/nginx/conf.d directory and add the following configurations into it.
```
server {
    listen 443;
    server_name mgt.esb.wso2.com;
    ssl on;
    ssl_certificate /etc/nginx/ssl/mgt.crt;
    ssl_certificate_key /etc/nginx/ssl/mgt.key;
 
    location / {
               proxy_set_header X-Forwarded-Host $host;
               proxy_set_header X-Forwarded-Server $host;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header Host $http_host;
               proxy_read_timeout 5m;
               proxy_send_timeout 5m;
        proxy_pass https://xxx.xxx.xxx.xx2:9443/;
        }
    error_log  /var/log/nginx/mgt-error.log ;
           access_log  /var/log/nginx/mgt-access.log;
}
```

* Restart the NGINX Plus server.
```
$sudo service nginx restart
```
> Tip: You do not need to restart the server if you are simply making a modification to the VHost file. The following command should be sufficient in such cases.
```
$sudo service nginx reload 
```

### Create SSL certificates

Create SSL certificates for both the manager and worker nodes using the instructions that follow.

* Create the Server Key.
```
$sudo openssl genrsa -des3 -out server.key 1024
```
* Certificate Signing Request.
```
$sudo openssl req -new -key server.key -out server.csr
```
* Remove the password.
```
$sudo cp server.key server.key.org
$sudo openssl rsa -in server.key.org -out server.key
```
* Sign your SSL Certificate.
```
$sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```
While creating keys, enter the host name (esb.wso2.com or mgt.esb.wso2.com) as the common name.

### Setting up the databases

See Setting up the Database for information on how to set up the databases for a cluster. The datasource configurations must be done in the $PRODUCT_HOME/conf/datasources/master-datasources.xml file for both the manager and worker nodes. You would also have to configure the shared registry database and mounting details in the $PRODUCT_HOME/conf/registry.xml file.

### Configuring the manager node

* [Download](http://ziben.com.br/coisitas/wso2ei-6.1.1.zip)  and unzip the WSO2 ESB binary distribution. Consider the extracted directory as $PRODUCT_HOME.

* Set up the cluster configurations. Edit the <PRODUCT_HOME>/repository/conf/axis2/axis2.xml file as follows.
* Enable clustering for this node:
```xml
<clustering class="org.wso2.carbon.core.clustering.hazelcast.HazelcastClusteringAgent" enable="true">
```
* Set the membership scheme to wka to enable the well-known address registration method (this node sends cluster initiation messages to the WKA members that we define later). 
```xml
<parameter name="membershipScheme">wka</parameter>
```
* Specify the name of the cluster this node will join.
```xml
<parameter name="domain">wso2.esb.domain</parameter>
```
* Specify the host used to communicate cluster messages.
```xml
<parameter name="localMemberHost">xxx.xxx.xxx.xx2</parameter>
```
* Specify the port used to communicate cluster messages. This port number is not affected by the port offset value specified in the <PRODUCT_HOME>/conf/carbon.xml. If this port number is already assigned to another server, the clustering framework automatically increments this port number. However, if two servers are running on the same machine, you must ensure that a unique port is set for each server.
```xml
<parameter name="localMemberPort">4100</parameter>
```
* Specify the well-known members. In this example, the well-known member is a worker node. The port value for the WKA worker node must be the same value as it's localMemberPort (in this case 4200).
```xml
<members>
    <member>
        <hostName>xxx.xxx.xxx.xx3</hostName>
        <port>4200</port>
    </member>
</members>
```

> Although this example only indicates one well-known member, it is recommended to add at least two well-known members here. This is done to ensure that there is high availability for the cluster.

> You can also use IP address ranges for the hostName. For example, 192.168.1.2-10. This should ensure that the cluster eventually recovers after failures. One shortcoming of doing this is that you can define a range only for the last portion of the IP address. You should also keep in mind that the smaller the range, the faster the time it takes to discover members since each node has to scan a lesser number of potential members.

* Change the following clustering properties. Ensure that you set the value of the subDomain as mgt to specify that this is the manager node. This ensures that traffic for the manager node is routed to this member.
```xml
<parameter name="properties">
      <property name="backendServerURL" value="https://${hostName}:${httpsPort}/services/"/>
      <property name="mgtConsoleURL" value="https://${hostName}:${httpsPort}/"/>
      <property name="subDomain" value="mgt"/>
</parameter>
```

* Configure the `HostName`. To do this, edit the <PRODUCT_HOME>/conf/carbon.xml file as follows.
```xml
<HostName>esb.wso2.com</HostName>
<MgtHostName>mgt.esb.wso2.com</MgtHostName>
```

* Enable SVN-based deployment synchronization with the AutoCommit property marked as true. To do this, edit the <PRODUCT_HOME>/conf/carbon.xml file as follows. See Configuring Deployment Synchronizer for more information on this.
```xml
<DeploymentSynchronizer>
    <Enabled>true</Enabled>
    <AutoCommit>true</AutoCommit>
    <AutoCheckout>true</AutoCheckout>
    <RepositoryType>svn</RepositoryType>
    <SvnUrl>https://svn.wso2.org/repos/esb</SvnUrl>
    <SvnUser>svnuser</SvnUser>
    <SvnPassword>xxxxxx</SvnPassword>
    <SvnUrlAppendTenantId>true</SvnUrlAppendTenantId>
</DeploymentSynchronizer>
```

* Map the host names to the IP. Add the below host entries to your DNS, or “/etc/hosts” file (in Linux) in all the nodes of the cluster. You can map the IP address of the database server. In this example, MySQL is used as the database server, so <MYSQL-DB-SERVER-IP> is the actual IP address of the database server.

```
<IP-of-MYSQL-DB-SERVER>  carbondb.mysql-wso2.com
```

* Allow access the management console only through the load balancer. Configure the HTTP/HTTPS proxy ports to communicate through the load balancer by editing the $PRODUCT_HOME/conf/tomcat/catalina-server.xml file as follows.
```xml
<Connector protocol="org.apache.coyote.http11.Http11NioProtocol"
    port="9763"
    proxyPort="80"
    ...
    />
<Connector protocol="org.apache.coyote.http11.Http11NioProtocol"
    port="9443"
    proxyPort="443"
    ...
    />
 ```

### Configuring the worker node

* Download and unzip the WSO2 ESB binary distribution. Consider the extracted directory as <PRODUCT_HOME>.
* Set up the cluster configurations. Edit the <PRODUCT_HOME>/conf/axis2/axis2.xml file as follows.
* Enable clustering for this node.
```
<clustering class="org.wso2.carbon.core.clustering.hazelcast.HazelcastClusteringAgent" enable="true">
```
* Set the membership scheme to wka to enable the well-known address registration method (this node will send cluster initiation messages to WKA members that we will define later).
```
<parameter name="membershipScheme">wka</parameter>
```
* Specify the name of the cluster this node will join.
```xml
<parameter name="domain">wso2.esb.domain</parameter>
```
* Specify the host used to communicate cluster messages.
```xml
<parameter name="localMemberHost">xxx.xxx.xxx.xx3</parameter>
```
* Specify the port used to communicate cluster messages. If this node is on the same server as the manager node, or another worker node, set this to a unique value, such as 4000 and 4001 for worker nodes 1 and 2. This port number will not be affected by the port offset in carbon.xml. If this port number is already assigned to another server, the clustering framework will automatically increment this port number.
```xml
<parameter name="localMemberPort">4200</parameter>
```

* Define the sub-domain as worker by adding the following property under the  <parameter name="properties">  element: 
```xml
<property name="subDomain" value="worker"/>
```

* Specify the well known member by providing the host name and localMemberPort values. Here, the well known member is the manager node. Defining the manager node is useful since it is required for the Deployment Synchronizer to function in an efficient manner. The deployment synchronizer uses this configuration to identify the manager and synchronize deployment artifacts across the nodes of a cluster.
```xml
<members>
    <member>
        <hostName>xxx.xxx.xxx.xx2</hostName>
        <port>4100</port>
    </member>
</members>
```
> Although this example only indicates one well-known member, it is recommended to add at least two well-known members here. This is done to ensure that there is high availability for the cluster.

> You can also use IP address ranges for the hostName. For example, 192.168.1.2-10. This should ensure that the cluster eventually recovers after failures. One shortcoming of doing this is that you can define a range only for the last portion of the IP address. You should also keep in mind that the smaller the range, the faster the time it takes to discover members, since each node has to scan a lesser number of potential members.

* Uncomment and edit the WSDLEPRPrefix element under `org.apache.synapse.transport.passthru.PassThroughHttpListener` and
`org.apache.synapse.transport.passthru.PassThroughHttpSSLListener` in the transportReceiver.
```xml
<parameter name="WSDLEPRPrefix" locked="false">http://esb.wso2.com:80</parameter>
<parameter name="WSDLEPRPrefix" locked="false">https://esb.wso2.com:443</parameter>
```

* Configure the $PRODUCT_HOME/conf/carbon.xml file to match your clustering configurations with your product.

* Configure the `HostName`. To do this, edit the $PRODUCT_HOME/conf/carbon.xml file as follows.
```xml
<HostName>esb.wso2.com</HostName>
```
* Enable SVN-based deployment synchronization with the AutoCommit property marked as false. To do this, edit the $PRODUCT_HOME/conf/carbon.xml file as follows.
```xml
<DeploymentSynchronizer>
    <Enabled>true</Enabled>
    <AutoCommit>false</AutoCommit>
    <AutoCheckout>true</AutoCheckout>
    <RepositoryType>svn</RepositoryType>
    <SvnUrl>https://svn.wso2.org/repos/esb</SvnUrl>
    <SvnUser>svnuser</SvnUser>
    <SvnPassword>xxxxxx</SvnPassword>
    <SvnUrlAppendTenantId>true</SvnUrlAppendTenantId>
</DeploymentSynchronizer>
```
* In the $PRODUCT_HOME/conf/carbon.xml file, you can also specify the port offset value. This is ONLY applicable if you have multiple WSO2 products hosted on the same server.

 Click here for more information on configuring the port offset.
 ```xml
<Ports>
    ...
    <Offset>0</Offset>
    ...
</Ports>
```
* Map the host names to the IP. Add the following host entries to your DNS, or “/etc/hosts” file (in Linux) in all the nodes of the cluster. You can map the IP address of the database server. In this example, MySQL is used as the database server, so <MYSQL-DB-SERVER-IP> is the actual IP address of the database server.
```
<IP-of-MYSQL-DB-SERVER>   carbondb.mysql-wso2.com
```
* Create the second worker node by getting a copy of the WSO2 product you just configured as a worker node and change the following in the $PRODUCT_HOME/conf/axis2/axis2.xml file. This copy of the WSO2 product can be moved to a server of its own.
```xml
<parameter name="localMemberPort">4300</parameter>
```

# Testing the cluster

* Restart the configured load balancer.
* Start the manager node. The additional `-Dsetup` argument creates the required tables in the database.
```
sh $PRODUCT_HOME/bin/wso2server.sh -Dsetup
```
* Start the two worker nodes. The additional -DworkerNode=true argument indicates that this is a worker node. This parameter basically makes a server read-only. A node with this parameter will not be able to do any changes such as writing or making modifications to the deployment repository etc. This parameter also enables the worker profile, where the UI bundles will not be activated and only the back end bundles will be activated once the server starts up. When you configure the axis2.xml file, the cluster sub domain must indicate that this node belongs to the "worker" sub domain in the cluster.

> Tip: It is recommendation is to delete the <PRODUCT_HOME>/repository/deployment/server directory and create an empty server directory in the worker node. This is done to avoid any SVN conflicts that may arise.
```
sh $PRODUCT_HOME/bin/wso2server.sh -DworkerNode=true
```
* Check for ‘member joined’ log messages in all consoles.

> Additional information on logs and new nodes

> When you terminate one node, all nodes identify that the node has left the cluster. The same applies when a new node joins the cluster. 

> If you want to add another new worker node, you can simply copy worker1 without any changes if you are running it on a new server (such as xxx.xxx.xxx.184). If you intend to use the new node on a server where another WSO2 product is running, you can use a copy of worker1 and change the port offset accordingly in the carbon.xml file. You also have to change localMemberPort in axis2.xml if that product has clustering enabled. Be sure to map all host names to the relevant IP addresses when creating a new node. The log messages indicate if the new node joins the cluster.

* Access management console through the LB using the following URL: https://xxx.xxx.xxx.xx2:443/carbon
* Test load distribution via http://xxx.xxx.xxx.xx3:80/ or https://xxx.xxx.xxx.xx3:443/.
* Add a sample proxy service with the log mediator in the inSequence so that logs will be displayed in the worker terminals, and then observe the cluster messages sent from the manager node to the workers.

The load balancer manages the active and passive states of the worker nodes, activating nodes as needed and leaving the rest in passive mode. To test this, send a request to the end point through the load balancer to verify that the proxy service is activated only on the active worker node(s) while the remaining worker nodes remain passive. For example, you would send the request to the following URL:

http://{Load_Balancer_Mapped_URL_for_worker}/services/{Sample_Proxy_Name}

