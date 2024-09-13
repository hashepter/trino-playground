# Trino-Playground Tutorial

A simple tutorial for setting up Trino and two databases.

Example queries are provided to demonstrate Trino's ability to query different databases and how to combine data from multiple database sources to provide a single result.

## Docker Setup: Postgres

Spin up a Postgres Docker container by running the following commands. Use the password of 'postgrespw' for this tutorial

```
docker run --name postgres -e POSTGRES_PASSWORD=postgrespw -p 5432:5432 -d postgres &&
docker exec -it postgres bash
```

Open psql CLI

```
psql -U postgres
```

Setup Postgres

```
CREATE DATABASE store WITH OWNER postgres ENCODING='UTF8' LC_COLLATE='en_US.utf8' LC_CTYPE='en_US.utf8';
```

Switch to the "store" database

```
\c store
```

Create the table that will hold data for purchased items. Each record will hold a purchaseId, customerId, item, and a total

```
CREATE TABLE purchase (
    purchaseId SERIAL PRIMARY KEY,
    customerId INT,
    item VARCHAR(255),
    total NUMERIC(10, 2)
);
```

Verify the purchase table was created

```
\dt+
```

Insert mock data into the purchase table

```
INSERT INTO purchase (customerId, item, total) VALUES
  (1001, 'Stapler', 8.50),
  (1002, 'Notepad', 3.25),
  (1003, 'Ballpoint Pens', 4.50),
  (1004, 'Highlighters', 5.00),
  (1005, 'Binder', 7.95),
  (1006, 'Folder', 1.75),
  (1007, 'Desk Organizer', 9.90),
  (1008, 'Tape', 2.45),
  (1009, 'Markers', 6.30),
  (1010, 'Glue Stick', 0.99);
```

Verify the data has been inserted into the purchase table

```
SELECT * FROM purchase;
```

Quit Postgres CLI

```
quit
```

Exit the Postgres Docker container

```
exit
```

## Docker Setup: MySQL

Spin up a Postgres Docker container by running the following commands. use a default password of 'mysqlpw'

```
docker run --name mysql -e MYSQL_ROOT_PASSWORD=mysqlpw -p 3306:3306 -d mysql &&
docker exec -it mysql bash
```

Open mysql CLI (You can set the password to 'mysqlpw' for this tutorial)

```
mysql -u root -p
```

Create the Customer database

```
CREATE DATABASE Customer;
```

Switch to the Customer database

```
USE Customer;
```

Create the table that will hold data for customers. Each record will hold a CustomerId, FirstName, LastName, City, State

```
CREATE TABLE Customer (
  CustomerId INT PRIMARY KEY,
  FirstName VARCHAR(255),
  LastName VARCHAR(255),
  City VARCHAR(255),
  State VARCHAR(50)
);
```

Insert mock data into the Customer table

```
INSERT INTO Customer (CustomerId, FirstName, LastName, City, State) VALUES
  (1001, 'John', 'Doe', 'Springfield', 'IL'),
  (1002, 'Jane', 'Doe', 'Centerville', 'CA'),
  (1003, 'Jim', 'Beam', 'Franklin', 'TX'),
  (1004, 'Jill', 'Hill', 'Clinton', 'NY'),
  (1005, 'Jack', 'Black', 'Hollywood', 'CA'),
  (1006, 'Jessica', 'Smith', 'Liberty', 'WA'),
  (1007, 'Jerry', 'Springer', 'Hope', 'NJ'),
  (1008, 'Joan', 'Ark', 'Bethlehem', 'PA'),
  (1009, 'Joe', 'Shmoe', 'Smallville', 'KS'),
  (1010, 'Jasmine', 'Rice', 'Plains', 'GA');
```

Verify the data has been inserted into the Customer table

```
SELECT * FROM Customer;
```

Quit MySQL CLI

```
quit
```

Exit the MySQL Docker container

```
exit
```

(Optional) DBeaver Connection Setup for PostgreSQL by setting the following fields

Host: localhost
Database: store
Port 5432
Username: postgres
Password: postgrespw

Run a test query

```
SELECT \* FROM purchase;
```

(Optional) DBeaver Connection Setup for MySQL by setting the following fields

```
URL: jdbc:mysql://localhost:3306/Customer?allowPublicKeyRetrieval=true&useSSL=false
Username: root
Password: mysqlpw
```

Run a test query

```
SELECT \* FROM Customer;
```

## Docker: Setup Trino

Run the following commnads to spin up a Docker container to setup Trino

```
docker pull redhat/ubi9 &&
docker run -td --name trinosandbox -p 8443:8443 redhat/ubi9 &&
docker exec -ti trinosandbox bash
```

Update the container and install the following packages

```
yum update -y &&
yum install vim wget curl httpd-tools python3 openssl net-tools iputils-ping -y --skip-broken
```

Run the following command to check your type of CPU.

```
uname -m
```

