Mattermost-LDAP - Docker module
===============================

## Summary

This repository provides necessary ressources to build the Docker image for Mattermost-LDAP module. This Docker image is usefull to try Mattermost-LDAP in a PoC, and is production ready.

## Description

The Mattermost-LDAP module is divided into two Docker images. On the one hand, the PostgreSQL database and on the other hand, the httpd server.

The PostgreSQL image installs a PostgreSQL database and then configures the Oauth user and the Oauth server database with associated tables. The Mattermost client ID, with its associated secret ID, and the Mattermost Redirect URI are added to the oauth_clients table to allow Mattermost to use the Oauth server.
The configuration of the database is done by the init.sh script whose parameters are gathered in config_init. These two files are located in the `postgres/files/` folder in this reporsitory.

The httpd server image is based on a CentOS 7 image on which an httpd server and the necessary dependencies are installed with yum. The Oauth server is configured from the Mattermost-LDAP project available on Github. LDAP and database configuration are provided by config_db.php and config_ldap.php in the `ouath/files/` folder in this repository.

## Architecture

![Docker Architecture of Mattermost-LDAP and interraction with Mattermost](https://github.com/Crivaledaz/Mattermost-LDAP/blob/master/Docker/mattermostldap-docker.png)

The Oauth container exposes port 80 and Postgres container port 5432. The user interacts with the Oauth server and the tokens generated by it are stored in the database. In addition, when a user logs in, his ID is stored with a unique ID. This behavior is necessary for authentication with Mattermost. The figure above illustrates interraction between Oauth server, Postgres database and Mattermost. 

## Image Build

Firstly, install `docker-ce` on your host server :
- For CentOS/RHEL : https://docs.docker.com/install/linux/docker-ce/centos/
- For Fedora : https://docs.docker.com/install/linux/docker-ce/fedora/
- For Debian : https://docs.docker.com/install/linux/docker-ce/debian/
- For Ubuntu : https://docs.docker.com/install/linux/docker-ce/ubuntu/


Then, clone this repository on your host and go in `Docker` directory :
```
git clone 
cd Mattermost-LDAP/Docker
``` 

There are two Dockerfiles, one in `postgres/` and another in `oauth/`. These Dockerfiles must be compiled to create the corresponding images. To do this, we use the docker build command as follows:
```
docker build -t mattermostldap-postgres:latest postgres/
docker build -t mattermostldap-oauth:latest oauth/
```
Once built, images are available in the Docker daemon and be used to create container with `docker run`. To view available images you can use : 
```
docker images list
```

## Configuration

Some image parameters can be changed, by specifying the desired parameters in container's environment variable, when you create a container to adapt it to your configuration. To apply custom parameters, they must be added to the container execution line with the --env (or -e) option followed by the parameter name and the desired value (-e <parameter> = <value> ). For more details, refer to the examples in the Usage section.

### LDAP
| Parameter             | Description                                                           | Default value            |
|-----------------------|-----------------------------------------------------------------------|--------------------------|
| ldap_host             | URL or IP to connect LDAP server                                      | ldap://ldap.company.com/ |
| ldap_port             | Port used to connect LDAP server                                      | 389                      |
| ldap_version          | LDAP version or protocol version used by LDAP server                  | 3                        |
| ldap_search_attribute | Attribute used to identify a user on the LDAP                 		| uid                      |
| ldap_filter           | Additional filter for LDAP search                         			| objectClass=*            |
| ldap_base_dn          | The base directory name of your LDAP server                           | ou=People,o=Company      |
| ldap_bind_dn          | The LDAP Directory Name of an service account to allow LDAP search  	|                          |
| ldap_bind_pass        | The password associated to the service account to allow LDAP search   |                          |


### Database
| Parameter  | Description                                                          | Default value      |
|------------|----------------------------------------------------------------------|--------------------|
| db_host    | Hostname or IP address of the Postgres container (database)          | 127.0.0.1          |
| db_port    | The port of your database to connect                                 | 5432               |
| db_type    | Database type to adapt PDO. Should be pgsql for Postgres container   | pgsql              |
| db_user    | User who manages oauth database                                      | oauth              |
| db_pass    | User's password to manage oauth database                 	    | oauth_secure-pass  |
| db_name    | Database name for oauth server                             	    | oauth_db           |


### Client
| Parameter       | Description                                                        | Default value                                   |
|-----------------|--------------------------------------------------------------------|------------------------------------------------------|
| client_id       | Token client ID shared with mattermost                             | 123456789                                            |
| client_secret   | Token client Secret shared with mattermost                         | 987654321                                            |
| redirect_uri    | The callback address where oauth will send tokens to Mattermost    | http://mattermost.company.com/signup/gitlab/complete |
| grant_types     | The type of authentification use by Mattermost                     | authorization_code                                   |
| scope           | The scope of authentification use by Mattermost                    | api                                                  |
| user_id         | The username of the user who create the Mattermost client in Oauth |                                                      |


## Usage

Both containers can be run separately, but the Mattermost-LDAP module requires both containers are working and communicating to be operational.

### Container mattermostldap-postgres

Once built, the mattermostldap-postgres image can be used to start a container running the postgresql database for the Mattermost-LDAP module. The image contains a default configuration specified in the configuration section. To run a container from the mattermostldap-postgres image :

```
docker run -d mattermostldap-postgres --name database
```

Image settings can be customized by passing the desired values per environment variable to the container. For example, to configure the client ID and secret ID, start the container with the following command:
```
docker run -d mattermostldap-postgres --name database -e client_id=123456789 -e client_secret=987654321
```

For more information about available parameters, refer to the configuration section or the [Mattermost-LDAP documentation](https://github.com/Crivaledaz/Mattermost-LDAP/blob/master/README.md). 

In addition, the mattermostldap-postgres container stores database entries in a volume outside the container to allow persistence of data beyond the life of the container. By default, Docker automatically creates a volume and stores the data in the postgresql database. However, the volume is destroyed as soon as no object references it. To overcome this problem, it is advisable to save the container data outside Docker, specifying the path of the folder that will be used for storage.  To bind the container to the folder chosen, you can use this command:
```
docker run -d mattermostldap-postgres --name database --volume /data/mattermostldap-postgres:/var/lib/postgresql/data
```

To delete the database container, you can use :
```
docker rm database
```

## Container mattermostldap-oauth

Once built, the mattermostldap-oauth image can be used to build a container running the oauth server of the Mattermost-LDAP module. The image contains a default configuration specified in the configuration section. To run a container from the mattermostldap-oauth image:
```
docker run -d mattermostldap-oauth --name oauth
```

To adapt the parameters of the image, youjust need to specify custom parameters in environment variables of the container at its start. For example, to configure the LDAP server, use the following command:
```
docker run -d mattermostldap-oauth --name oauth -e ldap_host="ldap.company.com" -e ldap_port=389
```

To delete the oauth container, you can use :
```
docker rm oauth
```

## Improvement

In order to allow a dynamic configuration of the mattermostldap-oauth and mattermostldap-postgres images, the choice has been made to pass the parameters by environmental variables to the container. However, this method exposes all user-defined settings to all processes in the container. As a result, passwords and security tokens are exposed throughout the container and can easily be recovered by any process running in the container, including a user shell.

Unfortunately, this is the simplest method to avoid defining static parameters in the image, forcing a recompilation of the image each time a value is changed. While waiting for a more secure solution, it is highly recommended to secure access to the container.
