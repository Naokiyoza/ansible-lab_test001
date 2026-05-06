FROM almalinux:9

# Ansible本体とSSH接続に必要なツールをインストール
RUN dnf install -y \
    python3 \
    python3-pip \
    openssh-clients \
    sshpass \
    vim \
    iputils \
    && dnf clean all

# pipでAnsibleをインストール(最新版が取得できる)
RUN pip3 install ansible

# SSH設定(known_hostsの厳格チェックを無効化)
RUN mkdir -p /root/.ssh && \
    echo 'Host *' > /root/.ssh/config && \
    echo '    StrictHostKeyChecking no' >> /root/.ssh/config && \
    echo '    UserKnownHostsFile /dev/null' >> /root/.ssh/config

WORKDIR /ansible
CMD ["/bin/bash"]
