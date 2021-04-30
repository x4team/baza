# Install SonarQube on Ubuntu 18.04

## Step 1 - Perform a system update

```
sudo apt-get update
sudo apt-get -y upgrade
```

## Step 2- Install and configure PostgreSQL

```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo apt-get -y install postgresql postgresql-contrib
```

```
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Switch to the postgres user.

```
sudo passwd postgres
```

### Create a new user by typing:

```
createuser sonar
```

### Switch to the PostgreSQL shell.

```
psql
```

### Set a password for the newly created user for SonarQube database.

```
ALTER USER sonar WITH ENCRYPTED password 'P@ssword';
```

### Create a new database for PostgreSQL database by running:

```
CREATE DATABASE sonar OWNER sonar;
```

### Exit from the psql shell:

```
\q
```

### Switch back to the sudo user by running the exit command.

```
exit
```


## Step 3: Download and configure SonarQube

```
cd /tmp
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.8.0.42792.zip
```

### Install unzip by running:

```
apt-get -y install unzip default-jre
sudo unzip sonarqube-8.8.0.42792.zip -d /opt
sudo mv /opt/sonarqube-7.3 /opt/sonarqube
sudo chmod 0777 -R /opt/sonarqube
```

### Assign permissions to sonaruser user for directory /opt/sonarqube

```
sudo groupadd --system sonaruser
sudo useradd -s /sbin/nologin --system -g sonaruser sonaruser
```

```
sudo chown -R sonaruser:sonaruser /opt/sonarqube/
```

### Open the SonarQube configuration file using your favorite text editor

```
sudo nano /opt/sonarqube/conf/sonar.properties
```

### Uncomment and provide the PostgreSQL username and password of the database that we have created earlier. It should look like:

```
sonar.jdbc.username=sonar
sonar.jdbc.password=P@ssword
sonar.jdbc.url=jdbc:postgresql://localhost/sonar
sonar.web.javaAdditionalOpts=-server
```

## Step 4: Configure Systemd service

```
sudo nano /etc/systemd/system/sonar.service
```

```
[Unit]
Description=SonarQube service
Documentation=https://docs.sonarqube.org/latest/setup/install-server/
After=network.target

[Service]
Type=simple
User=sonarqube
Group=sonarqube
WorkingDirectory=/opt/sonarqube
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl start sonar
sudo systemctl enable sonar

sudo systemctl status sonar
```

## Step 5 — Setting Up SonarQube

If you've setup in you local system then visit http://127.0.0.1:9000/