Go visit the Oracle downloads page to download the Java 22 RPM package: [https://www.oracle.com/java/technologies/downloads/]()

_Download the **ARM64 RPM** Package if 'uname -m' gave you; armv6l, armv7l, armv8l, aarch64, arm64, or armhf_

Copy the link to the exact RPM that you selected and run the following command

```
echo "Please enter the URL of the Java RPM package: " && read myvar && export RPM_PACKAGE="$myvar"
```

Run the following set of commands to install Java in the Trino container

```
wget $RPM_PACKAGE &&
yum localinstall jdk-22_linux-x64_bin.rpm -y &&
rm -f jdk-22_linux-x64_bin.rpm &&
java -version
```

Create 'python' symlink for 'python3'

```
ln -s /usr/bin/python3 /usr/bin/python &&
python --version
```

Install Trino Server

```
wget "https://repo1.maven.org/maven2/io/trino/trino-server/454/trino-server-454.tar.gz" &&
tar -xvzf trino-server-454.tar.gz &&
mkdir -p /usr/local/trino &&
mv trino-server-454/\* /usr/local/trino
```

Setup Trino Launcher in ~/.bashrc

```
printf "\n# Trino\n" >> ~/.bashrc &&
echo 'export TRINO_HOME=/usr/local/trino' >> ~/.bashrc &&
echo 'export PATH=$TRINO_HOME/bin:$PATH' >> ~/.bashrc &&
source ~/.bashrc &&
rm -f trino-server-454.tar.gz
rm -rf trino-server-454
```

Verify Trino server installation

```
launcher --help
```

Change directory to Trino and make and 'cd' into 'etc'

```
cd /usr/local/trino &&
mkdir etc &&
cd etc
```

Download Trino CLI

```
wget https://repo1.maven.org/maven2/io/trino/trino-cli/454/trino-cli-454-executable.jar &&
mv trino-cli-454-executable.jar trino &&
chmod +x trino &&
./trino --version
```

Create 'testuser' and set the password to "trinopw" for this tutorial

```
touch password.db &&
htpasswd -B -C 10 password.db testuser
```

Run the following commands to setup the files and directories for the Trino Server

```
touch password-authenticator.properties &&
echo "password-authenticator.name=file" >> password-authenticator.properties &&
echo "file.password-file=/usr/local/trino/etc/password.db" >> password-authenticator.properties
```

```
touch node.properties &&
echo "node.environment=production" >> node.properties &&
echo "node.id=ffffffff-ffff-ffff-ffff-ffffffffffff" >> node.properties &&
echo "node.data-dir=/usr/local/trino/data" >> node.properties
```

```
touch config.properties &&
echo "coordinator=true" >> config.properties &&
echo "node-scheduler.include-coordinator=true" >> config.properties &&
echo "http-server.http.port=8080" >> config.properties &&
echo "discovery.uri=http://localhost:8080" >> config.properties &&
echo "http-server.https.enabled=true" >> config.properties &&
echo "http-server.https.port=8443" >> config.properties &&
echo "http-server.https.keystore.path=/usr/local/trino/etc/trinosandbox.jks" >> config.properties &&
echo "http-server.https.keystore.key=trinosandboxkey" >> config.properties &&
echo "http-server.authentication.type=PASSWORD,CERTIFICATE" >> config.properties &&
echo "internal-communication.shared-secret=<>" >> config.properties
```

```
touch jvm.config &&
echo "-server" >> jvm.config &&
echo "-Xmx1G" >> jvm.config &&
echo "-XX:+UseG1GC" >> jvm.config &&
echo "-XX:G1HeapRegionSize=32M" >> jvm.config &&
echo "-XX:+UseGCOverheadLimit" >> jvm.config &&
echo "-XX:+ExplicitGCInvokesConcurrent" >> jvm.config &&
echo "-XX:+HeapDumpOnOutOfMemoryError" >> jvm.config &&
echo "-XX:+ExitOnOutOfMemoryError" >> jvm.config &&
echo "-Djdk.attach.allowAttachSelf=true" >> jvm.config
```

```
touch log.properties &&
echo "io.trino=INFO" >> log.properties
```

From your host machine terminal look up the IP address for the postgres Docker container with

```
docker inspect bridge
```

Return to the trinosandbox terminal and insert the following commands

```
echo "Please enter the IP Address of the Postgres container: " && read myvar && export POSTGRES_IP_ADDRESS="$myvar"
```

Create the catalog directory and an entry for Postgres

```
mkdir catalog &&
touch catalog/postgresqldb.properties &&
echo "connector.name=postgresql" >> catalog/postgresqldb.properties &&
echo "connection-url=jdbc:postgresql://$POSTGRES_IP_ADDRESS:5432/store" >> catalog/postgresqldb.properties &&
echo "connection-user=postgres" >> catalog/postgresqldb.properties &&
echo "connection-password=postgrespw" >> catalog/postgresqldb.properties &&
echo "case-insensitive-name-matching=true" >> catalog/postgresqldb.properties
```

From your host machine terminal look up the IP address for the mysql Docker container again. Feel free to run the docker command to inspect the bridge again

```
docker inspect bridge
```

Return to the trinosandbox terminal and insert the following commands

```
echo "Please enter the IP Address of the MySQL container: " && read myvar && export MYSQL_IP_ADDRESS="$myvar"
```

Create the catalog entry for MySQL

```
touch catalog/mysqldb.properties &&
echo "connector.name=mysql" >> catalog/mysqldb.properties &&
echo "connection-url=jdbc:mysql://$MYSQL_IP_ADDRESS:3306" >> catalog/mysqldb.properties &&
echo "connection-user=root" >> catalog/mysqldb.properties &&
echo "connection-password=mysqlpw" >> catalog/mysqldb.properties &&
echo "case-insensitive-name-matching=true" >> catalog/mysqldb.properties
```

Run the following commands to generate a self-signed certificate

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout trinosandbox.key -out trinosandbox.crt -subj "/CN=localhost" &&
openssl pkcs12 -export -in trinosandbox.crt -inkey trinosandbox.key -out trinosandbox.p12 -name trinosandbox -passout pass:trinosandboxkey &&
keytool -importkeystore -deststorepass trinosandboxkey -destkeystore trinosandbox.jks -srckeystore trinosandbox.p12 -srcstoretype PKCS12 -srcstorepass trinosandboxkey -alias trinosandbox
```

Launch the Trino server

```
launcher start
```

Verify that the Trino server is running (You may want to run it again after a few seconds to be sure the Trino server is still running)

```
launcher status
```

Verify that server is running with SSL

```
curl -v https://localhost:8443 --insecure
```

The first few lines should look like the following

- Connected to localhost (::1) port 8443 (#0)
- ALPN, offering h2
- ALPN, offering http/1.1
- CAfile: /etc/pki/tls/certs/ca-bundle.crt

Start Trino client using the 'testuser' password

```
./trino --server https://localhost:8443 --user testuser --password --keystore-path /usr/local/trino/etc/trinosandbox.jks --keystore-password trinosandboxkey
```

### **You can try the following queries to verify every has been setup correctly**

Run a query against the MySQL database

```
SELECT \* FROM mysqldb.customer.customer WHERE City LIKE '%H%';
```

Run a query against the Postgres database

```
SELECT customerId, item, total
FROM postgresqldb.public.purchase
WHERE item LIKE 'Binder';
```

Run a query that joins the Customer table from the MySQL database with the Purchase table in the PostgreSQL database ON their respective CustomerID fields where the MySQL LastName is 'Black'

```
SELECT
  mysql.customerId AS MySQL_customerId,
  mysql.FirstName AS MySQL_FirstName,
  mysql.LastName AS MySQL_LastName,
  postgresql.item as PostgreSQL_item,
  postgresql.total as PostgreSQL_total
FROM mysqldb.customer.customer AS mysql
INNER JOIN postgresqldb.public.purchase AS postgresql
ON mysql.CustomerId = postgresql.CustomerID
WHERE mysql.LastName = 'Black';
```

Quit the Trino CLI

```
quit
```

Exit the Trino Docker container

```
exit
```

### Try out the WebUI login

https://localhost:8443
U: testuser
P: trinopw

## (Optional) DBeaver Setup on host machine

Run the following commands from the host terminal

```
docker cp trinosandbox:/usr/local/trino/etc/trinosandbox.jks .
```

```
mkdir ~/certs
```

```
mv trinosandbox.jks ~/certs
```

1. Create a new connection in DBeaver
2. Select Trino from the list of connections (You might have run a search for it)
3. Set the following in the "Main" tab
   a. Host: localhost
   b. Port: 8443
   c. Username: testuser
   d. Password: trinopw
4. Select the "Driver properties tab
5. Set the following properties
   a. SSL: true
   b. SSLKeyStorePath: /path/to/trino-localhost.jks
   c. SSLKeyStorePassword: trinosandboxkey
   d. SSLVerification: CA

### You can try the following queries again in DBeaver by right clicking on the trino connection to access the 'SQL Editor' so you can start a 'New SQL Script'

Run a query against the MySQL database

```
SELECT \* FROM mysqldb.customer.customer WHERE City LIKE '%H%';
```

Run a query against the Postgres database

```
SELECT customerId, item, total
FROM postgresqldb.public.purchase
WHERE item LIKE 'Binder';
```

Run a query that joins the Customer table from the MySQL database with the Purchase table in the PostgreSQL database ON their respective CustomerID fields where the MySQL LastName is 'Black'

```
SELECT
  mysql.customerId AS MySQL_customerId,
  mysql.FirstName AS MySQL_FirstName,
  mysql.LastName AS MySQL_LastName,
  postgresql.item as PostgreSQL_item,
  postgresql.total as PostgreSQL_total
FROM mysqldb.customer.customer AS mysql
INNER JOIN postgresqldb.public.purchase AS postgresql
ON mysql.CustomerId = postgresql.CustomerID
WHERE mysql.LastName = 'Black';
```
