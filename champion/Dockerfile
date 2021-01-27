FROM alpine:3.11

### INSTALL PACKAGES ###

# Add `--repository http://dl-cdn.alpinelinux.org/alpine/edge/community` for 7.4
RUN apk --no-cache add \
  apache2 \
  apache2-proxy \
  bash \
  curl \
  git \
  mysql-client \
  openssh \
  patch \
  php7 \
  php7-bcmath \
  php7-curl \
  php7-dom \
  php7-fpm \
  php7-gd \
  php7-gmp \
  php7-intl \
  php7-json \
  php7-mbstring \
  php7-mysqli \
  php7-opcache \
  php7-openssl \
  php7-pdo \
  php7-pdo_mysql \
  php7-phar \
  php7-sockets \
  php7-soap \
  php7-tokenizer \
  php7-xml \
  php7-xmlreader \
  php7-xmlwriter \
  php7-xsl \
  php7-zip \
  php7-zlib \
  php7-pecl-mcrypt \
  php7-pecl-redis \
  php7-pecl-xdebug \
  ssmtp \
  supervisor \
  tidyhtml


### APACHE ###

# Enable vhosts and fcgi proxy
RUN sed -ri \
  -e 's!^#(LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so)!\1!g' \
  -e 's!^#(LoadModule proxy_module modules/mod_proxy.so)!\1!g' \
  -e 's!^#ServerName.*!ServerName localhost!g' \
  -e '/LoadModule rewrite_module/s/^#//g' \
  -e '/LoadModule alias_module/s/^#//g' \
  -e '/LoadModule deflate_module/s/^#//g' \
  -e '/LoadModule expires_module/s/^#//g' \
  /etc/apache2/httpd.conf


### PHP ###

ENV TZ=Australia/Melbourne

RUN echo $'[Date]\ndate.timezone = "Australia/Melbourne"' >> /etc/php7/conf.d/timezone.ini \
    && echo "mailhub=mailhog:1025\nUseTLS=NO\nFromLineOverride=YES" > /etc/ssmtp/ssmtp.conf \
    && echo "sendmail_path = /usr/sbin/ssmtp -t" > /etc/php7/conf.d/sendmail.ini \
    && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone \
    && mkdir -p /var/log/php


### PHP CLI TOOLS ###

ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_HOME "/usr/local/composer"
ENV COMPOSER_VERSION 1.8.0
ENV PATH "${COMPOSER_HOME}/vendor/bin:${PATH}"

RUN mkdir -p $COMPOSER_HOME \
  && php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" \
  && php composer-setup.php --1 --install-dir /usr/bin --filename=composer \
  && php -r "unlink('composer-setup.php');" \
  && composer global require phing/phing

RUN mkdir -p /var/www/.ssh && \
    chmod 700 /var/www/.ssh && \
    ssh-keyscan github.com >> /var/www/.ssh/known_hosts && \
    ssh-keyscan bitbucket.org >> /var/www/.ssh/known_hosts && \
    ssh-keyscan gitlab.com >> /var/www/.ssh/known_hosts  && \
    ssh-keyscan gitlab.symbiote.com.au >> /var/www/.ssh/known_hosts  && \
    chown -R nobody.nobody /var/www/.ssh && \
    chmod 600 /var/www/.ssh/known_hosts

### BRING IT UP ###

COPY config/vhost.conf    /etc/apache2/conf.d/httpd-vhosts.conf

COPY config/fpm-pool.conf /etc/php7/php-fpm.d/www.conf
COPY config/php.ini       /etc/php7/conf.d/custom.ini
COPY config/memory.ini    /etc/php7/conf.d/memory.ini
COPY config/xdebug.ini    /etc/php7/conf.d/xdebug.ini

COPY scripts/sspak.phar   /usr/local/bin/sspak

# Configure supervisord
COPY config/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

RUN mkdir /var/www/html && \
  chown -R nobody.nobody /var/www/html && \
  chown -R nobody.nobody /run && \
  chmod +x /usr/local/bin/sspak

# Switch to use a non-root user from here on
USER nobody

# Add application
WORKDIR /var/www/html

# Expose the port apache is reachable on
EXPOSE 80

# Let supervisord start nginx & php-fpm
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]

# Configure a healthcheck to validate that everything is up&running
HEALTHCHECK --timeout=10s CMD curl --silent --fail http://127.0.0.1:80/fpm-ping
