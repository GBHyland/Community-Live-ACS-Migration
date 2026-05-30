# ACS Migration Preparation

---  

**Connect to your provided Server and follow these steps on that server**

### Update the Server
```
sudo apt update
sudo apt upgrade -y
```

### Install Required Packages
```
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

### Add Docker’s official GPG key

**Add Docker’s official GPG key**  
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Add the Docker repository
```
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker
```
sudo apt update
```
```
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Start and enable Docker
```
sudo systemctl enable docker
sudo systemctl start docker
```

### Verify Docker works
```
docker --version
```
```
sudo docker run hello-world
```

### Allow your user to run Docker without sudo
```
sudo usermod -aG docker ubuntu
```
```
newgrp docker
```

### Test non-sudo Docker
```
docker ps
```

> If this command runs then you are done!


---  

### Follow Steps on the Migration Guide
[Alfresco OnPrem to Cloud Lab](https://github.com/GBHyland/Community-Live-ACS-Migration/blob/main/migrate-to-cloud-guide.md)  


---  









