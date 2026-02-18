# Setup MQTT in Docker

Please follow the step to install MQTT broker in Ubuntu 24 using docker compose

### Step 1

Make your system ready for the installation.

```
sudo apt-get update && sudo apt-get upgrade

```

### Step 2

Install Docker.

```
curl -sSL https://get.docker.com | sh

```

### Step 3

Create a directory to store all container related data in one place

```
sudo mkdir -p /opt/stacks/mqtt

cd /opt/stacks/mqtt

```

### Step 4

Download the mqtt-docker-compose.yml file

```
curl https://raw.githubusercontent.com/deepayannandy/quickCommands/refs/heads/main/MQTT/mqtt-docker-compose.yml | cat >> mqtt-docker-compose.yml

```

### Step 5

Create the config directory and config file

```
sudo mkdir config
sudo nano ./config/mosquitto.conf

```

And pest this configuration

```
#Configurations
listener 1883       #MQTT port
listener 9001       #WS port
protocol websockets
persistence true
persistence_file mosquitto.db
persistence_location /mosquitto/data/

#Authentication
allow_anonymous false
password_file /mosquitto/config/pwfile

```

Press CTRL + X to save and exit.

### Step 6

Create a Password file and give the file the require accesses

```
sudo touch ./config/pwfile

sudo chmod 0700 ./config/pwfile

```

### Step 7

Let's start the Docker container for the fast time to setup the credentials.

```
docker compose up -d

```

### Step 8

In this step we will telnet inside the container and execute the command to create and username and password

```
sudo docker compose exec mosquitto sh

mosquitto_passwd -c /mosquitto/config/pwfile <USERNAME>

```

Enter the Username asper your choice and verify the password for that user id.

### Step 9

In this step we will restart the docker container so that it can pick the credential details.

```

sudo docker compose restart mosquitto

```

Now your MQTT broker is up and running for you production use.
