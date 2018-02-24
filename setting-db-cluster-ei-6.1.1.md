
# Setting up the Database

Each Carbon-based product uses a database to store information such as user management details and registry data. All nodes in the cluster must use one central database for config and governance registry mounts. These instructions assume you are installing MySQL as your relational database management system (RDBMS), but you can install another supported RDBMS as needed. 

You can create the following databases and associated datasources.

WSO2_USER_DB
JDBC user store and authorization manager
REGISTRY_DB	Shared database for config and governance registry mounts in the product's nodes
REGISTRY_LOCAL1	Local registry space in the manager node
REGISTRY_LOCAL2	Local registry space in the worker node

The following diagram illustrates how these databases are connected to the manager and worker nodes.
![Alt text](http://ziben.com.br/coisitas/images/db-cluster-esb.png)

### The following topics will guide you through all the configurations necessary to set up databases for clustering. 

* Creating the databases
* Configuring the manager node
* Configuring the worker node
* Mounting the registry on manager and worker nodes
* Creating the databases
Do the following steps to create the databases necessary. Note that we use MySQL here as an example, but you can use any suitable database instead.

Download and install MySQL Server.

Download the MySQL JDBC driver.

Unzip the downloaded MySQL driver zipped archive, and copy the MySQL JDBC driver JAR (mysql-connector-java-x.x.xx-bin.jar) into the <PRODUCT_HOME>/repository/components/lib directory of both the manager and worker nodes.

Define the host name for configuring permissions for the new database by opening the /etc/hosts file and adding the following line:
<MYSQL-DB-SERVER-IP> carbondb.mysql-wso2.com
You would do this step only if your database is not on your local machine and on a separate server.

Enter the following command in a terminal/command window, where username is the username you want to use to access the databases:
mysql -u username -p
When prompted, specify the password that will be used to access the databases with the username you specified.
Create the databases using the following commands, where <PRODUCT_HOME> is the path to any of the product instances you installed, and username and password are the same as those you specified in the previous steps:

About using MySQL in different operating systems

For users of Microsoft Windows, when creating the database in MySQL, it is important to specify the character set as latin1. Failure to do this may result in an error (error code: 1709) when starting your cluster. This error occurs in certain versions of MySQL (5.6.x) and is related to the UTF-8 encoding. MySQL originally used the latin1 character set by default, which stored characters in a 2-byte sequence. However, in recent versions, MySQL defaults to UTF-8 to be friendlier to international users. Hence, you must use latin1 as the character set as indicated below in the database creation commands to avoid this problem. Note that this may result in issues with non-latin characters (like Hebrew, Japanese, etc.). The following is how your database creation command should look.

mysql> create database <DATABASE_NAME> character set latin1;
For users of other operating systems, the standard database creation commands will suffice. For these operating systems, the following is how your database creation command should look.

mysql> create database <DATABASE_NAME>;
mysql> create database WSO2_USER_DB;
mysql> use WSO2_USER_DB;
mysql> source <PRODUCT_HOME>/dbscripts/mysql.sql;
mysql> source <PRODUCT_HOME>/dbscripts/identity/mysql.sql;
mysql> grant all on WSO2_USER_DB.* TO regadmin@"carbondb.mysql-wso2.com" identified by "regadmin";
 
mysql> create database REGISTRY_DB;
mysql> use REGISTRY_DB;
mysql> source <PRODUCT_HOME>/dbscripts/mysql.sql;
mysql> grant all on REGISTRY_DB.* TO regadmin@"carbondb.mysql-wso2.com" identified by "regadmin";
 
mysql> create database REGISTRY_LOCAL1;
mysql> use REGISTRY_LOCAL1;
mysql> source <PRODUCT_HOME>/dbscripts/mysql.sql;
mysql> grant all on REGISTRY_LOCAL1.* TO regadmin@"carbondb.mysql-wso2.com" identified by "regadmin";
  
mysql> create database REGISTRY_LOCAL2;
mysql> use REGISTRY_LOCAL2;
mysql> source <PRODUCT_HOME>/dbscripts/mysql.sql;
mysql> grant all on REGISTRY_LOCAL2.* TO regadmin@"carbondb.mysql-wso2.com" identified by "regadmin";
For MySQL 5.7:

From Carbon kernel 4.4.6 onwards your product will be shipped with two scripts for MySQL as follows (click here to see if your product is based on this kernel version or newer):

mysql.sql : Use this script for MySQL versions prior to version 5.7.

mysql5.7.sql : Use this script for MySQL 5.7 and later versions.
 
Note that if you are automatically creating databases during server startup using the -DSetup option, the mysql.sql script will be used by default to set up the database. Therefore, if you have MySQL version 5.7 set up for your server, be sure to do the following before starting the server:

First, change the existing mysql.sql file to a different filename.

Change the <PRODUCT_HOME>/dbscripts/mysql5.7.sql script to mysql.sql.
Change the <PRODUCT_HOME>/dbscripts/identity/mysql5.7.sql script to mysql.sql.
MySQL 5.7 is only recommended for products that are based on Carbon 4.4.6 or a later version.

Configuring the manager node
Do the following configurations in the manager node of your cluster.

On the manager node, open the <PRODUCT_HOME>/repository/conf/datasources/master-datasource.xml file, and configure the datasources to point to the REGISTRY_LOCAL1, WSO2_REGISTRY_DB, and WSO2_USER_DB databases as follows (change the username, password, and database URL as needed for your environment).

<datasources-configuration xmlns:svns="http://org.wso2.securevault/configuration">
     <providers>
        <provider>org.wso2.carbon.ndatasource.rdbms.RDBMSDataSourceReader</provider>
    </providers>
    <datasources>
        <datasource>
            <name>REGISTRY_LOCAL1</name>
            <description>The datasource used for registry- local</description>
            <jndiConfig>
                <name>jdbc/WSO2CarbonDB</name>
            </jndiConfig>
            <definition type="RDBMS">
                <configuration>
                    <url>jdbc:mysql://carbondb.mysql-wso2.com:3306/REGISTRY_LOCAL1?autoReconnect=true</url>
                    <username>regadmin</username>
                    <password>regadmin</password>
                    <driverClassName>com.mysql.jdbc.Driver</driverClassName>
                    <maxActive>50</maxActive>
                    <maxWait>60000</maxWait>
                    <testOnBorrow>true</testOnBorrow>
                    <validationQuery>SELECT 1</validationQuery>
                    <validationInterval>30000</validationInterval>
                </configuration>
            </definition>
        </datasource>
        <datasource>
            <name>REGISTRY_DB</name>
            <description>The datasource used for registry- config/governance</description>
            <jndiConfig>
                <name>jdbc/WSO2RegistryDB</name>
            </jndiConfig>
            <definition type="RDBMS">
                <configuration>
                    <url>jdbc:mysql://carbondb.mysql-wso2.com:3306/REGISTRY_DB?autoReconnect=true</url>
                    <username>regadmin</username>
                    <password>regadmin</password>
                    <driverClassName>com.mysql.jdbc.Driver</driverClassName>
                    <maxActive>50</maxActive>
                    <maxWait>60000</maxWait>
                    <testOnBorrow>true</testOnBorrow>
                    <validationQuery>SELECT 1</validationQuery>
                    <validationInterval>30000</validationInterval>
                </configuration>
            </definition>
        </datasource>
         <datasource>
            <name>WSO2_USER_DB</name>
            <description>The datasource used for registry and user manager</description>
            <jndiConfig>
                <name>jdbc/WSO2UMDB</name>
            </jndiConfig>
            <definition type="RDBMS">
                <configuration>
                    <url>jdbc:mysql://carbondb.mysql-wso2.com:3306/WSO2_USER_DB</url>
                    <username>regadmin</username>
                    <password>regadmin</password>
                    <driverClassName>com.mysql.jdbc.Driver</driverClassName>
                    <maxActive>50</maxActive>
                    <maxWait>60000</maxWait>
                    <testOnBorrow>true</testOnBorrow>
                    <validationQuery>SELECT 1</validationQuery>
                    <validationInterval>30000</validationInterval>
                </configuration>
            </definition>
        </datasource>
   </datasources>
</datasources-configuration>
Make sure to replace username and password with your MySQL database username and password.

To configure the datasource, update the dataSource property found in <PRODUCT_HOME>/repository/conf/user-mgt.xml of the manager node as shown below:

<Property name="dataSource">jdbc/WSO2UMDB</Property>
 

You must also update the dataSource property found in the <PRODUCT_HOME>/repository/conf/registry.xml file of the worker node as shown below.

<dbConfig name="sharedregistry">   
    <dataSource>jdbc/WSO2RegistryDB</dataSource>
</dbConfig>


### Configuring the worker node

Do the following configurations in the worker node of your cluster.

On the worker node, open the $PRODUCT_HOME/conf/datasources/master-datasource.xml file and configure the datasources to point to the REGISTRY_LOCAL2, WSO2_REGISTRY_DB, and WSO2_USER_DB databases as follows (change the username, password, and database URL as needed for your environment):
```xml
<datasources-configuration xmlns:svns="http://org.wso2.securevault/configuration">
     <providers>
        <provider>org.wso2.carbon.ndatasource.rdbms.RDBMSDataSourceReader</provider>
    </providers>
    <datasources>
        <datasource>
            <name>REGISTRY_LOCAL2</name>
            <description>The datasource used for registry- local</description>
            <jndiConfig>
                <name>jdbc/WSO2CarbonDB</name>
            </jndiConfig>
            <definition type="RDBMS">
                <configuration>
                    <url>jdbc:mysql://carbondb.mysql-wso2.com:3306/REGISTRY_LOCAL2?autoReconnect=true</url>
                    <username>regadmin</username>
                    <password>regadmin</password>
                    <driverClassName>com.mysql.jdbc.Driver</driverClassName>
                    <maxActive>50</maxActive>
                    <maxWait>60000</maxWait>
                    <testOnBorrow>true</testOnBorrow>
                    <validationQuery>SELECT 1</validationQuery>
                    <validationInterval>30000</validationInterval>
                </configuration>
            </definition>
        </datasource>
        <datasource>
            <name>REGISTRY_DB</name>
            <description>The datasource used for registry- config/governance</description>
            <jndiConfig>
                <name>jdbc/WSO2RegistryDB</name>
            </jndiConfig>
            <definition type="RDBMS">
                <configuration>
                    <url>jdbc:mysql://carbondb.mysql-wso2.com:3306/REGISTRY_DB?autoReconnect=true</url>
                    <username>regadmin</username>
                    <password>regadmin</password>
                    <driverClassName>com.mysql.jdbc.Driver</driverClassName>
                    <maxActive>50</maxActive>
                    <maxWait>60000</maxWait>
                    <testOnBorrow>true</testOnBorrow>
                    <validationQuery>SELECT 1</validationQuery>
                    <validationInterval>30000</validationInterval>
                </configuration>
            </definition>
        </datasource>
         <datasource>
            <name>WSO2_USER_DB</name>
            <description>The datasource used for registry and user manager</description>
            <jndiConfig>
                <name>jdbc/WSO2UMDB</name>
            </jndiConfig>
            <definition type="RDBMS">
                <configuration>
                    <url>jdbc:mysql://carbondb.mysql-wso2.com:3306/WSO2_USER_DB</url>
                    <username>regadmin</username>
                    <password>regadmin</password>
                    <driverClassName>com.mysql.jdbc.Driver</driverClassName>
                    <maxActive>50</maxActive>
                    <maxWait>60000</maxWait>
                    <testOnBorrow>true</testOnBorrow>
                    <validationQuery>SELECT 1</validationQuery>
                    <validationInterval>30000</validationInterval>
                </configuration>
            </definition>
        </datasource>
   </datasources>
</datasources-configuration>
```

> Make sure to replace username and password with your MySQL database username and password.

To configure the datasource, update the dataSource property found in the <PRODUCT_HOME>/repository/conf/user-mgt.xml file of the worker node as shown below.

<Property name="dataSource">jdbc/WSO2UMDB</Property>
You must also update the dataSource property found in the <PRODUCT_HOME>/repository/conf/registry.xml file of the worker node as shown below.

<dbConfig name="sharedregistry">   
    <dataSource>jdbc/WSO2RegistryDB</dataSource>
</dbConfig>
Mounting the registry on manager and worker nodes
We do this step to ensure that the shared registry for governance and config is mounting to both the nodes. This database is REGISTRY_DB.

Configure the shared registry database and mounting details in the <PRODUCT_HOME>/repository/conf/registry.xml file of the manager node as shown below:

Note: The existing dbConfig called wso2registry must not be removed when adding the following configurations.

```xml
<dbConfig name="sharedregistry">
    <dataSource>jdbc/WSO2RegistryDB</dataSource>
</dbConfig>
 
<remoteInstance url="https://localhost:9443/registry">
    <id>instanceid</id>
    <dbConfig>sharedregistry</dbConfig>
    <readOnly>false</readOnly>
    <enableCache>true</enableCache>
    <registryRoot>/</registryRoot>
    <cacheId>regadmin@jdbc:mysql://carbondb.mysql-wso2.com:3306/REGISTRY_DB?autoReconnect=true</cacheId>
</remoteInstance>
 
<mount path="/_system/config" overwrite="true">
    <instanceId>instanceid</instanceId>
    <targetPath>/_system/config</targetPath>
</mount>
 
<mount path="/_system/governance" overwrite="true">
    <instanceId>instanceid</instanceId>
    <targetPath>/_system/governance</targetPath>
</mount>
Configure the shared registry database and mounting details in <PRODUCT_HOME>/repository/conf/registry.xml of the worker node as shown below:

<dbConfig name="sharedregistry">
    <dataSource>jdbc/WSO2RegistryDB</dataSource>
</dbConfig>
 
<remoteInstance url="https://localhost:9443/registry">
    <id>instanceid</id>
    <dbConfig>sharedregistry</dbConfig>
    <readOnly>true</readOnly>
    <enableCache>true</enableCache>
    <registryRoot>/</registryRoot>
    <cacheId>regadmin@jdbc:mysql://carbondb.mysql-wso2.com:3306/REGISTRY_DB?autoReconnect=true</cacheId>
</remoteInstance>
 
<mount path="/_system/config" overwrite="true">
    <instanceId>instanceid</instanceId>
    <targetPath>/_system/config</targetPath>
</mount>
 
<mount path="/_system/governance" overwrite="true">
    <instanceId>instanceid</instanceId>
    <targetPath>/_system/governance</targetPath>
</mount>
```
The following are some key points to note when adding these configurations:

The dataSource you specify under the <dbConfig name="sharedregistry"> tag must match the jndiConfig name specified in the master-datasources.xml file of the manager and worker.
The registry mount path is used to identify the type of registry. For example, ”/_system/config” refers to configuration registry, and "/_system/governance" refers to the governance registry.
The dbconfig entry enables you to identify the datasource you configured in the master-datasources.xml file. We use the unique name sharedregistry to refer to that datasource entry. 
The remoteInstance section refers to an external registry mount. We can specify the read-only/read-write nature of this instance as well as caching configurations and the registry root location. In case of a worker node, the readOnly property should be true, and in case of a manager node, this property should be set to false. 
Additionally, we must specify cacheId, which enables caching to function properly in the clustered environment. Note that cacheId is the same as the JDBC connection URL of the registry database. This value is the cacheId of the remote instance. Here the cacheId should be in the format of $database_username@$database_url, where $database_username is the username of the remote instance database and $database_url is the remote instance database URL. This cacheID is used to identify the cache it should look for when caching is enabled. In this case, the database we should connect to is REGISTRY_DB, which is the database shared across all the master/workers nodes. You can identify that by looking in the mounting configurations, where the same datasource is being used.
You must define a unique name “id” for each remote instance, which is then referred to from mount configurations. In the above example, the unique ID for the remote instance is instanceId. 
In each of the mounting configurations, we specify the actual mount path and target mount path. The targetPath can be any meaningful name. In this instance, it is /_system/config.
Now your database is set up.
