# Meta TON CON
![space](/img/spaceship.png)

## Introduction

This repo contains technical knowledge about Meta TON CON.

If you have advice or experience sharing about:
- Best practice
- Server hosting
- Server resource usages
- Tips

# Installing
Remember! if you got a problem with npm or dependency that you cant to solve for 1 hour. Just restart your PC. Trust me.

# Requirement:
### Hardware:
- at least 8GB of RAM
- recommended using fast CPU

### Software
- Node js installed v16

### Knowledge
I assume you already know, if no you must up-skill first
- Javascript
- React js
- Basic Webpack dev server
- Basic Elixir and phoenix
- Basic Web Socket

# Overview

![overview](/img/over.png)

### Summary

Reticulum is the main host. it sync position, rotation, state of object. Comunicates with client browser through http request and websocket.

Dialog sync video and audio user. comunicates with clients browser through websocket.

Hubs, Spoke serve static assets then reticulum takes it and forward to client browser.

postREST is a server that help hubs Admin to doing basic task like CRUD (create read update delete)

Hubs Admin use websocket to comunicates with postgREST for authentication (login). for CRUD purpose hubs admin send http request (GET, POST, etc) to reticulum then reticulum doing proxy pass to postgREST.

# Attention!
There is major step [Cloning and Preparation](#1-cloning-and-preparation) -> [Setting up HOST](#2-setting-up-host) -> [Setting up HTTPS (SSL)](#3-setting-up-https-ssl) -> [Running](#4-runing)

# 1. Cloning and preparation

## 1.1 Reticulum

It's a backend server that uses elixir and phoenix.

### 1.1.1 Clone

```bash
git clone https://github.com/mozilla/reticulum.git
cd reticulum
```

### 1.1.2 Install requirement

#### Postgres Database

Install on [linux ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart)

Install on mac

With brew for installing CLI Postgres

```
brew install postgres
```

Then create user/change password

user: `postgres`

password : `postgres`

and alter it 

```
ALTER USER postgres WITH SUPERUSER
```
#### Elixir and Erlang (Elixir 1.12 and erlang version 23)

You can install those with follow [this tutorial](https://www.pluralsight.com/guides/installing-elixir-erlang-with-asdf)

Be careful about the version of elixir and erlang.

<!-- **Ansible**

You can use `pip` to install. take a look at this [tutorial](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#from-pip) -->

### 1.1.3 run this command

1. `mix deps.get`
2. `mix ecto.create`
   - If step 2 fails, you may need to change the password for the `postgres` role to match the password configured `dev.exs`.
   - From within the `psql` shell, enter `ALTER USER postgres WITH PASSWORD 'postgres';`
   - If you receive an error that the `ret_dev` database does not exist, (using psql again) enter `create database ret_dev;`
3. From the project directory `mkdir -p storage/dev`

### 1.1.4 Run Reticulum against a local Dialog instance

1. Update the Janus host in `dev.exs`:

```elixir
dev_janus_host = "localhost"
```

2. Update the Janus port in `dev.exs`:

```elixir
config :ret, Ret.JanusLoadStatus, default_janus_host: dev_janus_host, janus_port: 4443
```

3. Add the Dialog meta endpoint to the CSP rules in `add_csp.ex`:

```elixir
default_janus_csp_rule =
   if default_janus_host,
      do: "wss://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port}/meta",
      else: ""
```

<!-- 4. Find on google how to install coturn, and manage it

[install coturn on ubuntu](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04)

5. Edit the Dialog [configuration file](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04) `turnserver.conf` and update the PostgreSQL database connection string to use the _coturn_ schema from the Reticulum database:

```
psql-userdb="host=localhost dbname=ret_dev user=postgres password=postgres options='-c search_path=coturn' connect_timeout=30"
``` -->

## 1.2 Dialog

Using mediasoup RTC will handle audio and video real-time communication. like camera stream, share screen.

### 1.2.1 Clone and get dependencies

```bash
git clone https://github.com/mozilla/dialog.git
cd dialog
npm install
```

### 1.2.2 Setting up secret key

thanks to this [comment](https://github.com/mozilla/hubs/discussions/3323#discussioncomment-1857495)

Generate RSA (Public and Private key) with [generator online](https://travistidwell.com/jsencrypt/demo/)

make empty file `perms.pub.pem` and fill it with RSA Public key

![RSA generator ](/img/rsa.png)

![Paste](/img/rsa-1.png)

Goto reticulum directory on `reticulum/config/dev.exs` change PermsToken with the RSA private key that you generate before.

```elixir
config :ret, Ret.PermsToken, perms_key: "-----BEGIN RSA PRIVATE KEY----- paste here copyed key but add every line \n before END RSA PRIVATE KEY-----"
```

## 1.3 Spoke

In here you can create/edit the scenes/buildings whatever you call it.

### 1.3.1 Clone

```
git clone https://github.com/mozilla/Spoke.git
cd Spoke
yarn install
```

### 1.3.2 Set the base routes

I hope you know the basic `react-router-dom` with the default URL in  slash `/` on `localhost:9090`

But in the end, we will access the spoke on `localhost:4000/spoke`

So we must set the base URL to `/spoke`

Add the `ROUTER_BASE_PATH=/spoke` params to the `start` command on `package.json`

![Spoke](/img/spoke_change.png)

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.pem --key certs/key.pem
```

## 1.4 Hubs

In this [repo](https://github.com/mozilla/hubs) contains the hubs client and hubs admin (hubs/admin)

Clone and install dependencies

```
git clone https://github.com/mozilla/hubs.git
cd hubs
npm ci
```

## 1.5 Hubs Admin

from the [hubs repo](#14-hubs) you can move to `hubs/admin` then run

```
npm install
```

# 2. Setting up HOST

We are not using `hubs.local` domain. we use `localhost`

so change every host configuration on reticulum, dialog, hubs, hubs admin, spoke.

# 3. Setting up HTTPS (SSL)

All the servers must serve with HTTPS. you must generate a certificate and key file

## 3.1 Generating certificate and making it trust

Open terminal in reticulum directory

run command

```bash
mix phx.gen.cert
```

It will generate key `selfsigned_key.pem` and certificate `selfsigned.pem` in the `priv/cert` folder

Rename `selfsigned_key.pem` to `key.pem`

Rename `selfsigned.pem` to `cert.pem`

#### Now we have `key.pem` and `cert.pem` file

In Mac OS, I don't know in windows or Linux. please find it yourself

Open the `cert.pem` on the tab system find that certificate then clicks twice and change to always trust.

Select the `cert.pem` and `key.pem` and copy it. next step we will distribute those two files into hubs, hubs admin, spoke, dialog, and reticulum.

Oke first set up in the reticulum.

## 3.2 Setting https for reticulum

On the `config/dev.exs` We must be setting the path for the certificate and key file.

![Https mozilla hubs](/img/cert-1.png)

## 3.3 Setting HTTPS for hubs

Paste [that](#now-we-have-keypem-and-certpem-file) file into `hubs/certs`

We run hubs with `npm run local` right?
so add additional params on `package.json`

`--https --cert certs/cert.pem --key certs/key.pem`

Like this picture

![ssl hubs](/img/ssl-hubs.png)

## 3.4 Setting HTTPS for hubs admin

Paste [that](#now-we-have-keypem-and-certpem-file) file into `hubs/admin/certs`

We run hubs with `npm run local` right?
so add additional params on `package.json`

`--https --cert certs/cert.pem --key certs/key.pem`

Like this picture

![ssl hubs admin](/img/ssl-hubs-admin.png)

## 3.5 Setting HTTPS for spoke

Paste [that](#now-we-have-keypem-and-certpem-file) file into `spoke/certs`

We run spoke with `yarn start` right?
So change the `start` command

![ssl hubs admin](/img/ssl-spoke.png)

With this

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.pem --key certs/key.pem
```

Short description:

BASE_ASSETS_PATH = basicaly we run the spoke on localhost:9090

## 3.6 Setting https for dialog

Paste [that](#now-we-have-keypem-and-certpem-file) file into `dialog/certs`

rename `cert.pem` to `fullchain.pem`

rename `key.pem` to `privkey.pem`

![ssl hubs dialog](/img/ssl-dialog-1.png)

# 4. Running

Open five terminals. for each reticulum, dialog, spoke, hubs, hubs admin.

![Running preparation](/img/ss.png)

## 4.1 Run reticulum

with command

```bash
iex -S mix phx.server
```

## 4.2 Run dialog

Edit the `start` command on the package.json with 

```
MEDIASOUP_LISTEN_IP=127.0.0.1 MEDIASOUP_ANNOUNCED_IP=127.0.0.1 DEBUG=${DEBUG:='*mediasoup* *INFO* *WARN* *ERROR*'} INTERACTIVE=${INTERACTIVE:='true'} node index.js
```

For giving params `MEDIASOUP_LISTEN_IP` and `MEDIASOUP_ANNOUNCED_IP`

![Running preparation](/img/run-dialog.png)

Start dialog server with command:
```
npm run start
```

`127.0.0.1` is the default IP of localhost on Mac / Linux you can look at the IP with this command:

```bash
sudo nano /etc/hosts
```

## 4.3 Run spoke

with command

```bash
yarn start
```

## 4.4 Run hubs and hubs admin

Each with command

```bash
npm run local
```

## 4.5 Run postgREST server

More about this is in [this](https://github.com/mozilla/hubs-ops/wiki/Running-PostgREST-locally)

Download postREST

```
sudo apt install libpq-dev
wget https://github.com/PostgREST/postgrest/releases/download/v9.0.0/postgrest-v9.0.0-linux-static-x64.tar.xz
tar -xf postgrest-v9.0.0-linux-static-x64.tar.xz
```

On reticulum iex

paste this
```
jwk = Application.get_env(:ret, Ret.PermsToken)[:perms_key] |> JOSE.JWK.from_pem(); JOSE.JWK.to_file("reticulum-jwk.json", jwk)
```

then it will create `reticulum-jwk.json` in your reticulum directory

Make `reticulum.conf` file 

```
nano reticulum.conf
```
and paste 

```
# reticulum.conf
db-uri = "postgres://postgres:postgres@localhost:5432/ret_dev"
db-schema = "ret0_admin"
db-anon-role = "postgres_anonymous"
jwt-secret = "@/absolute_path_to_your_file/reticulum-jwk.json"
jwt-aud = "ret_perms"
role-claim-key = ".postgrest_role"
```

then the folder looks like this (contain two files)

```
/
   postgrest
   reticulum.conf
```

then run  postREST with

```
postgrest reticulum.conf
```

Now you can access with lock symbol (SSL secure)

### Hubs
[https://localhost:4000](https://localhost:4000)

### Hubs admin
[https://localhost:4000/admin](https://localhost:4000/admin)

### Spoke
[https://localhost:4000/spoke](https://localhost:4000/spoke)
