FROM php:8.3-apache

ARG COMPOSER_AUTH
ENV COMPOSER_AUTH=${COMPOSER_AUTH}

COPY . /var/www/html
COPY .docker/php/vhost.conf /etc/apache2/sites-available/000-default.conf
COPY .docker/php/apache.ini $PHP_INI_DIR/conf.d/

WORKDIR /var/www/html

RUN apt-get update \
  && apt-get install -y --no-install-recommends apt-utils curl gnupg gpg locales apt-transport-https zlib1g-dev libzip-dev unzip libcurl4-openssl-dev pkg-config libssl-dev libxml2-dev

RUN apt-get update && apt-get install -y \
  # software-properties-common \
  npm

RUN apt-get update \
  && mkdir -p /etc/apt/keyrings \
  && curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /etc/apt/keyrings/microsoft.gpg \
  && chmod go+r /etc/apt/keyrings/microsoft.gpg \
  && echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/microsoft.gpg] https://packages.microsoft.com/debian/12/prod bookworm main" > /etc/apt/sources.list.d/mssql-release.list \
  && echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && locale-gen \
  && apt-get update \
  && ACCEPT_EULA=Y apt-get install -y --no-install-recommends unixodbc-dev msodbcsql18 mssql-tools18 \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN apt-get update && \
  apt-get install libldb-dev libldap2-dev -y && \
  docker-php-ext-install ldap

RUN apt-get update && apt-get install -y \
  libfreetype6-dev \
  libjpeg62-turbo-dev \
  libpng-dev \
  && docker-php-ext-configure gd \
  && docker-php-ext-install -j$(nproc) gd

RUN docker-php-ext-install mysqli pdo_mysql zip xml pcntl

RUN apt-get update && apt-get install -y apt-transport-https

RUN pecl install sqlsrv pdo_sqlsrv && \
  echo "extension=sqlsrv.so" >> /usr/local/etc/php/conf.d/sqlsrv.ini && \
  echo "extension=pdo_sqlsrv.so" >> /usr/local/etc/php/conf.d/sqlsrv.ini

RUN apt-get autoremove -y && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN usermod -u 1000 www-data

RUN a2enmod ssl 
RUN a2enmod proxy
RUN a2enmod proxy_http
RUN a2enmod proxy_wstunnel
RUN a2enmod headers
RUN a2enmod rewrite
RUN service apache2 restart

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer && composer install --no-interaction

RUN apt-get update && apt-get install -y \
    nano \
    supervisor \
    cron

# RUN npm install && npm run build

COPY .docker/supervisor/laravel-worker.conf /etc/supervisor/conf.d/laravel-worker.conf
COPY .docker/supervisor/reverb.conf /etc/supervisor/conf.d/reverb.conf
COPY .docker/php/entrypoint.sh /usr/local/bin/entrypoint.sh
COPY .docker/php/health-check.sh /usr/local/bin/health-check.sh

RUN mkdir -p /var/logs/supervisor
RUN chown root:root /etc/supervisor/conf.d/laravel-worker.conf && \
    chown root:root /etc/supervisor/conf.d/reverb.conf && \
    chmod +x /usr/local/bin/entrypoint.sh && \
    chmod +x /usr/local/bin/health-check.sh

RUN chown -R www-data:www-data /var/www/html
RUN find /var/www/html -type d -exec chmod 775 {} \;
RUN find /var/www/html -type f -exec chmod 0664 {} \;
RUN chgrp -R www-data storage /var/www/html/bootstrap/cache
RUN chmod -R ug+rwx storage /var/www/html/bootstrap/cache

RUN chmod o+w ./storage/ -R
RUN chown www-data:www-data -R ./storage

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
