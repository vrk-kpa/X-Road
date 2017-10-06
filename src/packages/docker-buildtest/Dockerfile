FROM ubuntu:trusty

ARG uid
ARG gid

RUN groupadd -g $gid builder && useradd -m -u $uid -g $gid builder && adduser builder sudo

RUN echo '%sudo  ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/notty && chmod 0440 /etc/sudoers.d/notty

USER builder
