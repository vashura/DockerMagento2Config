# Establece versión PHP por defecto y usa multi-stage build
ARG PHP_VERSION=8.2
FROM php:${PHP_VERSION}-apache AS builder

# Argumentos obligatorios con valores por defecto
ARG COMPOSER_VERSION=2.6.5 
ARG APACHE_USER=jarvis
ARG USER_ID=1000
ARG APACHE_RUN_GROUP=${APACHE_USER}

LABEL maintainer="Ing. Valerio Lopez: valerio.lopez @outlook.com"

# Configuración de usuario y permisos
RUN adduser --uid ${USER_ID} --gecos 'Apache User' --disabled-password "${APACHE_USER}" && \
    chown -R "${APACHE_USER}:${APACHE_RUN_GROUP}" /var/lock/apache2 /var/run/apache2 /var/www/html

# Variables de entorno para Magento
ENV APACHE_RUN_USER=${APACHE_USER} \
    APACHE_RUN_GROUP=${APACHE_RUN_GROUP} \
    PHP_MEMORY_LIMIT=2G \
    PHP_UPLOAD_MAX_FILESIZE=64M \
    PHP_MAX_EXECUTION_TIME=1800

# Instalación de dependencias y extensiones PHP optimizadas para Magento
RUN apt-get update && apt-get install -y \
    nano \
    cron \
    gzip \
    libbz2-dev \
    libfreetype6-dev \
    libicu-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libwebp-dev \
    libsodium-dev \
    libssh2-1-dev \
    libxslt1-dev \
    libssl-dev \
    libonig-dev \
    libxml2-dev \
    libzip-dev \
    lsof \
    vim \
    zip \
    jq \
    unzip \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Configuración de extensiones PHP
RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp && \
    docker-php-ext-install -j$(nproc) \
    bcmath \
    bz2 \
    calendar \
    exif \
    gd \
    gettext \
    intl \
    mysqli \
    opcache \
    pdo_mysql \
    soap \
    sockets \
    sodium \
    xsl \
    zip \
    mbstring

# Configuración de Apache
COPY ./.apache/000-default.conf /etc/apache2/sites-available/000-default.conf
COPY ./.ssl/ /etc/apache2/ssl/
RUN a2enmod ssl rewrite headers && \
    a2ensite 000-default

# Instalación de Composer CORREGIDA
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && \
    php composer-setup.php --install-dir=/usr/local/bin --filename=composer --version=${COMPOSER_VERSION} && \
    php -r "unlink('composer-setup.php');" && \
    chmod +x /usr/local/bin/composer

# Configuración de Composer para Magento
RUN mkdir -p /home/${APACHE_USER}/.composer && \
    chown ${APACHE_USER}:${APACHE_RUN_GROUP} /home/${APACHE_USER}/.composer
COPY ./.docker/auth.json /home/${APACHE_USER}/.composer/auth.json
RUN chown ${APACHE_USER}:${APACHE_RUN_GROUP} /home/${APACHE_USER}/.composer/auth.json

# Configuración de MailHog
RUN curl -Lo /usr/local/bin/mhsendmail https://github.com/mailhog/mhsendmail/releases/download/v0.2.0/mhsendmail_linux_amd64 && \
    chmod +x /usr/local/bin/mhsendmail

# Configuración de Cron para Magento
RUN mkdir -p /var/log/cron && \
    touch /var/log/cron/magento.log
COPY ./.apache/crontab /etc/cron.d/magento
RUN chmod 0644 /etc/cron.d/magento && \
    chown ${APACHE_USER}:${APACHE_RUN_GROUP} /etc/cron.d/magento && \
    crontab -u ${APACHE_USER} /etc/cron.d/magento

# Script de inicio personalizado
COPY ./.docker/docker-php-entrypoint /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-php-entrypoint && \
    sed -i 's/^exec /service cron start\n\nexec /' /usr/local/bin/apache2-foreground

ENTRYPOINT ["docker-php-entrypoint"]
CMD ["apache2-foreground"]

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost/ || exit 1