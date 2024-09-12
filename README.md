# Automatic Installation
## For Debian Bullseye/Bookworm
Just clone this repository on your server, and run `installer`. Automatic installation assumes you have a server with a static public IP address, and you have a domain with properly configured DNS records. It also assumes you want to install the two components of the `singularity`, namely, `dispatch` and `reflector` on the same machine. 

As `singularity` is a distributed runtime, you absolutely can install these components on two different machines, as long as they have their own public IP address and you configure your DNS properly. Refer to the **manual installation** section for more details. 

# Manual Installation

1- Add to `.bashrc`:

```
# will be needed for go:
export PATH=$PATH:/usr/local/go/bin

# optional, for your convenience:
alias restart='systemctl restart'
alias status='systemctl status'
alias stop='systemctl stop'
alias enable='systemctl enable'
```
then: `source .bashrc`

2- Install Required Packages: 
```
apt install -y curl gnupg git nginx ufw
```
3- Install Go:

```
wget https://go.dev/dl/go1.23.1.linux-amd64.tar.gz &&  rm -rf /usr/local/go && tar -C /usr/local -xzf go1.23.1.linux-amd64.tar.gz && rm go*tar.gz
```

4- Install Mongodb:

```
 curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

 echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] http://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" | tee /etc/apt/sources.list.d/mongodb-org-7.0.list

 apt update && apt install -y mongodb-org
```

5. Check Installation Status:
```
 go version #go version go1.23.1 linux/amd64

 enable mongod && restart mongod && status mongod #active
```

## Configuration

### Firewall:

1- Allow OpenSSH (port 22) to avoid future interruptions: ``` ufw allow in OpenSSH ```

2- Enable and allow incoming HTTP (port 80) and HTTPS (port 443) traffic:
```
ufw enable
ufw allow in 80/tcp
ufw allow in 443/tcp
ufw status
```

### SSL Certificates
It's highly recommended to use ssl with mongodb server and nginx. Essentially, you should place the **private key** and **full chain certificate** of your (sub)domain, inside a folder (with appropriate permissions), to later address them in `mongod` and `nginx` configurations.

Here, we assume you're using *let's encrypt* certificates generated from `certbot`. For example:

```
apt install certbot

# obtain a new certificate for your domain:
certbot certonly --key-type ecdsa --elliptic-curve secp384r1

```
Assuming certbot is used, we're particularly interested in:
* `/etc/letsencrypt/live/your_domain/fullchain.pem` 
* `/etc/letsencrypt/live/your_domain/privkey.pem` 

## Mongodb Configuration
1- `mongosh`: 
```
use admin

db.createUser({
  user: "admin_username",
  pwd: "admin_pass",
  roles: [
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "userAdminAnyDatabase", db: "admin" }
  ]
})

exit
```
2- Generate `mongodb.pem`:
```
cat /path/to/privkey.pem /path/to/fullchain.pem > /path/to/mongodb.pem
```
3- Ensure the following is included in `/etc/mongod.conf`:
```
net:
  port: 27017
  bindIp: 0.0.0.0

  tls: 
    mode: requireTLS
    certificateKeyFile: /path/to/mongodb.pem
    CAFile: /path/to/fullchain.pem
    allowConnectionsWithoutCertificates: true #disabling mTLS

security:
  authorization: enabled
```
**IMPORTANT**: Only distribute the `fullchain.pem`. **DO NOT DISTRIBUTE THE `mongodb.pem` AS IT CONTAINS YOUR PRIVATE KEY!**

4- Ensure mongod has access to `.pem` files: `restart mongod && status mongod`

5- In case mongod failed, run `cat /var/log/mongodb/mongod.log | tail | grep permission`, if there was any *permission denied* statements, check for `.pem` file permissions/ownership, otherwise carefully check the `/etc/mongod.conf` for syntax/typo issues.

6- Check local connectivity:
```
mongosh -u admin_username --tls --tlsCAFile /path/to/fullchain.pem --tlsAllowInvalidHostnames
```
7- Allow incoming traffic to mongodb: `ufw allow in 27017`. Do this only after setting `authorization: enabled` as discussed above.

