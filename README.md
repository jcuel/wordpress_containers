# wordpress_containers
## Containers with Wordpress, MariaDB, and using docker volumes and networking.
##################################
#
##		Idea:
#		- *Networking Isolation*
#		- *Multiple containers (services) working together*
#		- *Volumes*
#		
#		1) Create Private Network Bridge
#		2) Create MariaDB Container
#		3) Create Adminer Container with 2 networks (bridge & PRIVATE_SUBNET) for db management
#		4) Install wordpress container

##1)  Create Private Network Bridge
### Create the Network Bridge
```
root@containerhost:/home/instructor/projects/wordpress_containers# docker network create --driver bridge PRIVATE_SUBNET
ee2ac45cec03fc2d3b7b460e40e4d61f183f5e4a4b5fe9bbd6b547f5884e782d
root@containerhost:/home/instructor/projects/wordpress_containers# docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
ee2ac45cec03   PRIVATE_SUBNET   bridge    local
a705938ba61c   bridge           bridge    local
171e524f7556   host             host      local
9ff60fa89f6d   none             null      local

```
### Validate it's configuration

```
root@containerhost:/home/instructor/projects/wordpress_containers# docker network inspect PRIVATE_SUBNET
[
    {
        "Name": "PRIVATE_SUBNET",
        "Id": "ee2ac45cec03fc2d3b7b460e40e4d61f183f5e4a4b5fe9bbd6b547f5884e782d",
        "Created": "2021-02-23T21:34:22.720481632Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "9af546348c036458bd2913e7df5d334e73c9ad24499cbb4b8499bb0faa1cb9a1": {
                "Name": "wordpressdb",
                "EndpointID": "378025052da6c62865a916f7bbc116ddf6d107dc14fcd3ae1b773d4495ecc5ad",
                "MacAddress": "02:42:ac:15:00:02",
                "IPv4Address": "172.21.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

```
## 2) Create MariaDB Container
 
### a) pull mariadb image from docker source

```
root@containerhost:/home/instructor/projects/wordpress_containers# docker pull mariadb:latest
latest: Pulling from library/mariadb
83ee3a23efb7: Pull complete
db98fc6f11f0: Pull complete
f611acd52c6c: Pull complete
aa2333e25466: Pull complete
f53ac4b825fd: Pull complete
c20afcf9b055: Pull complete
54c5dc6dcf19: Pull complete
b1c71d744483: Pull complete
863a8cc01d1c: Pull complete
ea6c59f9e205: Pull complete
6aa441240c22: Pull complete
c1fee6e1dead: Pull complete
Digest: sha256:a13b01d9af44097efeca7a3808a524c8052870e59a1eeef074857ba01e70c1f2
Status: Downloaded newer image for mariadb:latest
docker.io/library/mariadb:latest

```

### b) Create mariadb container & validate

```
root@containerhost:/home/instructor/projects/wordpress_containers# docker run -e MYSQL_ROOT_PASSWORD=root-password -e MYSQL_USER=wpuser -e MYSQL_PASSWORD=password -e MYSQL_DATABASE=wpdb -v /home/instructor/projects/wordpress_containers/database:/var/lib/mysql --name wordpressdb -d mariadb
93d29fa6a29826fd1d86d6d93da33e5e5a9ef54482800b564e5f4495e3a03fcc
root@containerhost:/home/instructor/projects/wordpress_containers# docker ps
ONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS      NAMES
9af546348c03   mariadb   "docker-entrypoint.s…"   20 seconds ago   Up 19 seconds   3306/tcp   wordpressdb
```	
### *Definitions:*
```
- *MYSQL_ROOT_PASSWORD: This option will configure MariaDB root password.
- *MYSQL_USER: This will create a new wpuser for WordPress.
- *MYSQL_PASSWORD: This will set the password for wpuser.
- *MYSQL_DATABASE: This will create a new database named wpdb for WordPress.
- *-v /home/instructor/projects/wordpress_containers/database:/var/lib/mysql: This will link the database directory to the mysql directory.

```	
### c) Add PRIVATE_SUBNET to the container and remove bridge subnet. 

```
root@containerhost:/home/instructor/projects/wordpress_containers# docker network disconnect bridge wordpressdb
root@containerhost:/home/instructor/projects/wordpress_containers#  docker network connect PRIVATE_SUBNET wordpressdb

```	
### d) Validate it has updated it's ip address to be under PRIVATE_SUBNET

```
root@containerhost:~# docker inspect  wordpressdb | grep IPAddress
"SecondaryIPAddresses": null,
"IPAddress": "",
"IPAddress": "172.21.0.2",

```
### e) Validate DB is running on mapped volume (host drive) This ensures db data is actually in an non-volatile storage so data remains even if container fails.		

```
root@containerhost:/home/instructor/projects/wordpress_containers/database# ls
aria_log.00000001  ib_buffer_pool  ib_logfile0  multi-master.info  performance_schema
aria_log_control   ibdata1         ibtmp1       mysql              wpdb

```	
## 3) Create Adminer Container with 2 networks (bridge & PRIVATE_SUBNET) db for management

*This command will not be used as --link is marked as legacy command*

