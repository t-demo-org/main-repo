# POC ReadME
## Infrastruture
* 2 Ubuntu20.04LTS VM on Linode
* Github Organization
## Server setup
### Common
#### Common tools
```sh
sudo apt update
sudo apt install python3-pip -y
sudo apt install jq net-tools -y
pip3 install yq
```
#### Docker Install
```sh
## Install docker from official site
curl -fSsL https://get.docker.com | sudo sh - 
sudo apt install docker-compose -y
## Disable IPtables integration for docker since ufw is used.
echo '{ "iptables" : false }' | sudo tee /etc/docker/daemon.json 
sudo systemctl restart docker
## Add ubuntu user to docker group to use docker-cli without sudo
sudo usermod -aG docker ubuntu 
```
#### Github Runner Install
```sh
## Create Runner Dir
mkdir actions-runner && cd actions-runner
## Download the latest runner package
curl -o actions-runner-linux-x64-2.294.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.294.0/actions-runner-linux-x64-2.294.0.tar.gz
## Optional: Validate the hash
echo "a19a09f4eda5716e5d48ba86b6b78fc014880c5619b9dba4a059eaf65e131780  actions-runner-linux-x64-2.294.0.tar.gz" | shasum -a 256 -c
## Extract the installer
tar xzf ./actions-runner-linux-x64-2.294.0.tar.gz
## Configure Org Runner with the correct tagging
./config.sh --url https://github.com/t-demo-org --token $TOKEN
## Install Runner as a service
./svc.sh install 
## Start the Runner service
./svc.sh start
```
### Instance-1
#### Nginx Setup & Certbot
```sh
## Install Nginx
sudo apt install nginx -y
## Setup Nginx site
echo  "
server {
    server_name t-siren-demo.duckdns.org;
    listen 80;
    location / {
        root /home/ubuntu/frontend;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
    location ~* /api/(.*)$ {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
        proxy_pass http://194.233.166.173:8080/api/$1$is_args$args;  # CSG_SUBSCRIBER
        }
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /home/ubuntu/frontend;
    }
}
"
| sudo tee /etc/nginx/sites-enabled/t-siren-demo.duckdns.org
## Install pip3 python package manager
sudo apt install python3-pip 
## Install certbot and certbot nginx provider
sudo pip3 install certbot certbot-nginx 
## Setup certbot for the correct site
sudo certbot --nginx -d t-siren-demo.duckdns.org
```
#### Postgres 12
```sh
mkdir postgres/db -p && cd postgres
echo "
version: '3.8'
services:
  db:
    image: postgres:12-alpine
    restart: always
## enable wal size and level for replication
    command: -c wal_level=logical -c wal_log_hints=on -c max_wal_senders=8 -c max_wal_size=1GB -c hot_standby=on
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    ports:"
      - '5432:5432'
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
    volumes: 
      - db:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/create_tables.sql
volumes:
  db:
    driver: local
" | tee docker-compose.yml
echo "
### create replication user
CREATE USER rep_user REPLICATION LOGIN ENCRYPTED PASSWORD 'rep_pass';
### create app database
CREATE DATABASE testdb;
### create dummy table & data
CREATE TABLE dummy (
    id SERIAL PRIMARY KEY,
    qoute character varying(255) NOT NULL ,
    author character varying(255) NOT NULL 
);

INSERT INTO dummy (qoute,author) VALUES 
( 'There are only two kinds of languages: the ones people complain about and the ones nobody uses.', 'Bjarne Stroustrup'), 
( 'Any fool can write code that a computer can understand. Good programmers write code that humans can understand.', 'Martin Fowler'), 
( 'First, solve the problem. Then, write the code.', 'John Johnson'), 
( 'Java is to JavaScript what car is to Carpet.', 'Chris Heilmann'), 
( 'Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live.', 'John Woods'), 
( 'I''m not a great programmer; I''m just a good programmer with great habits.', 'Kent Beck'), 
( 'Truth can only be found in one place: the code.', 'Robert C. Martin'), 
( 'If you have to spend effort looking at a fragment of code and figuring out what it''s doing, then you should extract it into a function and name the function after the \"what\".', 'Martin Fowler'), 
( 'The real problem is that programmers have spent far too much time worrying about efficiency in the wrong places and at the wrong times; premature optimization is the root of all evil (or at least most of it) in programming.', 'Donald Knuth'), 
( 'SQL, Lisp, and Haskell are the only programming languages that I’ve seen where one spends more time thinking than typing.', 'Philip Greenspun'), 
( 'Deleted code is debugged code.', 'Jeff Sickel'), 
( 'There are two ways of constructing a software design: One way is to make it so simple that there are obviously no deficiencies and the other way is to make it so complicated that there are no obvious deficiencies.', 'C.A.R. Hoare'), 
( 'Simplicity is prerequisite for reliability.', 'Edsger W. Dijkstra'), 
( 'There are only two hard things in Computer Science: cache invalidation and naming things.', 'Phil Karlton'), 
( 'Measuring programming progress by lines of code is like measuring aircraft building progress by weight.', 'Bill Gates'), 
( 'Controlling complexity is the essence of computer programming.', 'Brian Kernighan'),
( 'The only way to learn a new programming language is by writing programs in it.', 'Dennis Ritchie');
" | tee db/init.sql
docker compose up -d
```
#### UFW setup
```sh
sudo ufw allow 22 #SSH
sudo ufw allow 80 #HTTP
sudo ufw allow 443 #HTTPS
sudo ufw allow from 194.233.166.173 proto tcp to any port 5432 #Postgres for backend
sudo ufw enable
```
### Instance-2
#### Setup Postgres replica
```sh
# Create the file repository configuration:
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
# Import the repository signing key:
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
# Update the package lists:
sudo apt-get update
# Install PostgreSQL.
sudo apt-get -y install postgresq-12 postgresql-client-common
# Stop postgresql service
sudo systemctl stop postgresql
# Change to postgres user
sudo su - postgres
# Move to postgres 12 folder and clear out the main db folder
cd 12 && rm -rf main/*
# Create a copy of the master db and enable stream replication
/usr/lib/postgresql/12/bin/pg_basebackup -D main -h 194.233.169.165 -X stream -c fast -U rep_user -W -R
# go back to main User
exit
# Enable postgres v12
sudo systemctl enable postgresql@12
# Start postgres v12
sudo systemctl start postgresql
```
#### UFW setup
```sh
sudo ufw allow 22 #SSH
sudo ufw allow from 194.233.169.165 proto tcp to any port 8080 #Backend access for nginx proxy-pass
sudo ufw enable
```
## Github flows
### Projects:
#### Trigger project
* [Main-Repo](https://github.com/t-demo-org/main-repo)
* [Triggered-Repo](https://github.com/t-demo-org/triggered-repo)

Main repo triggers a dispatch event onto the triggered repo. More details inside the workflow files.
#### Demo Project
* [Demo-Frontend](https://github.com/t-demo-org/demo-frontend)
    - AngularJS Frontend
    - Published using Nginx
* [Demo-Backend](https://github.com/t-demo-org/demo-backend)
    - Java Springboot Backend with Postgresql database.
    - Published Using docker and through nginx proxy-pass.
The demo is reachable through [t-siren-demo.duckdns.org](https://t-siren-demo.duckdns.org).
More details about each workflow in each project.
