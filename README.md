
# Traefik, Self-Signed and Portainer configuration

This repo contains what is needed to have:
* A self-signed certificate for a domain (so a wildcard one)
* The traefik and Portainer configuration needed to deploy the Traefik reverse proxy with http and https read config plus a portainer config




## The why
So I have a physical server that I use for hosting some services at home and I use portainer to host/manage some containers (like jellyfin, some ldap tools, etc.). I use portainer because I am lazy and it gives me a nice view on my pods
Most of the time I do not use traefik, I just expose my pod with different ports and deal with the fact that I have to access some weird url like http://this-pod:8080 and http://that-pod:8083.
I use to have rancher to host some pods and rancher use treafik under the hood which was working perfectly fine.. for a while. I love Rancher but it causes some io issues that just make my all server unuseable from time to time so I have deciced (as suggested by my little brother) to have a separate server to test Rancher and hove pods running on vms.
Now that Rancher is not here, I thought that maybe try to use treafik could be cool to access some pods, that the whole reason behind this setup

Oh! just few more details:
* I do not pretend to the an expert, so maybe this stuff may not work for you
* I ma lazy that my I am use self signed certs in my home domain. I know that there are ways to have it working with a valid certificate and all but I do not have the time right now :)

## Tech details
To make this banana sandwich, you will need:
* A dns server in your home lab. I use pfsense at home that host my dns server for my domain.  
* A server that is not a grill cheese maschine that can host some vms
## Howto
- The vm will be called tools.mydom.lan.
- docker is installed on this vm (docker-ce)

- mydom.lan is your domain

- One pod that we will create will be jellyfin and will be accessible use jellyfin.mydom.lan.

Create a alias or cname for jellyfin to your tools.mydom.lan dns entry (because it will be the same ip)

* In your mydomain.lan, create a host tools that point to the ip of your vm
* Create a cname/alias for jellyfin that point for tools.mydomain.lan
* Create another cname/alias for the portainer pod itself (portainer-tools.mydom.lan)
* Create another cname/alias for the traefik dashboard (traefik-tools.mydom.lan)

Note: You can use whatever name here :)

Now,
* Clone this repo on the tools vm
* in the ./certs folder, use the certs.sh script to generate a wildcard certificate for your domain, like
```
$ ./certs.sh mydom.lan
```
this will generate some cert/key file (selfsigned.pem, selfsigned.crt, selfsigned.key) that will be used later

- In the root folder for this git repo you have:
    - The docker-compose.yml
    - a folder called config containing the treafik.. config.

Let's look a those briefly:
### docker-compose.yml
The port part:
```
    ports:
      - "80:80"      # HTTP
      - "443:443"    # HTTPS
      - "8888:8888"  # Traefik dashboard
      - "5432:5432"  # a tcp port for postgres for ex
```
define the ports on which treafik will listen to. Here we have:
- 80 for http requests
- 443 for https requests
- 8888 for the treafik dashboard
- 5432 will be use as a tcp port for postgres for ex

Just remind those stuff because we will check them later in the config

```
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./certs:/certs:ro"
      - "./config:/etc/traefik:ro"
```
What is important here is the ./certs folder which contains our certificate and key and the config that contains.. the config.

You will find also the definition of the creatiun of the portainer pod

Note: Please create the portainer_data volume using
```
docker volume create portainer_data
```
you will need this.

Just replace the =Host(`<host>.<domain>`) with what fit your settings.

For the config/treafik.yml config file:
```
entryPoints:
  dashboard:
    address: ":8888"  # Dashboard on port 8888

  web:
    address: ":80"

  websecure:
    address: ":443"
    http:
      tls: {}

  tcp:
    address: ":5432"

api:
  dashboard: true
  insecure: true  # Enables Traefik dashboard on port 8080

tls:
  certificates:
    - certFile: "/certs/selfsigned.crt"
      keyFile: "/certs/selfsigned.key"

providers:
  docker:  # Enable Docker provider
    exposedByDefault: false  # Only expose services with labels
```

You can see here that the ports defnied earlier have now names (we, websecure). The dashbaord has been enabled and the certificates are defined here. Nothing to change here for now.

Note: let say you want to add another tcp or udp port, you will have to modify the config here plus the docker-compose.yml file and reploy. See the command below

Now that all has been setup, you can deploy the solution:
```
docker compose up -d --force-recreate
```

Now you can access:
http://traefik-tools.mydom.la:8888 for the treafik dashboard (it's fun but nothing really useful)
and
https://portainer-tools.mydom.lan

Do it quickly to setup the password and the user for portainer. If you received a message that the session expired, just re-deploy using the same command as above

Note:
If you have already deployed some pod, they will appear in portainer but you cannot really managed them from portainer. My best recommandation is to:
- do an inspect on the pod and get:
    - image name
    - mapped volumes
    - ports exposed
    - environements variables

then you could create a stack with the correct definition. See below for an example.

At this point you have traefik and portainer working.



## Adding a stack - jellyfin
Ok!
Let's create a stack to add jellyfin. In portainer add a stack and define it like this:

