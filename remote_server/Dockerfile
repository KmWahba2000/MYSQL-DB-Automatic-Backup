FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y \
        openssh-server \
        nano \ 
        curl \
        unzip \
        default-mysql-client && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash remote_user && \
    echo "remote_user:1234" | chpasswd && \
    mkdir -p /home/remote_user/.ssh && \
    chmod 700 /home/remote_user/.ssh

COPY remote-key.pub /home/remote_user/.ssh/authorized_keys
RUN chown -R remote_user:remote_user /home/remote_user/.ssh && \
    chmod 600 /home/remote_user/.ssh/authorized_keys

RUN mkdir -p /var/run/sshd && ssh-keygen -A

RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip && \
    unzip awscliv2.zip && \
    ./aws/install && \
    rm -rf awscliv2.zip aws

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
