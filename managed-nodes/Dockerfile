FROM almalinux:9

# 必要パッケージのインストール
RUN dnf install -y \
    openssh-server \
    python3 \
    sudo \
    passwd \
    && dnf clean all

# SSH起動に必要なホスト鍵を生成
RUN ssh-keygen -A

# rootパスワードを設定
RUN echo 'root:ansible' | chpasswd

# RHEL9系のSSH設定変更
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    echo 'PasswordAuthentication yes' >> /etc/ssh/sshd_config

# ansibleユーザーの作成(sudoを無制限に使用可能)
RUN useradd -m -s /bin/bash ansible && \
    echo 'ansible:ansible' | chpasswd && \
    echo 'ansible ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

EXPOSE 22

# sshdをsystemdで管理するために有効化
RUN systemctl enable sshd

CMD ["/sbin/init"]
