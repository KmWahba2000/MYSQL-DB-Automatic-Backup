## MySQL DB Automatic Backup

**Overview:**

In this project, I automate MySQL database backups using **Jenkins** and **Bash scripting**.

The setup runs on an Ubuntu machine with **Docker** and **Docker Compose**, which I use to build three containers: **Jenkins**, a **MySQL database server**, and a **remote server**. Jenkins runs inside a Docker container with its data mapped to the local machine, the MySQL container hosts the database, and the remote server connects to the database through SSH.

Backups are stored in an **AWS S3 bucket**. I create a dedicated IAM user and apply a custom policy that allows access only to this bucket. The access key and secret key are then used during the backup process.

I write a Bash script to take a database backup and upload it to S3, and Jenkins is used to run this script. S3-related variables are configured in Jenkins so the backup destination can be changed easily later if needed.

---

### **1- Building Jenkins Image**

I used the official Jenkins Docker installation method, but instead of running the Dockerfile directly, I built it using **Docker Compose**.

This is the Dockerfile used to build the Jenkins image:

```Dockerfile
FROM jenkins/jenkins:2.528.3-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"
```

I used **Docker Compose** to build the Jenkins image along with the other two images at the same time. This makes it easier to modify the setup and recreate the containers at any time.

The commands used:

```bash
docker compose build
docker compose up -d
```

*These commands must be executed from the directory that contains the `docker-compose.yml` file.*

The output will look like this:

![Alt text](assets/1.png)

After the image is built, Jenkins generates an initial admin password that is required for the first-time setup.
It can be retrieved using the following command:

```bash
docker logs -f <container_name>
```

This is the part of the Docker Compose file responsible for creating the Jenkins container:

```yml
my-jenkins:
  build:
    context: .
    dockerfile: Dockerfile
  container_name: my-jenkins
  restart: on-failure
  networks:
    - jenkins
    - remote
  volumes:
    - /home/karim/jenkins-data:/var/jenkins_home
    - /var/run/docker.sock:/var/run/docker.sock
  ports:
    - "8080:8080"
    - "50000:50000"
```

It uses the Dockerfile defined above, assigns the required networks, and maps volumes to keep Jenkins data persistent even after container deletion.
The `remote` network is shared with the remote server to allow SSH communication.
The required ports are also exposed to access Jenkins.

---

### **2- Building MySQL Server Image**

This is the Docker Compose configuration used to build the MySQL container:

```yml
db_host:
  image: mysql:latest
  container_name: db
  restart: always
  environment:
    MYSQL_ROOT_PASSWORD: "1234"
  volumes:
    - /home/karim/db_data:/var/lib/mysql
  networks:
    - db
```

After the image is created, I can access the container using:

```bash
docker exec -it db bash
```

---

### **3- Building Remote Server Image**

This is the Docker Compose configuration used to build the remote server container:

```yml
remote-server:
  build:
    context: ./remote_server
    dockerfile: Dockerfile
  container_name: remote_server
  restart: unless-stopped
  volumes:
    - /home/karim/scripts/aws-s3.sh:/tmp/script.sh
  networks:
    - remote
    - db
  ports:
    - "2222:22"
```

This server is connected to Jenkins and is responsible for interacting with the database.
The backup script is mapped from the local machine to the container to keep it persistent.

I can access the container using:

```bash
docker exec -it remote_server bash
    mysql -u root -h db -p
```

Inside the database, I added some data:

```mysql
create database testdb;
use testdb;
create table info (FirstName varchar(20), LastName varchar(20), Age int(2));
insert info values ('Karim','Wahba','25');
select * from info;
```

![Alt text](assets/2.png)

![Alt text](assets/3.png)


I can also manually take a backup of the database created in the MySQL container:

```bash
mysqldump -u root -h db -p testdb > /tmp/db.sql
```

![Alt text](assets/17.png)

![Alt text](assets/18.png)


If needed, the backup can be manually uploaded to the S3 bucket. Before doing that, AWS S3 and the AWS CLI must be configured.

---

### **4- Creating AWS S3 Bucket**

I created the S3 bucket:

![Alt text](assets/4.png)

Then I created an IAM user:

![Alt text](assets/5.png)

I used the AWS Policy Generator to create a custom IAM policy that allows the user to access only this specific bucket:

![Alt text](assets/6.png)

![Alt text](assets/7.png)

![Alt text](assets/8.png)


The policy was then attached to the user:

![Alt text](assets/9.png)

![Alt text](assets/10.png)

![Alt text](assets/11.png)

![Alt text](assets/12.png)

This is the onlytime i can view the secret access key, so i downloaded the CSV file

![Alt text](assets/13.png)


After that, I generated access keys for the user and used the access key and secret access key to configure the AWS CLI.

![Alt text](assets/14.png)

![Alt text](assets/15.png)

![Alt text](assets/16.png)


**S3 Permissions Issue & Fix**

When I tried to list S3 buckets, I didn’t have access to S3 in general, so the command failed:

![Alt text](assets/20.png)

I then tried to upload the backup using the `aws s3 cp` command, but it also failed because the IAM policy allowed access to the bucket itself, not to the objects inside it:

![Alt text](assets/21.png)

To fix this, I added `/*` at the end of the bucket ARN to allow access to all objects inside the bucket:

![Alt text](assets/22.png)

After that, the upload worked correctly:

![Alt text](assets/23.png)

Once configured, I was able to manually upload the database backup to the S3 bucket using:

```bash
aws s3 cp /tmp/db.sql s3://jenkins-mysql-backup2026/db.sql
```

![Alt text](assets/24.png)

---

### **5- Writing Bash Script for the Backup**

The script has a single purpose: take a MySQL database backup and upload it to S3.

![Alt text](assets/25.png)

I later refined the script by using environment variables to make it easier to modify and to keep sensitive data secure. These variables are defined later in Jenkins as secret values, so no passwords are hardcoded anywhere.

---

### **6- Jenkins Configuration**

I added credentials in Jenkins for the database password and the AWS secret access key as **secret text**:

![Alt text](assets/26.png)

![Alt text](assets/27.png)

![Alt text](assets/28.png)

I generated an RSA key pair on the local machine:

```bash
ssh-keygen -t rsa -b 4096 -f remote-key
```

This generated a public and a private key.
I copied the public key to the remote server’s `authorized_keys` file, and added the private key to Jenkins to set up SSH credentials between Jenkins and the remote server.

![Alt text](assets/39.png)

![Alt text](assets/29.png)


Finally, I created a **freestyle Jenkins project** named `MYSQL-BackupToAWS`:

![Alt text](assets/30.png)

I passed the required parameters to the script:

![Alt text](assets/31.png)

![Alt text](assets/32.png)

![Alt text](assets/33.png)

I also used the secret parameters defined earlier for the database password and AWS secret access key:

![Alt text](assets/34.png)

When the job is built, Jenkins asks for the defined parameters, which makes it easy to reuse the job for another database or a different S3 bucket.

![Alt text](assets/36.png)

The job runs successfully, and the console output confirms that the backup was uploaded to the S3 bucket:

![Alt text](assets/35.png)

![Alt text](assets/38.png)

