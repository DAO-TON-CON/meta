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
There is major step [Cloning and Preparation](#1-cloning-and-preparation) -> [Setting up HOST](#2-setting-up-host) -> [Setting up HTTPS (SSL)](#3-setting-up-https-ssl) -> [Running](#4-running) 

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

# Estimating Costs and Cost Charts

* How do costs work for Meta?
* Minimizing costs - Recommended
* Minimizing costs - Settings in stack template

## Disclaimer for Estimating Costs

Estimating costs is difficult because bills by resource usage and everyone uses Meta differently. Below are estimates from our tests and should not be used as a source of truth for your Meta costs.


## Accurately Predict Future Costs - AWS Cost Explorer
For the most accurate way to see previous costs to predict your future costs enable:

* [AWS Cost Explorer](https://console.aws.amazon.com/billing/home)
* [AWS Cost Explorer Documentation](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/ce-what-is.html)


## Minimize your Costs
Our recommendation to minimize costs for automatic settings is to turn database pausing on by default. When no one is using your hub, turn your hub to Offline mode or a small instance type like t3.medium. Also use a Cloudflare worker as your content CDN.

### Before TON CON: Development
For development with only a few users connecting + setting rooms + scenes, we recommend at least a t3.medium instance (?). When not in use, set your instance to Offline mode. Then switch back to Online when beginning development again.

### Before your event: 1.5 hours
If your instance is in Offline mode, manually update the stack to Online and wait 10 minutes.

```
Offline Mode - manual
When you set Offline mode to "Offline", you've completely turned off your servers and stopped all EC2 costs + database costs. You're still paying for storage for your backups and data. No one can connect to your hub while your servers are "Offline". While "Offline," your hubs instance will redirect you to the specified offline url.

Turning Offline mode to "Offline" to "Online" and vice versa is a manual process. Wait 10 minutes afterward to connect.
```

After, at least 1 hour before event, manually update the stack to scale up your AWS Server Type. For example 1 hour before your event, update the stack from a t3.medium to c4.large (?).

### Monthly Database Budget - automatic
Careful with the Monthly Database Budget setting, we recommend $0 (unlimited) or at least $20 or more. If costs hit your set database budget (set other than $0), your database will forcibly shut off for the month. This allows no surprise costs for the cost sensitive.

Personal and Enterprise defaults to $0 (unlimited).

## During TON CON
If you notice performance issues, you can ad hoc update the stack up more from a c4.large to c5.2xlarge Your users in the rooms will have a brief freeze/voice drop while the users roll to the new servers.

## After TON CON
Scale down your AWS Server Type by updating the stack from the c5.2xlarge to t3.medium when finished or there are less users connected.

## When no one is connecting to your instance for a long time
You can turn your hub to Offline mode where no one can connect to your hub or a redirect URL, if specified. Via Offline mode all costs except for asset storage like backups, scenes, and avatars are $0.

# Stack Cost Management Options
You can change various settings of your hub's stack by performing a stack Update. You will not experience any downtime when making these changes. To Update your stack:

* Select the stack in the CloudFormation console
* Go to Stack Actions -> Update Stack
* Choose "Use current template"
* Review the parameter selections and choose 'Update'
Some of the things you can do via a stack update:

* Change the number and type of servers
* Switch your hub into Offline Mode to save costs (and redirect to a URL)
* Add or change a monthly database budget or adjust storage limits
* Add or remove an Application Load Balancer
* Disable or enable database auto-pausing
* Change the database max ACU capacity
* Change the SSH keypair used by your servers
Some things you should not update or change after the stack is created, and should leave as-is:

* Your domains or mail settings
* Everything under Restore from Backup section (to restore from a backup, see Backup and Restore)
* Everything under Advanced

```
* Enable Auto-Pause Database. On by default for Personal and settable by Enterprise.
* Toggle Offline mode to "Online" to "Offline" manually. Your EC2 and database costs will be $0/hour when you've turned your servers off.
* Set Account Monthly Database Budget
* Enable Content CDN to Cloudflare workers
```
## Database Pausing - automatic
If Auto-Pause Database or database pausing is "Yes - Pause database when not in use", after no one has connected to your instance for a while, your database and the costs incurred by your database will stop until a user connects again. It takes 1-3 minutes for the database to turn back on and allow the first user to connect. Subsequent connections will occur quickly afterward.

## Offline Mode - manual
When you set Offline mode to "Offline", you've completely turned off your servers and stopped all EC2 costs + database costs. You're still paying for storage for your backups and data. No one can connect to your hub while your servers are "Offline". While "Offline," your hubs instance will redirect you to the specified offline url.

Turning Offline mode to "Offline" to "Online" and vice versa is a manual process. Wait 10 minutes afterward to connect.

## Monthly Database Budget - automatic
Careful with the Monthly Database Budget setting, we recommend $0 (unlimited) or at least $20 or more. If costs hit your set database budget (set other than $0), your database will forcibly shut off for the month. This allows no surprise costs for the cost sensitive.

Personal and Enterprise defaults to $0 (unlimited).

# Rough Calculation for Estimating Costs

* № of servers = Personal (1 server), Enterprise multi-server (varies, 2 app x 2 stream = 4 servers)
* № Hours in state expected to be in estimated Scalar state
* Cost for EC2 (US$/hr) see below estimate cost charts (alpha)
* SCALAR Roughly estimate costs of running other services like RDS, EFS, and Data transfer costs.
   * 5x - roughly TOP ACTIVE CAPACITY, estimate a hard upper bound and heavy other service use: top CCU capacity, streaming videos, large scenes, avatars moving and talking.
   * 4x - AVERAGE USE for other service: no videos, some people connected
   * 2x, 3x - ACTIVE, a few people are connected, setting up development environment or creating scenes
   * 2x - ONLINE, NOT ACTIVE, database pausing is off
   * 1.2x - ONLINE, NOT ACTIVE, not connected and database pausing is on
   * ~ 0x - Offline mode, paying only for scene, avatar assets and backups
   * To understand these states, read Recommended User Story in Minimizing Costs Page
   * Use AWS's cost explorer to estimate previous costs for future ones.

## THE FORMULA

```Estimate for stack state ($) = # Hours in state x Cost for EC2 (US$/hr) x # of servers x SCALAR (#)```

## Meta Cost Rough Estimate
1. Base cost (easier to estimate) = Setup Cost + Off-time (Scaled down "Online Not Active" or "Offline")
2. Min Total Event Cost = Min Top Capacity Est. + Base cost
3. Max Total Event Cost = Max Top Capacity Estimated + Base cost

## EXAMPLE COST ESTIMATE
TON CON with expected 500 CCU for 4 hours for 2 days. Cost charts estimate an Enterprise 2x2 c5.xlarge instance is sufficient.

1. Calculate Top Capacity Min to Max Range
   * Minimum Top Capacity Est. (~ $103.68) = (4hrs x 2days) x ($1.08 - c5.xlarge) x (4 - Enterprise 2x2) x 3 SCALAR
      * 3x SCALAR reasoning - people will be connected, with a few streaming videos in rooms
   * Max Top Capacity Est. (~ $172.80) = (4hrs x 2days) x ($1.08 - c5.xlarge) x (4 - Enterprise 2x2) x 5 SCALAR   
      * 5x SCALAR reasoning - upper bound for costs just in case
2. Off-time, I'm putting the instance in offline mode.
* ~$0 - ~$10 for storing backups
* If you want the instance to be online, do THE FORMULA calculation for a different instance size. Use x1.2 SCALAR for database pausing and x2 SCALAR for database pausing off.
3. During setup/development, I use the t3.medium. I need to create scenes + deploy a custom client. I estimate that will take me an active 16 hours.
   * Development cost (~ $43) = (16 hrs) x ($0.224 - t3.medium) x (4 - Enterprise 2x2) x 3 (reasoning, upper bound for costs just in case)

## EXAMPLE TOTAL COST ESTIMATE
1. Base cost = (Setup = $43) + (Off-time = ~ $0) = ~ $43
2. Min Total Event Cost = (Min Top Capacity Est. = ~ $103.68) + (Base cost = ~ $43) = ~ $146.68
3. Max Total Event Cost = (Max Top Capacity Est. = ~ $172.80) + (Base cost = ~ $43) = ~ $216.80

Rough HC Cost Range for Example Event = ~ $146.68 - ~ $216.80

## Minimize # Hours at top Capacity to Minimize Cost
If you are diligent with decreasing the # of hours at top capacity, outlined in the minimizing costs user story, your event costs can be extremely low especially when comparing an in-person event:

* Scale the EC2 instance down during lower traffic
* Turn on offline mode (costs are extremely minimal because the EC2 costs + RDS costs + EFS costs are down and you're only storing backups)
* Enable database pausing
* Use Cloudflare for content CDN - not recommended if you're streaming videos

## EC2 Server Type Recommendations
See Cost Charts BELOW to factor costs with the recommendations.

t3.medium is recommended for development/setup with only a few users connecting + setting rooms + scenes.

```Note: This does not exclude the t3.small, see what works for you. Scale the server type up or down ad hoc via updating the stack.```

c4.large is recommended for during an event.

```Note: This does not exclude any other instances types, see what works for you. Scale the server type up or down ad hoc based on performance via updating the stack.```

We do not recommend using a t3.micro because of low memory.

## Why Enterprise 2 app x 2 stream?
We recommend using an Enterprise 2x2 multi-server setup to optimize for resiliency. If one server goes down suddenly, the other will take its place without your users noticing.

Scale vertically before scaling horizontally. Horizontal scaling can result in running out of Let's Encrypt Certificates between servers (we've seen this for 12x12 builds).

For very large events 4x4 and 8x8 Enterprise multiserver stacks are recommended.

## Estimate Costs Charts
How to read and use Cost Charts
* CCU is the concurrent users connected on an instance.
* vCPU (#) is defined by the EC2 Server Type see Amazon EC2 Instance Types Documentation.
* CCU Min - Max CCU for active avatars (lot of avatar movement, talking, at a meetup, etc.)
* CCU Max - Max CCU for mostly inactive avatars (only watching a video, 1 avatar speaking)
* Cost for EC2 (US$/hr) - Cost per hour for running the EC2 instance type in us-east-1 (N. Virginia).
* Note: t3.micro, t3.small, t3.medium have smaller CCU/vCPU because of past performance experience and lower memory.

Below are our CCU estimates for best performance. Performance may vary depending on client power: high power devices (Desktop/VR) vs. low power devices (Mobile).

## Estimating Personal / Enterprise Costs with 1 server

| EC2 Server Type                | vCPU (#) | CCU Min | CCU Max | Cost for EC2 (US\$/hr) |
| ------------------------------ | -------- | ------- | ------- | ---------------------- |
| t3.micro _**NOT recommended**_ | 2        | N/A     | N/A     | \$0.024                |
| t3.small                       | 2        | 10      | 20      | \$0.035                |
| t3.medium                      | 2        | 20      | 40      | \$0.056                |
| t3.large                       | 2        | 40      | 80      | \$0.183                |
| t3.xlarge                      | 4        | 80      | 160     | \$0.266                |
| t3.2xlarge                     | 8        | 160     | 320     | \$0.433                |
| c4.large                       | 2        | 40      | 80      | \$0.200                |
| c5.large                       | 2        | 40      | 80      | \$0.185                |
| c5.xlarge                      | 4        | 80      | 160     | \$0.270                |
| c5.2xlarge                     | 8        | 160     | 320     | \$0.440                |
| c5.4xlarge                     | 16       | 320     | 640     | \$0.780                |
| c5.9xlarge                     | 36       | 720     | 1,440   | \$1.630                |
| c5.12xlarge                    | 48       | 960     | 1,920   | \$2.140                |
| c5.18xlarge                    | 72       | 1,440   | 2,880   | \$3.160                |
| c5.24xlarge                    | 96       | 1,920   | 3,840   | \$4.180                |

### Estimating Enterprise Costs for 4 servers

2 app x 2 streaming servers recommended for best performance. 

| EC2 Server Type                | Total vCPU (#) | Min CCU | Max CCU | Cost for EC2 (US\$/hr) |
| ------------------------------ | -------------- | ------- | ------- | ---------------------- |
| t3.micro _**NOT recommended**_ | 8              | N/A     | N/A     | \$0.096                |
| t3.small                       | 8              | 80      | 160     | \$0.140                |
| t3.medium                      | 8              | 80      | 160     | \$0.224                |
| t3.large                       | 8              | 240     | 400     | \$0.732                |
| t3.xlarge                      | 16             | 480     | 800     | \$1.064                |
| t3.2xlarge                     | 32             | 960     | 1,600   | \$1.732                |
| c4.large                       | 8              | 240     | 400     | \$0.800                |
| c5.large                       | 8              | 240     | 400     | \$0.740                |
| c5.xlarge                      | 16             | 480     | 800     | \$1.080                |
| c5.2xlarge                     | 32             | 960     | 1,600   | \$1.760                |
| c5.4xlarge                     | 64             | 1,920   | 3,200   | \$3.120                |
| c5.9xlarge                     | 144            | 4,320   | 7,200   | \$6.520                |
| c5.12xlarge                    | 192            | 5,760   | 9,600   | \$8.560                |
| c5.18xlarge                    | 288            | 8,640   | 14,400  | \$12.640               |
| c5.24xlarge                    | 384            | 11,520  | 19,200  | \$16.720               |