8- Check remote connectivity:
```
mongosh --host your_domain --port 27017 -u admin_username --tls --tlsCAFile /path/to/fullchain.pem
```
9- Your mongodb server is all set! 

## Reflector Configuration

1- Generate a new config file with `nano /etc/nginx/sites-available/reflector`.

2- The following configuration is recommended (default port is `8011`):
```
server {
    listen 443 ssl;
    server_name [reflector_sub_domain];

    ssl_certificate [/path/to/cert/for/reflector_subdomain/]fullchain.pem;
    ssl_certificate_key [/path/to/cert/for/reflector_subdomain]/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
    
    location / {
        proxy_pass http://localhost:[reflector_port];
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

3- Enable the configuration with:
```
ln -s /etc/nginx/sites-available/reflector /etc/nginx/sites-enabled/
```

4- Test the configuration with `nginx -t`

5- Restart the nginx server with `restart nginx`

6- Run the `reflector` program:
```
git clone https://github.com/kevmasasjedi/reflector
cd reflector
go build -o reflector main.go
./reflector
```
7- Make sure `https://reflector_subdomain` is available and working correctly

8- To start reflector as a service run service_install.sh

## Dispatch Configuration

1- Generate a new config file with `nano /etc/nginx/sites-available/dispatch`.

2- The following configuration is recommended (default port is `8012`):
```
server {
    listen 443 ssl;
    server_name [dispatch_sub_domain];

    ssl_certificate [/path/to/cert/for/dispatch_sub_domain/]fullchain.pem;
    ssl_certificate_key [/path/to/cert/for/dispatch_sub_domain]/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
    
    location / {
        proxy_pass http://localhost:[dispatch_port];
    }
}
```
3- Enable the configuration with:
```
ln -s /etc/nginx/sites-available/dispatch /etc/nginx/sites-enabled/
```

4- Test the configuration with `nginx -t`

5- Restart the nginx server with `restart nginx`

6- Download and initialize `dispatch` program:
```
git clone https://github.com/kevmasajedi/dispatch

# to establish tls connection:
cd dispatch
mkdir cert
cp /path/to/mongo/fullchain.pem cert/
```
7- Connect to mongosh as stated before and create a new db with a new user. For example. :
```
use <db_name>

db.createUser({
  user: "user",
  pwd: "pass",
  roles: [{ role: "readWrite", db: "<db_name>" }]
})

exit
```

8- In the *dispatch* directory, `nano .env` and complete it:
```
VERSE_USERNAME="<user>"
VERSE_PASSWORD="<pass>"
VERSE_DB="<db_name>"
VERSE_HOST="<db_sub_domain>"
VERSE_PORT="27017 or <your_mongo_port>"
```

9- Run the dispatch program:

```
go build -o dispatch main.go
./dispatch
```

10- `https://dispatch_subdomain` should be available now. To start reflector as a service run service_install.sh

## Final Steps:

To test all the components of the system are working as intended, let's download an *Event Horizon* to run a *Hello World* example:

1- Clone into event_horizon `git clone https://github.com/kevmasajedi/event_horizon`

2- In the *event_horizon* directory, `nano .env` and complete it (db_name should be the same as db_name used in dispatch):

```
VERSE_USERNAME="<user>"
VERSE_PASSWORD="<pass>"
VERSE_DB="<db_name>"
VERSE_REFLECTOR="<reflector_sub_domain>"
VERSE_HOST="<db_sub_domain>"
VERSE_PORT="27017 or <your_mongo_port>"
```

3- To establish tls connection with mongodb:
```
mkdir cert
cp /path/to/mongo/fullchain.pem cert/
```

4- Run `go run main.go`. It should output something like the following:
```
Self declared as hello with 127.0.0.1:32634 in domain_workers
```
**Note**: If you want to host your *domain worker* on another server, `nano main.go` and change the `"local"` in the AutoInitialize arguments to `"remote"`:
```
autoinvoker.AutoInitialize("hello", &context, []string{"name"}, "remote", "/failed")
```

5- As our domain_worker declared itself as `hello`, we can access it by: `https://dispatch_subdomain/hello`. 

## You're All Set!
Finally, you're all set. You can clone more **Event Horizons** as above and access them via dispatch.
