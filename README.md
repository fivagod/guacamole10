# Auth Token application

## Prequisites

### Creating token signature with bash

```
secret='1234567890'
location="/testlocationlive"
path="/testlive.smil/"
nva_pref="?nva="
nva="1538337566"
dir_pref="&dirs="
dirs="1"
file="playlist.m3u8"
token=$(echo -n ${path}?nva=${nva}&dirs=${dirs} | openssl sha1 -hmac $secret -binary | xxd -p | cut -c1-20)
```

### Creating URL

```
echo "$location/token=nva=$nva~dirs=$dirs~hash=0$token$path$file"
```

### URL example

```
/testlocationlive/token=nva=1538337566~dirs=1~hash=004acb40fa3d37b94fdcd/testlive.smil/playlist.m3u8
```

## Application

### Requirements

Python 2.7
pip
virtualenv
git

```
apt-get update && \
apt-get install -y python-dev python-pip python-virtualenv git
```

### Clone and prepare environment

```
set -e
mkdir -p /var/lib/auth_token
cd /var/lib/auth_token
git clone --depth 1 https://github.com/freddygood/guacamole10.git app
cd app
virtualenv venv
. venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
deactivate
```

### Start application manually

```
cd /var/lib/auth_token/app
. venv/bin/activate
uwsgi --ini uwsgi.ini
```

### Start application

#### systemd (Ubuntu 16)

```
cp /var/lib/auth_token/app/auth_token.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable auth_token.service
systemctl start auth_token.service
```

#### upstart (Ubuntu 14)

```
cp /var/lib/auth_token/app/auth_token.conf /etc/init/
start auth_token
```

### Configuration

There is a config file /var/lib/auth_token/app/config.py for configuration standalone application:

```
host = '0.0.0.0'
port = 8080
debug = True
cache_timeout=60

secret_default = 'qwertyuiop'
secret = {
        'testlocationlive': '1234567890',
        'salloum': 'djdheylsksjlak248du'
}

```

- cache_timeout - time to cache validate token function (integer)
- secret_default - default secret (string)
- secret - secret by location (dict). Used during the validation token. The script matches first part of URL (with no leading slash) with keys of the dictionary. If a key not found the default secret will be used. For example:

- /testlocationlive/token.. - secret `1234567890` will be used
- /verynicelocation/token.. - secret `qwertyuiop` will be used

There is /var/lib/auth_token/app/uwsgi.ini file for configuration uwsgi daemon:

```
[uwsgi]

master = true
module = wsgi
socket = 0.0.0.0:8080
processes = 16
harakiri = 15
```

- socket - ip and port to listen
- processes - number of workers, way to scale
- harakiri - kill and restart worker in case it hangs

The application must be restarted to apply configuration files changes

### Restart the application

#### systemd

```
systemctl restart auth_token.service
```

#### upstart

```
restart auth_token.service
```

### Deployment

The application might be deployed on the same servers with nginx and on dedicated servers as well.

#### Easy deployment

Each edge server with nginx has it's own instance of application and sends requests to it 127.0.0.1:8080

#### Advanced deployment

There are dedicated servers with the application working in parallel. Requests must be balanced with upstream module in nginx.

## Nginx configuration

### Upstream definition

Insert the block within http section (before servers definition)

```
upstream auth_token {
        server 127.0.0.1:8080;
}
```

### Auth location

Create one location per server

```
# Auth token common location
location = /auth_token {
        internal;
        include uwsgi_params;
        rewrite / $request_uri break;
        uwsgi_pass auth_token;
        uwsgi_pass_request_body off;
}
```

### Streaming secured location

Create the block per each secured location

```
# Secured testlocationlive location
location /testlocationlive/token {
        auth_request /auth_token;
        error_page 404 =200 @testlocationlive_auth_passed;
}
location @testlocationlive_auth_passed {
        rewrite ^(/testlocationlive)/token=.*hash=[a-z0-9]+(/.*)$ $1$2 last;
}
```

### Token checking service

Only for dev and test purpose - ability to check validity of token and secured URL without hitting any files

```
# Checking token service
location /_check_auth_token  {
        auth_request /auth_token;
        error_page 404 =200 @check_auth_passed;
}
location @check_auth_passed {
        return 200;
}
```

### Using token checking service

Response code 200 or 403

```
curl -sv http://nginx.testcdn.yes/_check_auth_token/token=nva=1537000000~dirs=1~hash=06bffd04a860d31992619/testlive.smil/playlist.m3u8
```
