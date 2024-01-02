## Bash Script Automation
- [Bash Script Automation](#bash-script-automation)
- [Note:](#note)
- [Goals:](#goals)
- [Diagnostic Tools:](#diagnostic-tools)
- [Manually Setting the Database Virtual Machine](#manually-setting-the-database-virtual-machine)
  - [Installing MySQL Server](#installing-mysql-server)
  - [Creating an Empty Database](#creating-an-empty-database)
  - [Loading the Database](#loading-the-database)
  - [Creating a new user](#creating-a-new-user)
  - [Changing the Binding IP](#changing-the-binding-ip)
  - [Database prov.sh (and app data)](#database-provsh-and-app-data)
- [Manually Setting the Application Virtual Machine](#manually-setting-the-application-virtual-machine)
  - [Installing MySQLClient, Maven, JDK 17 and app code](#installing-mysqlclient-maven-jdk-17-and-app-code)
  - [Setting environment variables](#setting-environment-variables)
  - [Starting our Spring Application](#starting-our-spring-application)
  - [Setting up our Reverse Proxy](#setting-up-our-reverse-proxy)
  - [Application prov.sh (and app data)](#application-provsh-and-app-data)

## Note:
If you are only interested in the App Data script for a quickstart, simply head to the Database prov.sh heading.

## Goals:
We would like to create a two-tier deployment in two separate VMs - one containing our MySQL Database, and one containing our app code. The application VM should also contain a reverse proxy for the default HTML port to map to map to port 5000. It is also important to note that no matter what level of automation, we will need to run our database virtual machine first to obtain a target IP address for the SQL connection before running our Application Virtual Machine.

## Diagnostic Tools:
We will often encounter problems when trying to automate the deployment of things. A critical aspect of this is the ability to diagnose errors for each problem and so this document will also highlight potential problems and how you can check them.

* **Database creation problems** - using MySQL shell followed by certain SQL commands such as ```SHOW DATABASES;``` or ```USE world; SHOW TABLES;``` can give particular insight on what the problem really is.
  
* **Database connection problems** - if we try and jump the gun and run the Spring application, there will be build failures that are not particularly helpful. Often it will be useful to try and establish a preliminary connection to the VM or the database first. ```netstat``` can be used to determine connection to a virtual machine, while ```mysql -h 3.214.54.25 -P 3306 -u root -p``` can be used to perform the aforementioned commands to determine connectivity.
  
* **Environment variable problems** - we may enc

## Manually Setting the Database Virtual Machine
We can identify the steps needed for our Database virtual machine as follows:
* Install MySQL
* Setup database
* Setup database connection

### Installing MySQL Server
To manually host an SQL database, we will need a MySQL package that is able to handle some SQL queries as well as adjusting IP addresses that it will accept requests from.
```
# Install MySQL
sudo DEBIAN_FRONTEND=noninteractive apt install mysql-server -y
```

### Creating an Empty Database
Next we will need to prepare an empty database to store our SQL data in. Here we will cover an instance of creating a database called world.
```
# Create new database
sudo mysql -u root -p'root' -h localhost -P 3306 -e "CREATE DATABASE world;"
```

### Loading the Database
After cloning the appropriate repository from Git, we will then need to navigate to the correct directory and 'unpack' some SQL into our new database. This can be performed using the following commands.

```
sudo git clone https://github.com/Affiq/World_SQL_Database.git repo
cd /repo
sudo mysql -u root -p'root' -h localhost -P 3306 -D world < world.sql
```

### Creating a new user
We will also need to prepare a new MySQL user for our database. Currently, there are only users compatible with localhost access and not with access outside the VM. Here we create a user root with password root, using a % wildcard to allow any IP address to use the allocated user. As well as being given all privileges to the MySQL database (which can be potentially bad practice but will be shown for testing purposes) before applying the ```FLUSH PRIVILEGES``` command to apply the permissions immediately.

```
sudo mysql -e "CREATE USER 'root'@'%' IDENTIFIED BY 'root'; 
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'; 
FLUSH PRIVILEGES;"
```

### Changing the Binding IP
By default, MySQL will only allow database connections to local users only. To enable this, we will need to edit the /etc/mysql/mysql.conf.d/mysql.cnf file which typically looks like this.

```
[mysqld]
bind-address = 127.0.0.1
```
to
```
[mysqld]
bind-address = 0.0.0.0
```
to allow all connections. Of course this can then be readjusted later to a specific IP address. The following sed command allows us to achieve the desired result.

```
# Enable remote connections
sudo sed -i 's/^bind-address\s*=\s*127\.0\.0\.1/bind-address = 0.0\.0\.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
```

### Database prov.sh (and app data)
Once we have tested all the manual commands, we can then compile this into one BASH script. Once we have executed this BASH script from within a fresh virtual machine, we can then experiment with User Data to ensure that the application is 'bullet-proof'.

```
#!/bin/bash

# Update
echo "Updating"
sudo apt update -y
echo "Updating: Done"
echo ""

# Upgrade
echo "Upgrading"
sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y
echo "Upgrading: Done"
echo ""

# Install MySQL
echo "Installing MySQL Server"
sudo DEBIAN_FRONTEND=noninteractive apt install mysql-server -y
echo "Installation: Done"
echo ""

# Start MYSQL on boot
echo "Enabling MySQL"
sudo systemctl enable mysql
echo "Enabling: Done"
echo ""

# Enable remote connections
echo "Opening connections"
sudo sed -i 's/^bind-address\s*=\s*127\.0\.0\.1/bind-address = 0.0\.0\.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
echo "Connection setup: Done"
echo ""

# Restart MySQL
echo "Restarting MySQL"
sudo systemctl restart mysql.service
echo "Restarting: Done"
echo ""

# Downloading app code
echo "Downloading app code"
sudo git clone https://github.com/Affiq/World_SQL_Database.git repo
echo "Downloading: Done"
echo ""

# Create new user for VM to connect to
echo "Creating new MySQL User"
sudo mysql -e "CREATE USER 'root'@'%' IDENTIFIED BY 'root'; GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'; FLUSH PRIVILEGES;"
echo "User Creation: Done"
echo ""

# Create new database
echo "Creating database"
sudo mysql -u root -p'root' -h localhost -P 3306 -e "CREATE DATABASE world;"
echo "Database creation done"
echo ""

# Extract SQL data
echo "Extracting MySQL file"
cd /repo
sudo mysql -u root -p'root' -h localhost -P 3306 -D world < world.sql
echo "Extraction: Done"
echo ""

# Show Databases
echo "Showing Databases..."
sudo mysql -e "SHOW DATABASES;"
echo ""
```

## Manually Setting the Application Virtual Machine
We will now need to configure the application virtual machine for our project.

### Installing MySQLClient, Maven, JDK 17 and app code
We will first need to install the relevant packages for our application to function - this would include Maven and JDK 17 for our standard Spring project, but we will also need a MySQL client to connect and communicate with the database virtual machine.

```
echo "Installing MySQL client"
sudo DEBIAN_FRONTEND=noninteractive apt install mysql-client -y
echo "Installation: Done"
echo ""

# Install Maven
echo "Installing Maven"
sudo DEBIAN_FRONTEND=noninteractive apt install maven -y
echo "Installation: Done"
echo ""

# Install JDK17
echo "Installing JDK17"
sudo DEBIAN_FRONTEND=noninteractive apt install openjdk-17-jdk -y
echo "Installation: Done"
echo ""

```

### Setting environment variables
Our Spring project's application.properties file makes use of environment variables - we will need to configure these in our environment - more specifically the Ubuntu's env variables (this might differ for the app data). We can perform this with the following commands. Change the IP address to the your database VM's private IP.

```
export DB_HOST=jdbc:mysql://<db-private-ip>:3306/world
export DB_PASS=root
export DB_USER=root
```

### Starting our Spring Application
We will need to setup our application much like before. However, some build errors are caused and require the application recompile, as well as requiring the ubuntu's environment variables. When using a sudo command, this will need to be passed using a -E flag to use the current users environment variables rather than superuser's environment variables.

```
cd /repo
sudo -E mvn package spring-boot:start
```

### Setting up our Reverse Proxy
Considering the similarities for setting up our project with our previous project using Apache2 reverse proxy, we can simply utilise the exact same code (command by command) to set up the reverse proxy.

### Application prov.sh (and app data)
Combining with our final shell will then give us the following BASH script. We can test this in a BASH Script before being confident moving it into the User Data stage.

```
#!/bin/bash

# Update
echo "Updating"
sudo apt update -y
echo "Updating: Done"
echo ""

# Upgrade
echo "Upgrading"
sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y
echo "Upgrading: Done"
echo ""

echo "Installing MySQL client"
sudo DEBIAN_FRONTEND=noninteractive apt install mysql-client -y
echo "Installation: Done"
echo ""

# Start MYSQL on boot
echo "Enabling MySQL"
sudo systemctl enable mysql
echo "Enabling: Done"
echo ""

# Install Maven
echo "Installing Maven"
sudo DEBIAN_FRONTEND=noninteractive apt install maven -y
echo "Installation: Done"
echo ""

# Install JDK17
echo "Installing JDK17"
sudo DEBIAN_FRONTEND=noninteractive apt install openjdk-17-jdk -y
echo "Installation: Done"
echo ""

# Cloning repository
echo "Cloning app code"
sudo git clone https://github.com/Affiq/World_Project_VM.git repo
echo "Cloning: Done"
echo ""

# Set env variables
echo "Setting environment variables"
cd /home/ubuntu
export DB_HOST=jdbc:mysql://<db-private-ip>:3306/world
export DB_PASS=root
export DB_USER=root
echo "Env variables set"
echo ""


# Starting Spring-Boot server
echo "Starting Spring-Boot Server"
cd /repo
sudo -E mvn package spring-boot:start
echo "Server: Done"
echo ""

# OUR NEW CODE
# Nav back to root
cd /

# Install Apache2
sudo DEBIAN_FRONTEND=noninteractive apt install apache2 -y

# Enable relevant apache proxy modules
sudo a2enmod proxy
sudo a2enmod proxy_http

# Restart Apache 
sudo systemctl restart apache2 

# Edit config file if not configured
if grep -q 'ProxyPass / http://localhost:5000/' /etc/apache2/sites-available/000-default.conf; then
    # The string exists, so nothing to do
    echo "Reverse proxy already configured."
else
    # reverse proxy not configured yet
    echo "Reverse proxy NOT configured."
    sudo sed -i '/DocumentRoot \/var\/www\/html/ a\ProxyPreserveHost On\nProxyPass \/ http:\/\/localhost:5000\/\nProxyPassReverse \/ http:\/\/localhost:5000\/\n' /etc/apache2/sites-available/000-default.conf
fi

# Restart Apache
sudo systemctl reload apache2
```

