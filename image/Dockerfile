FROM httpd:2.4.54

# apache2: to have the apache web server able to authenticate and to reverse proxy the real application
# libapache2-mod-shib: to have mod_shib for apache and the shibd daemon
# vim: for debug
RUN apt-get update \
  && apt-get -y install vim apache2 libapache2-mod-shib \
  && apt-get clean \
  && apt-get -y autoremove

# ssl: for apache beeing able to listen with HTTPS (SGY 10/06/2022: WHY is it mandatory to have HTTPS ?)
# shib: the famouse apache mod for beeing able to authenticate users thanks to shibboleth and SAML
# proxy_http: for apache beeing able to proxy user's HTTP requests to the real application
# remoteip: for apache beeing able to log real IP addresses instead of the above reverse proxy one
RUN sed -i \
  -e 's/^#\(LoadModule .*mod_ssl.so\)/\1/' \
  -e 's/^#\(LoadModule .*mod_socache_shmcb.so\)/\1/' \
  -e '/LoadModule proxy_module/s/^#//g' \
  -e '/LoadModule proxy_http_module/s/^#//g' \
  -e '/LoadModule remoteip_module/s/^#//g' \
  -e '/LoadModule rewrite_module/s/^#//g' \
  -e '/LoadModule rewrite_module/a LoadModule mod_shib /usr/lib/apache2/modules/mod_shib.so' \
  -e '/LoadModule macro_module/s/^#//g' \
  -e '/LoadModule headers_module/s/^#//g' \
  -e '/httpd-vhosts.conf/s/^#//g' \
  /usr/local/apache2/conf/httpd.conf

# apache conf and shib config
# (templates in order to be able to customize it fro external parameters)
COPY ./httpd-vhosts.conf /usr/local/apache2/conf/extra/httpd-vhosts.conf.orig
RUN touch /usr/local/apache2/conf/extra/httpd-vhosts.inc.conf
RUN touch /usr/local/apache2/conf/extra/httpd-vhosts.public_proxy.inc.conf
RUN touch /usr/local/apache2/conf/extra/httpd-vhosts.protected_proxy.inc.conf
COPY ./shibboleth/ /etc/shibboleth/

# redirect apache logs to docker stdout/stderr
RUN ln -sf /proc/1/fd/1 /var/log/apache2/access.log
RUN ln -sf /proc/1/fd/2 /var/log/apache2/error.log
RUN touch /var/log/apache2/access-rotated.log

# creates the entrypoint of this container
COPY ./httpd-foreground /usr/local/bin/
CMD [ "httpd-foreground" ]

EXPOSE 443