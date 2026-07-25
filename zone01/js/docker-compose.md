# docker-compose

a docker tool that allows us to build entire dockerized worlds with one file

installation:
sudo apt update
sudo apt install docker.io docker-compose -y

whenever working with docker compose, you need to make a folder dedicated to that one composed thing you're doing !

default config filename: docker-compose.yaml (yaml is a data serialization language -> easy for config files)

example: (substituting a command with a compose file)

```sh
sudo docker run --name web -itd -p 8080:80 nginx
```

```yaml
version: "3" # docker version
services: # place to define containers
    website: # container name
        image: nginx
        ports:
            - "8081:80"
        restart: always # so it always backs up after the machine reboots
```

* run command inside same folder, or you'll need to specify the path with -f
```sh
sudo docker-compose up -d
```
it creates (default behavior):
    - a network: folder-name_default (a bridge network)
    - a container: folder-name_container-name_1 (it assumes you'll add more containers inside that network)

* Show running containers in that network
```sh
sudo docker-compose ps
```

* Reverts everything:
```sh
sudo docker-compose down # stops & removes all containers and networks
```

* Stops containers:
```sh
sudo docker-compose stop
```


to change that behavior:
to add a new container, just add it inside services with a new name

to add a network: (substituting a command with a section in docker compose file)
```sh
sudo docker network create coffee --subnet 192.168.92.0/24
# (learn docker networking)
```

```yaml
# add a new section 'networks' inline with 'services':
networks:
    coffee: # network name
        ipam: # ip address management
            driver: default # network driver (default network in docker is the bridge)
            config:
                - subnet: "192.168.92.0/24"
```
and to add a container to that network:
```yaml
# ...
    website:
        # ...
        networks:
            coffee:
                ipv4_address: 192.168.92.21
```

if the compose file up and running, and we changed the file and re-run it once more, it only updates what we changed !

* to list networks:
```sh
sudo docker network ls
```

and if we inspect our network:
```sh
sudo docker inspect coffetime_coffee
```
we will see the list of containers

### example (running a wordpress project -> frontend + database)

```yaml
version: "3"
services:
    wordpress:
        image: wordpress # from docker hub 
        ports:
            - "8080:80"
        depends_on: # to make sure it runs after making the follwing containers
            - mysql
        environment: # specific to wordpress
            WORDPRESS_DB_HOST: mysql
            WORDPRESS_DB_USER: root
            WORDPRESS_DB_PASSWORD: "coffee"
            WORDPRESS_DB_NAME: wordpress
        networks:
            ohyeah:
                ipv4_address: 10.56.1.21
    mysql:
        image: "mysql:5.7"
        environment:
            MYSQL_DATABASE: wordpress
            MYSQL_ROOT_PASSWORD: "coffee"
        volumes:
            # docker volumes allow us to map a docker directory to a directory on our -host- system
            # if the docker system gets blown up, dies, restart it, or delete it, or whatever -> that directory - the data - still exists on the host which is kind of important for a database !
            ./mysql:/var/lib/mysql
            # and you'll also find the mysql folder inside the same directory !
            # and if you do: "sudo docker-compose down", the data is preserved in that folder !
        networks:
            ohyeah:
                ipv4_address: 10.56.1.20
networks:
    ohyeah:
        ipam:
            driver: default
            config:
                - subnet: "10.56.1.0/24"
```

### More

* https://github.com/vulhub/vulhub

pre-built hacking labs as docker-compose files

* https://github.com/docker/awesome-compose

docker compose inspiration (from professional projects)