# Base image is the Official HAProxy
FROM haproxy:1.5

# TODO: [GEN] Replace with your name and email
MAINTAINER Laurent Prevost <laurent.prevost@heig-vd.ch>

# Install some tools
# TODO: [HB] Update to install required tool to install NodeJS
RUN apt-get update && apt-get -y install wget curl vim iputils-ping rsyslog xz-utils

# [S6] Install
# Download and install S6 overlay
RUN curl -sSLo /tmp/s6.tar.gz https://github.com/just-containers/s6-overlay/releases/download/v1.17.2.0/s6-overlay-amd64.tar.gz \
  && tar xzf /tmp/s6.tar.gz -C / \
  && rm -f /tmp/s6.tar.gz

# Install serf (for decentralized cluster membership: https://www.serf.io/)
RUN mkdir /opt/bin \
    && curl -sSLo /tmp/serf.gz https://releases.hashicorp.com/serf/0.7.0/serf_0.7.0_linux_amd64.zip \
    && gunzip -c /tmp/serf.gz > /opt/bin/serf \
    && chmod 755 /opt/bin/serf \
    && rm -f /tmp/serf.gz

# TODO: [HB] Install NodeJS
RUN curl -sSLo /tmp/node.tar.xz https://nodejs.org/dist/v4.4.4/node-v4.4.4-linux-x64.tar.xz \
  && tar -C /usr/local --strip-components 1 -xf /tmp/node.tar.xz \
  && rm -f /tmp/node.tar.xz

# TODO: [HB] Install Handlebars
# Install the handlebars-cmd node module and its dependencies
RUN npm install -g handlebars-cmd

# [S6] Replace the two following instructions
# Copy the S6 service and make the run script executable
COPY services/ha /etc/services.d/ha
RUN chmod +x /etc/services.d/ha/run

# TODO: [Serf] Add Serf S6 setup
COPY services/serf /etc/services.d/serf
RUN chmod +x /etc/services.d/serf/run

# TODO: [Serf] Copy events handler scripts
COPY scripts/member-join.sh /serf-handlers/member-join.sh
RUN chmod +x /serf-handlers/member-join.sh

COPY scripts/member-leave.sh /serf-handlers/member-leave.sh
RUN chmod +x /serf-handlers/member-leave.sh

# Copy the haproxy and syslog config
COPY config/haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
COPY config/rsyslogd.cfg /etc/rsyslog.d/49-haproxy.conf

# TODO: [HB] Copy the haproxy configuration template
COPY config/haproxy.cfg.hb /config/haproxy.cfg.hb

# Clean up
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# TODO: [CFG] Create the nodes folder
RUN mkdir /nodes

# TODO: [Serf] Expose ports
EXPOSE 7946 7373

# Expose the HA proxy ports
EXPOSE 80 1936

# Define an environment variable
ENV ROLE balancer

# [S6] Replace the following instruction
# This will start S6 as our main process in our container
ENTRYPOINT ["/init"]
