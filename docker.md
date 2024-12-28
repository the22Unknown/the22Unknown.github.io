# Docker Lab: Wordpress

For this lab I used my Arch Linux Virtual machine, as described in my other Github Page. I chose to do Wordpress as it seems to be a very important part of the internet and could be a useful setup to have around later.

## Installing Docker

```
pacman -Syu
pacman -S docker
pacman -S docker-compose
```

To install docker, I had to update pacman and then install both the `docker` and `docker-compose` packages.

## Setting up Wordpress

For this section, I followed [this tutorial](https://www.hostinger.com/tutorials/run-docker-wordpress). The most important thing this tutorial provided was the `docker-compose.yml` file for a simple Wordpress server. To create my wordpress container, I had to create the `docker-compose.yml` file with the information they provided, then run the following command. This file is contained at the end of this tutorial.

```
docker-compose up -d
```

This 'ran' the compose file and started my wordpress server. Once started, I was able to open firefox and navigate to the admin page, which looked as follows:

![wordpressAdminPage](https://user-images.githubusercontent.com/79318023/142014302-487fb42c-8c54-42d8-ae21-236fa2b1d761.jpg)

## Appendix: docker-compose.yml
```
version: '3.3'
services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress
   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
       WORDPRESS_DB_NAME: wordpress
volumes:
    db_data: {}
```
