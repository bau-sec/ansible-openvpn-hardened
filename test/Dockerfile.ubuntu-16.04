FROM ubuntu:16.04
ENV container docker

RUN apt-get update; apt-get -y upgrade

RUN DEBIAN_FRONTEND=noninteractive apt-get -q -y install systemd dbus openssh-server sudo openssh-client locales

# Uncomment to speed up local tests by pre-installing most required packages
#RUN DEBIAN_FRONTEND=noninteractive apt-get -q -y install aptitude iptables-persistent apt-transport-https ca-certificates python-software-properties unattended-upgrades debsums debsecan apt-listchanges libpam-pwquality aide-common
#RUN DEBIAN_FRONTEND=noninteractive apt-get -q -y install sudo python-pip python-virtualenv git gnupg iptables aide openssl dnsmasq auditd

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

RUN systemctl enable ssh.service

VOLUME [ "/sys/fs/cgroup" ]
VOLUME ["/run"]

EXPOSE 22
EXPOSE 1194

RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

CMD ["/bin/systemd"]
