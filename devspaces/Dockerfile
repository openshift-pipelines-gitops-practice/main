# OpenShift Dev Spaces 用: Universal Developer Image (RHEL 9) に OpenJDK 25 と Node.js 24 を追加する。
# ビルド例:
#   podman build -f devspaces/Dockerfile \
#     --build-arg UDI_TAG=3.16 \
#     -t image-registry.openshift-image-registry.svc:5000/<namespace>/udi-rhel9-java25-node24:latest .

ARG UDI_IMAGE=registry.redhat.io/devspaces/udi-rhel9
ARG UDI_TAG=latest

FROM ${UDI_IMAGE}:${UDI_TAG}

USER root

# Java 25（UBI 9 AppStream）、Node 24（module stream でベース UDI の Node 20 を置換）
RUN microdnf install -y dnf && \
    dnf -y install --setopt=tsflags=nodocs \
        java-25-openjdk \
        java-25-openjdk-devel \
        java-25-openjdk-headless \
    && dnf -y module reset nodejs \
    && dnf -y module install nodejs:24/development \
    && dnf -y clean all \
    && rm -rf /var/cache/yum /var/cache/dnf

ENV JAVA_HOME=/usr/lib/jvm/java-25-openjdk

# 親イメージは HOME=/home/user と ~/.java/current を前提に PATH を組んでいるため、
# tooling / user 両方の default JDK を 25 に合わせる（親と同様に JDK ルート直下を symlink）
RUN for base in /home/tooling /home/user; do \
      rm -rf "${base}/.java/current" && \
      mkdir -p "${base}/.java/current" && \
      for entry in /usr/lib/jvm/java-25-openjdk/*; do \
        ln -sfn "${entry}" "${base}/.java/current/$(basename "${entry}")"; \
      done && \
      chgrp -R 0 "${base}/.java" && chmod -R g=u,g+w "${base}/.java"; \
    done

RUN echo "========" \
    && echo -n "java (JAVA_HOME): " && "${JAVA_HOME}/bin/java" -version \
    && ls -la "${JAVA_HOME}/bin/java" \
    && echo "========" \
    && echo -n "node: " && node --version \
    && echo -n "npm: " && npm --version \
    && echo "========"

USER 10001
ENV HOME=/home/user
