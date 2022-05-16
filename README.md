# oracle_FDW
## Prerequisites: Must comply entirely before proceeding.
Centos 7 (latest with all packages updated) with 16GB 8Core 500GB Disk
Need root access at the installation (with sudo/as root)
Client machine with windows, pgadmin and putty installed.

Passwords keeping, need to keep each password in separate place, as it needs for future implementations.

Connectivity requirement (with internet only at installation later need to be removed for the sake of security)
Connectivity to theoracle remote db is a must and verified before proceeding.

## Installation Procedure.
### Install required packages
```
yum install mlocate wget bash-completion net-tools gcc readline-devel zlib-devel git unzip libaio -y 
```
### firewall permitions if any.
Execute following
```
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --reload
```
### Install postgres 12 from source.
```
wget https://ftp.postgresql.org/pub/source/v12.7/postgresql-12.7.tar.gz
tar -zxvf postgresql-12.7.tar.gz
cd postgresql-12.7/
./configure
make install
```
### To run postgres we need postgres user
#### replace 'your_password' with your needed password
```
useradd -p $(openssl passwd -1 your_password) postgres
```
### creating data directory & give ownership
```
mkdir /postgresql/data -p
chown postgres.postgres /postgresql/ -R
```
### Starting the service
```
su - postgres
/usr/local/pgsql/bin/initdb -D /postgresql/data
/usr/local/pgsql/bin/pg_ctl -D /postgresql/data -l logfile start

Observation
#You will get these following messages after second command.
#[postgres@postgres-srv01 ~]$ /usr/local/pgsql/bin/pg_ctl -D /postgresql/data -l logfile start
#waiting for server to start.... done
#server started
```
### Need to edit /postgresql/data/pg_hba.conf for permission
```
#Add these lines at the bottom
host    all             all             127.0.0.1/32            md5
host    all             all              0.0.0.0/0              md5

#Comment the following lines
# IPv4 local connections:
#host    all             all             127.0.0.1/32         trust
# IPv6 local connections:
#host    all             all             ::1/128              trust
```
### At bottom we need to add following, 
vi /postgresql/data/postgresql.conf
listen_addresses = '*'
### Make sure you are postgress user, verify with below command.
```
whoami
#You should see, 'postgres' 
```
### Change your directory, 
```
cd /home/postgres/
```
### Get Instant client from oracle by downloading below, 
```
wget https://download.oracle.com/otn_software/linux/instantclient/1911000/instantclient-basic-linux.x64-19.11.0.0.0dbru.zip
wget https://download.oracle.com/otn_software/linux/instantclient/1911000/instantclient-sqlplus-linux.x64-19.11.0.0.0dbru.zip
wget https://download.oracle.com/otn_software/linux/instantclient/1911000/instantclient-sdk-linux.x64-19.11.0.0.0dbru.zip
```
### Uncompress following by executing, 
```
unzip instantclient-basic-linux.x64-19.11.0.0.0dbru.zip
unzip instantclient-sqlplus-linux.x64-19.11.0.0.0dbru.zip
unzip instantclient-sdk-linux.x64-19.11.0.0.0dbru.zip

Observation
#You should see following, 
$ls
instantclient_19_11                                instantclient-sdk-linux.x64-19.11.0.0.0dbru.zip      logfile
instantclient-basic-linux.x64-19.11.0.0.0dbru.zip  instantclient-sqlplus-linux.x64-19.11.0.0.0dbru.zip
```
### make sure ownership is for postgres
```
chown postgres.postgres instantclient_19_11 -R
```
### Network configurations
```
$vi /home/postgres/instantclient_19_11/network/admin/tnsnames.ora

#Copy following
#Change the IP to mentioned IP from the DB Admin.
MAMBOTEST =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 172.30.11.20)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = mambotest)
    )
  )


$vi /home/postgres/instantclient_19_11/network/admin/sqlnet.ora
NAMES.DIRECTORY_PATH= (TNSNAMES, HOSTNAME)
NAMES.DEFAULT_DOMAIN = WORLD
TRACE_LEVEL_CLIENT = OFF
SQLNET.EXPIRE_TIME = 30
```
### Edit postgres home profile
```
vi /home/postgres/.bash_profile

#Comment the lines which is not needed as below, 
#PATH=$PATH:$HOME/.local/bin:$HOME/bin

#At bottom, 
#add following
#User specific environment and startup programs

#PATH=$PATH:$HOME/.local/bin:$HOME/bin
PATH=$PATH:$HOME/.local/bin:$HOME/bin:$HOME/instantclient_19_11:/usr/local/pgsql/bin

export PATH
export ORACLE_HOME=~/instantclient_19_11
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME:$ORACLE_HOME/sdk/include
export TNS_ADMIN=$ORACLE_HOME/network/admin
export PGDATA=/postgresql/data

export PATH
```
### Now update the profile by following
```
source ~/.bash_profile
```
### Start following instant client started normally. If the installation started correctly the below will show, 
```
sqlplus /nolog

Observations
#Output will be as follows
SQL*Plus: Release 19.0.0.0.0 - Production on Wed Aug 4 14:57:34 2021
Version 19.11.0.0.0

Copyright (c) 1982, 2020, Oracle.  All rights reserved.

SQL> 

quit;
#Use quit; to exit
```
### Test instant client installation correct by following, 
```
sqlplus system/manager@ORATARGET

Observation
[postgres@postgres-srv01 ~]$ sqlplus system/manager@ORATARGET

SQL*Plus: Release 19.0.0.0.0 - Production on Wed Aug 4 14:59:32 2021
Version 19.11.0.0.0

Copyright (c) 1982, 2020, Oracle.  All rights reserved.

ERROR:
ORA-12154: TNS:could not resolve the connect identifier specified


Enter user-name:


# Give a ctrl + c and an enter for exit the process
```
### Logout to root user by command
```
logout

Observations
#[postgres@postgres-srv01 ~]$ logout
#[root@postgres-srv01 ~]#
```
### Change ownership from root to postgres
```
chown postgres.postgres /usr/local/pgsql -R
```
### Change user back to postgres
```
su - postgres
```

### Now execute following for installing oracle_fdw
```
git clone https://github.com/laurenz/oracle_fdw.git
cd oracle_fdw/
make
make install
```
## Database 
##################################
#Database part
##################################
#In your client Windows 10 machine, please install pgadmin (if not installed earlier)

### Restart the postgress sql,
```
#Stop
$/usr/local/pgsql/bin/pg_ctl -D /postgresql/data -l logfile stop

Observation
[postgres@postgres-srv01 ~]$ /usr/local/pgsql/bin/pg_ctl -D /postgresql/data -l logfile stop
waiting for server to shut down.... done
server stopped

#Start
$/usr/local/pgsql/bin/pg_ctl -D /postgresql/data -l logfile start

Observation
[postgres@postgres-srv01 ~]$ /usr/local/pgsql/bin/pg_ctl -D /postgresql/data -l logfile start
waiting for server to start.... done
server started
```

### Now set a password for super user,
#Note down password(each one we used should kept in confidential. This has to be collected from the original oracle machine where we are planning to replicate).
```
$psql
[postgres@postgres-srv01 ~]$ psql
psql (12.7)
Type "help" for help.
postgres=#  \password
Enter new password:<Provide intended password>      
Enter it again: <Provide intended password>
postgres=# \q
type \q to exit
```
#Now go to the pgadmin application Windows(which should be installed earlier)
#In PGADMIN in servers right click create > server now add 

