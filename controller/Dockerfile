FROM almalinux:9

# Ansible本体とSSH接続に必要なツールをインストール
RUN dnf install -y \
    python3 \
    python3-pip \
    openssh-clients \
    sshpass \
    vim \
    iputils \
    git \
    rsync \
    dnf-plugins-core \
    && dnf clean all

# Docker CLI をインストール (Molecule から host docker daemon を操作するため)
# socket 経由で host の docker と通信するので daemon は不要、CLI のみ
RUN dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo && \
    dnf install -y docker-ce-cli && \
    dnf clean all

# pip で Ansible + Molecule + 関連ツールをインストール
# - ansible: Ansible本体
# - molecule: ロールテストフレームワーク
# - molecule-plugins[docker]: Docker driver(別パッケージ化された)
# - docker: PythonのDocker SDK
# - ansible-lint / yamllint: 静的解析
# - pytest-testinfra: 検証用ライブラリ(オプション)
RUN pip3 install --upgrade pip && \
    pip3 install \
    ansible \
    molecule \
    'molecule-plugins[docker]' \
    'docker>=7.1.0' \
    'requests<2.32' \
    ansible-lint \
    yamllint \
    pytest-testinfra

# SSH設定(known_hostsの厳格チェックを無効化)
RUN mkdir -p /root/.ssh && \
    echo 'Host *' > /root/.ssh/config && \
    echo '    StrictHostKeyChecking no' >> /root/.ssh/config && \
    echo '    UserKnownHostsFile /dev/null' >> /root/.ssh/config

WORKDIR /ansible
CMD ["/bin/bash"]
