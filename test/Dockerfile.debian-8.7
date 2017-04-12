FROM debian:8.7
ENV container docker

RUN apt-get update; apt-get -y upgrade

RUN DEBIAN_FRONTEND=noninteractive apt-get -q -y install systemd openssh-server sudo openssh-client

RUN systemctl mask dev-mqueue.mount dev-hugepages.mount \
    sys-kernel-debug.mount sys-fs-fuse-connections.mount \
    display-manager.service graphical.target systemd-logind.service

RUN useradd -m -G sudo -s /bin/bash docker
RUN echo 'docker:password' | chpasswd

RUN mkdir /home/docker/.ssh
ADD test/id_rsa.pub /home/docker/.ssh/authorized_keys
RUN chown -R docker:docker /home/docker/.ssh/
RUN chmod 701 /home/docker
RUN chmod 700 /home/docker/.ssh
RUN chmod 600 /home/docker/.ssh/authorized_keys

VOLUME [ "/sys/fs/cgroup" ]
VOLUME ["/run"]

EXPOSE 22
EXPOSE 1194

CMD ["/bin/systemd"]
