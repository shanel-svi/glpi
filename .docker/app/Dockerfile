# ARM64-compatible build — replaces ghcr.io/glpi-project/glpi-development-env
# which is x86_64 only. Builds from the official PHP Apache image (multi-arch).

FROM composer:latest AS composer

FROM php:8.4-apache

LABEL org.opencontainers.image.title="GLPI development environment (ARM64)"

RUN apt-get update \
  \
  # Install bz2 extension (for marketplace).
  && apt-get install --assume-yes --no-install-recommends --quiet libbz2-dev \
  && docker-php-ext-install bz2 \
  \
  # Install exif extension.
  && docker-php-ext-install exif \
  \
  # Install GD PHP extension.
  && apt-get install --assume-yes --no-install-recommends --quiet libfreetype6-dev libjpeg-dev libpng-dev libwebp-dev \
  && docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
  && docker-php-ext-install gd \
  \
  # Install intl PHP extension.
  && apt-get install --assume-yes --no-install-recommends --quiet libicu-dev \
  && docker-php-ext-install intl \
  \
  # Install ldap PHP extension (use dpkg-architecture for cross-arch lib path).
  && apt-get install --assume-yes --no-install-recommends --quiet libldap2-dev \
  && docker-php-ext-configure ldap --with-libdir=lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH)/ \
  && docker-php-ext-install ldap \
  \
  # Install memcached PHP extension.
  && apt-get install --assume-yes --no-install-recommends --quiet libmemcached-dev \
  && pecl install memcached \
  && docker-php-ext-enable memcached \
  \
  # Install mysqli PHP extension.
  && docker-php-ext-install mysqli \
  \
  # Install bcmath PHP extension.
  && docker-php-ext-install bcmath \
  \
  # Install pcntl PHP extension.
  && docker-php-ext-install pcntl \
  \
  # Install redis PHP extension.
  && pecl install redis \
  && docker-php-ext-enable redis \
  \
  # Install Zip PHP extension.
  && apt-get install --assume-yes --no-install-recommends --quiet libzip-dev \
  && docker-php-ext-install zip \
  \
  # Install xdebug from source (3.5.0 not on PECL for PHP 8).
  && mkdir -p /tmp/xdebug \
  && curl --fail --silent --show-error --location \
       https://github.com/xdebug/xdebug/archive/refs/tags/3.5.0.tar.gz \
     | tar --extract --ungzip --directory="/tmp/xdebug" --strip 1 \
  && docker-php-ext-install /tmp/xdebug \
  && docker-php-ext-enable xdebug \
  && rm -rf /tmp/xdebug \
  \
  # Install XMLRPC PHP extension from source (PHP 8+).
  && apt-get install --assume-yes --no-install-recommends --quiet libxml2-dev \
  && mkdir -p /tmp/xmlrpc \
  && curl --fail --silent --show-error --location \
       https://github.com/php/pecl-networking-xmlrpc/archive/0f782ffe52cebd0a65356427b7ab72d48b72d20c/xmlrpc-0f782ff.tar.gz \
     | tar --extract --ungzip --directory="/tmp/xmlrpc" --strip 1 \
  && docker-php-ext-configure /tmp/xmlrpc --with-xmlrpc \
  && docker-php-ext-install /tmp/xmlrpc \
  && rm -rf /tmp/xmlrpc \
  \
  # Enable Apache mod_rewrite and mod_ssl.
  && a2enmod rewrite ssl \
  \
  # Install Node.js 22.x (ARM64-compatible via NodeSource).
  && apt-get install --assume-yes --no-install-recommends --quiet gnupg \
  && mkdir -p /etc/apt/keyrings \
  && curl --fail --silent --show-error --location \
       https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
     | gpg --dearmor --output /etc/apt/keyrings/nodesource.gpg \
  && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" \
     | tee /etc/apt/sources.list.d/nodesource.list \
  && apt-get update \
  && apt-get install --assume-yes --no-install-recommends --quiet nodejs \
  \
  # Install git, unzip (used by composer), gettext (translations), sudo, misc tools.
  && apt-get install --assume-yes --no-install-recommends --quiet \
       git unzip gettext sudo jq nano vim zsh \
  \
  # Clean up.
  && rm -rf /var/lib/apt/lists/*

# Copy composer binary from official image.
COPY --from=composer /usr/bin/composer /usr/bin/composer

# Use PHP development ini.
RUN cp "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"

# Copy Apache and PHP config files.
RUN mkdir /etc/apache2/vhosts
COPY ./files/etc/apache2/ports.conf /etc/apache2/ports.conf
COPY ./files/etc/apache2/vhosts/glpi-common.conf /etc/apache2/vhosts/glpi-common.conf
COPY ./files/etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/000-default.conf
COPY ./files/etc/apache2/sites-available/glpi-10.x.conf /etc/apache2/sites-available/glpi-10.x.conf
COPY ./files/etc/apache2/sites-available/glpi-ssl.conf /etc/apache2/sites-available/glpi-ssl.conf
COPY ./files/etc/php/conf.d/glpi.ini $PHP_INI_DIR/conf.d/glpi.ini

# Enable the SSL site.
RUN a2ensite glpi-ssl

# GLPI environment variables.
ENV GLPI_ENVIRONMENT_TYPE=development
ENV GLPI_URI=http://itsm.shanel.com:8080

# Allow extra PHP configuration via a volume-backed directory.
ENV PHP_INI_SCAN_DIR "/usr/local/etc/php/conf.d/:/usr/local/etc/php/custom_conf.d/"
RUN mkdir /usr/local/etc/php/custom_conf.d/

# Workaround to make apache use same UID/GID as host user.
ARG HOST_GROUP_ID=1000
RUN groupmod --gid ${HOST_GROUP_ID} www-data
ARG HOST_USER_ID=1000
RUN usermod --uid ${HOST_USER_ID} www-data

# Allow login as www-data user and give it passwordless sudo.
RUN usermod --shell /bin/bash www-data \
  && echo "www-data ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# Make www-data home persistent for cache.
RUN mkdir --parents /home/www-data \
  && chown www-data:www-data /home/www-data \
  && usermod --home /home/www-data www-data
VOLUME /home/www-data

USER www-data
WORKDIR /var/www/glpi
