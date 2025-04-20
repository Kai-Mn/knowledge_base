1) create lxc
2)
``` 
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
3) `sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
4) `sudo docker run hello-world`
5) `touch /etc/systemd/network/.pve-ignore.eth0.network`
6) `nano /etc/systemd/network/eth0.network`
```
DHCP = yes
IPv6AcceptRA = true
```
6) `systemctl restart systemd-networkd.service`
7) `/etc/docker/daemon.json`\
    ```
    {
        "experimental": true,
        "ipv6": true,
        "fixed-cidr-v6": "fd52::/64"
    }
    ```  
8) `systemctl restart docker`
9) `docker network inspect bridge`
10) `docker run --rm alpine ping -6 -c 4 google.ch`

