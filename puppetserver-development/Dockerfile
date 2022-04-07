#
# BASE IMAGE
ARG BASE_REGISTRY=registry1.dsop.io
ARG BASE_IMAGE=ironbank/redhat/openjdk/openjdk8
ARG BASE_TAG=1.8.0
FROM ${BASE_REGISTRY}/${BASE_IMAGE}:${BASE_TAG}

#
# LABEL
#
LABEL name="Puppet Server" \
      maintainer="jweatherford@oteemo.com" \
      vendor="Puppet Labs" \
      version="6.13.0" \
      release="1" \
      summary="Puppet Server Image" \
      description="Puppet Server Image"


COPY puppetserver-6.13.0-1.el8.noarch.rpm \
     puppetdb-6.12.0-1.el8.noarch.rpm \
     puppet-agent-6.18.0-1.el8.x86_64.rpm \
     puppetdb-termini-6.12.0-1.el8.noarch.rpm \
     dumb-init-1.2.2-6.el8.x86_64.rpm /opt/
COPY *.gem /opt/

USER root
RUN yum update -y && \
    yum -y upgrade && \
    yum -y install java-1.8.0-openjdk-headless net-tools procps && rpm -Uvh /opt/puppet*.rpm


ENV PUPPETSERVER_JAVA_ARGS="-Xms512m -Xmx512m" \
    PATH=/opt/puppetlabs/server/bin:/opt/puppetlabs/puppet/bin:/opt/puppetlabs/bin:$PATH \
    SSLDIR=/etc/puppetlabs/puppet/ssl \
    LOGDIR=/var/log/puppetlabs/puppetserver \
    PUPPETSERVER_HOSTNAME="" \
    DNS_ALT_NAMES="" \
    PUPPET_MASTERPORT=8140 \
    AUTOSIGN="" \
    PUPPETSERVER_MAX_ACTIVE_INSTANCES=1 \
    PUPPETSERVER_MAX_REQUESTS_PER_INSTANCE=0 \
    CA_ENABLED=true \
    CA_HOSTNAME=puppet \
    CA_MASTERPORT=8140 \
    CA_ALLOW_SUBJECT_ALT_NAMES=false \
    USE_PUPPETDB=true \
    PUPPETDB_SERVER_URLS=https://puppetdb:8081 \
    PUPPET_STORECONFIGS_BACKEND="puppetdb" \
    PUPPET_STORECONFIGS=true \
    PUPPET_REPORTS="puppetdb"


COPY scripts/docker-entrypoint.sh \
     scripts/healthcheck.sh \
      /


RUN mkdir /docker-entrypoint.d && \
    chmod 0755 /docker-entrypoint.d
COPY scripts/10-analytics.sh scripts/20-use-templates-initially.sh scripts/30-set-permissions.sh \
     scripts/40-update-puppetdb-conf.sh scripts/50-set-certname.sh scripts/55-set-masterport.sh \
     scripts/60-setup-autosign.sh scripts/70-set-dns-alt-names.sh scripts/80-ca.sh \
     scripts/85-setup-storeconfigs.sh scripts/90-log-config.sh /docker-entrypoint.d/


RUN chmod +x /docker-entrypoint.sh /healthcheck.sh 



RUN gem install --local --no-rdoc --no-ri --ignore-dependencies /opt/*.gem && puppet config set autosign true --section master
RUN cp -pr /etc/puppetlabs/puppet /var/tmp && cp -pr /opt/puppetlabs/server/data/puppetserver /var/tmp
RUN rm -rf /var/tmp/puppet/ssl

COPY scripts/puppetserver /etc/default/puppetserver
COPY scripts/logback.xml \
     scripts/request-logging.xml \
     /etc/puppetlabs/puppetserver/
COPY scripts/puppetserver.conf /etc/puppetlabs/puppetserver/conf.d/
COPY scripts/puppetdb.conf /var/tmp/puppet/


#############################################


# CLEANUP

RUN yum clean all && rm -rf /var/cache/yum && rm -rf /opt/*.rpm && rm -rf /opt/*.gem \
    && find /opt/puppetlabs/puppet/lib/ruby/gems/2.5.0/gems/httpclient-2.8.3/ -name "*.key" | xargs rm -f \
    && find /opt/puppetlabs/puppet/lib/ruby/gems/2.5.0/gems/httpclient-2.8.3/ -name "*.crt" | xargs rm -f \
    && find /opt/puppetlabs/puppet/lib/ruby/gems/2.5.0/gems/httpclient-2.8.3/ -name "*.pem" | xargs rm -f

EXPOSE 8140

USER puppet

# To ensure all the scripts are executable by the puppet user
ENTRYPOINT ["dumb-init", "/docker-entrypoint.sh"]
CMD ["foreground"]

# k8s uses livenessProbe, startupProbe, readinessProbe and ignores HEALTHCHECK
HEALTHCHECK --interval=10s --timeout=15s --retries=12 --start-period=3m CMD ["/healthcheck.sh"]
