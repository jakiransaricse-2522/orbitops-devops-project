FROM jenkins/jenkins:lts-jdk21
USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl gnupg nodejs npm python3 python3-pip python3-venv git \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
 && chmod a+r /etc/apt/keyrings/docker.asc \
 && . /etc/os-release \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian ${VERSION_CODENAME} stable" > /etc/apt/sources.list.d/docker.list \
 && apt-get update && apt-get install -y --no-install-recommends docker-ce-cli \
 && python3 -m venv /opt/ansible-venv \
 && /opt/ansible-venv/bin/pip install --no-cache-dir ansible-core docker \
 && /opt/ansible-venv/bin/ansible-galaxy collection install community.docker \
 && ln -s /opt/ansible-venv/bin/ansible /usr/local/bin/ansible \
 && ln -s /opt/ansible-venv/bin/ansible-playbook /usr/local/bin/ansible-playbook \
 && rm -rf /var/lib/apt/lists/*
USER jenkins
