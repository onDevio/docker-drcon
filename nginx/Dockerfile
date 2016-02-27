FROM nginx:1.7

#Install Curl
RUN apt-get update -qq && apt-get -y install curl unzip

#Link to Consul Template Binary
ENV CT_ZIP consul-template_0.12.2_linux_amd64.zip
ENV CT_URL https://releases.hashicorp.com/consul-template/0.12.2/$CT_ZIP

#Install Consul Template
RUN curl -L $CT_URL -o /tmp/$CT_ZIP ; unzip -d /usr/local/bin /tmp/$CT_ZIP

#Setup Consul Template Files
RUN mkdir /etc/consul-templates
ENV CT_FILE /etc/consul-templates/nginx.conf

#Setup Nginx File
ENV NX_FILE /etc/nginx/conf.d/default.conf

#Default Variables
ENV CONSUL consul:8500
ENV SERVICE consul-8500

# Command will
# 1. Write Consul Template File
# 2. Start Nginx
# 3. Start Consul Template

CMD /usr/sbin/nginx -c /etc/nginx/nginx.conf \
& CONSUL_TEMPLATE_LOG=debug consul-template \
  -consul=$CONSUL \
  -template "$CT_FILE:$NX_FILE:/usr/sbin/nginx -s reload; cat /etc/nginx/conf.d/default.conf";
