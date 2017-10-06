FROM ubuntu:14.04

ARG uid
ARG gid

RUN groupadd -g $gid builder && useradd -m -u $uid -g $gid builder

USER builder