```
	#docker run --link wordpressdb -p 8080:8080 adminer

```
### Procedure
```
root@containerhost:/home/instructor/projects/wordpress_containers# docker run -d --network PRIVATE_SUBNET -p 8080:8080 adminer
Unable to find image 'adminer:latest' locally
latest: Pulling from library/adminer
ba3557a56b15: Pull complete
4ccf1c569cdb: Pull complete
ec65deac30f6: Pull complete
0a32b6dca15d: Pull complete
c08e12ee6bd8: Pull complete
9606d3207760: Pull complete
7fa05ab621d4: Pull complete
8b3133c328f3: Pull complete
0b7052c04711: Pull complete
58750e9a4300: Pull complete
c0d6ad862f7b: Pull complete
72eaf755091d: Pull complete
70e1deae3eb5: Pull complete
8fd1b11c6a41: Pull complete
f2142adba370: Pull complete
Digest: sha256:a71089fb10f9b130acdaa61cc0b7ff698028534667c6e6a5494e2e5d945cd34a
Status: Downloaded newer image for adminer:latest
dcffb6bde658d8ea753f990bd7e3b1fc8e7cf0684d06f8dd20fc9b14a3fa110e

```	
### OOpps fogot to name the container, let's rename it.

```	
root@containerhost:/home/instructor/projects/wordpress_containers# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS              PORTS                    NAMES
dcffb6bde658   adminer   "entrypoint.sh docke…"   About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp   stupefied_ardinghelli
93d29fa6a298   mariadb   "docker-entrypoint.s…"   3 hours ago          Up 5 minutes        3306/tcp                 wordpressdb
root@containerhost:/home/instructor/projects/wordpress_containers# docker container rename dcffb6bde658 adminer
root@containerhost:/home/instructor/projects/wordpress_containers# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED             STATUS             PORTS                    NAMES
dcffb6bde658   adminer   "entrypoint.sh docke…"   About an hour ago   Up About an hour   0.0.0.0:8080->8080/tcp   adminer
93d29fa6a298   mariadb   "docker-entrypoint.s…"   4 hours ago         Up About an hour   3306/tcp                 wordpressdb

```	
### attach public switch to be able to access tool and validate

```
root@containerhost:/home/instructor/projects/wordpress_containers# docker network connect bridge adminer
root@containerhost:/home/instructor/projects/wordpress_containers# docker inspect  adminer | grep IPAddress
"SecondaryIPAddresses": null,
"IPAddress": "172.17.0.2",
"IPAddress": "172.21.0.3",
"IPAddress": "172.17.0.2",

```	
### Don't forget to open port 8080 on your host

```
root@containerhost:/home/instructor/projects/wordpress_containers# sudo ufw allow 8080
Rules updated
Rules updated (v6)

```
* Now we have an adminer instance running on our container accesible from db network and exteranl on 8080 for management *
	
## 4) Install wordpress container

```
root@containerhost:/home/instructor/projects/wordpress_containers/html# docker run -d -e WORDPRESS_DB_USER=wpuser -e WORDPRESS_DB_PASSWORD=password -e WORDPRESS_DB_NAME=wpdb -p 8081:80 -v /home/instructor/projects/wordpress_containers/html:/var/www/html --network PRIVATE_SUBNET --name wpcontainer -d wordpress
Unable to find image 'wordpress:latest' locally
latest: Pulling from library/wordpress
45b42c59be33: Pull complete
a48991d6909c: Pull complete
935e2abd2c2c: Pull complete
61ccf45ccdb9: Pull complete
27b5ac70765b: Pull complete
5638b69045ba: Pull complete
0fdaed064166: Pull complete
e932cec09ced: Pull complete
fbe190145b1c: Pull complete
f747612094ef: Pull complete
300f68c220b1: Pull complete
efd583fc4f80: Pull complete
011e53c9540e: Pull complete
90d05db0a960: Pull complete
5faae26e6219: Pull complete
7bf1209c35d8: Pull complete
527f0104274c: Pull complete
435b4e30e1cf: Pull complete
e3e9c13fe04e: Pull complete
f5bce9c529a8: Pull complete
Digest: sha256:583894369838ecd5fee59532059600b2fbd854b856ee45f8ef220397d8090c64
Status: Downloaded newer image for wordpress:latest
8e150b0c297d42f7db450cbb7db42d0059d17358965f38791a2c3f9e923ac43d

```
### let's attach public switch so we can navigate wordpress

```
root@containerhost:/home/instructor/projects/wordpress_containers/html# docker network connect bridge wpcontainer
root@containerhost:/home/instructor/projects/wordpress_containers/html# docker inspect wpcontainer | grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.3",
                    "IPAddress": "172.21.0.4",
                    "IPAddress": "172.17.0.3",

```
### ensure port 8081 is open

```
ufw allow 8081
root@containerhost:/home/instructor/projects/wordpress_containers/html# ufw allow 8081
Rules updated
Rules updated (v6)

```

*I forgot to define the ip address for our sql server under the wp deployment.* 
*Hence I would need to update it under /home/instructor/projects/wordpress_containers/html/wp-content.php*
*Now we can do what ever we want with it, and if crash, data is stored on our host, hence it would remain live and we justneed to redeploy and afterwards mv the data.*

	
