FROM quay.io/alexcheng1982/apache2-php7:7.3.12

LABEL maintainer="alexcheng1982@gmail.com"
LABEL php_version="7.3.12"
LABEL magento_version="2.3.3"
LABEL description="Magento 2.3.3 with PHP 7.3.12"

ENV MAGENTO_VERSION 2.3.3
ENV INSTALL_DIR /var/www/html
ENV COMPOSER_HOME /var/www/.composer/
ARG APCU_VERSION=5.1.11

#Docker doesn't initialise each RUN step as a full login shell might, it's creates a pretty minimal environment.
#The umask is set to 0022 by runc when a container is started which is also not configurable.
#So, like most Docker builds, you may need to chain the commands into one RUN step so it's all under the same shell process.

RUN curl -sS https://getcomposer.org/installer | php && mv composer.phar /usr/local/bin/composer

COPY ./auth.json $COMPOSER_HOME

RUN requirements="libpng++-dev git-core libzip-dev libmcrypt-dev libmcrypt4 libcurl3-dev libfreetype6 libjpeg-turbo8 libjpeg-turbo8-dev libfreetype6-dev libicu-dev libxslt1-dev unzip" \
    && apt-get update \
    && apt-get install -y $requirements \
    && rm -rf /var/lib/apt/lists/* \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install gd \
    && docker-php-ext-install mbstring \
    && docker-php-ext-install zip \
    && docker-php-ext-install intl \
    && docker-php-ext-install xsl \
    && docker-php-ext-install soap \
    && docker-php-ext-install bcmath \
	&& docker-php-ext-install opcache

# Use the default production configuration
#RUN mv "/usr/local/etc/php/php.ini-production" "/usr/local/etc/php/php.ini"

#RUN docker-php-ext-configure opcache --enable-opcache

RUN yes '' | pecl install mcrypt-1.0.3 \
    && echo 'extension=mcrypt.so' > /usr/local/etc/php/conf.d/mcrypt.ini

#INSTALL XDEBUG
RUN pecl install xdebug-2.9.5 && docker-php-ext-enable xdebug
#XDEBUG

#INSTALL APCU
RUN pecl install apcu-${APCU_VERSION} && docker-php-ext-enable apcu
#APCU

#speed up composer by enabling paralel download
RUN composer global require hirak/prestissimo
#

RUN chsh -s /bin/bash www-data

RUN chown -R www-data:www-data /var/www
#umask 0007 is 770 for directories and 660 for files
RUN umask 0007 && \
    su www-data -c "composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition $INSTALL_DIR $MAGENTO_VERSION" \
	&& chmod u+x bin/magento

#this takes way too long on Windows (Linux too actually)
#perhaps setting umask before adding data makes this obsolete
#RUN cd $INSTALL_DIR \
#    && find . -type d -exec chmod 770 {} \; \
#    && find . -type f -exec chmod 660 {} \; \
#    && chmod u+x bin/magento
#RUN cd $INSTALL_DIR && chmod -R 770 .

COPY ./ops $INSTALL_DIR/ops
RUN cd $INSTALL_DIR/ops && chmod -R 770 . && chown -R www-data:www-data .

COPY ./install-magento /usr/local/bin/install-magento
RUN chmod +x /usr/local/bin/install-magento

COPY ./install-sampledata /usr/local/bin/install-sampledata
RUN chmod +x /usr/local/bin/install-sampledata

RUN a2enmod rewrite
RUN echo "memory_limit=2048M" > /usr/local/etc/php/conf.d/memory-limit.ini

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

#initialize optimizers at the end
RUN { \
	echo 'zend_extension=opcache.so'; \
	echo 'opcache.enable = 1'; \
	echo 'opcache.enable_cli=1'; \
	echo 'opcache.memory_consumption=1024M'; \
	echo 'opcache.interned_strings_buffer=64'; \
	echo 'opcache.max_accelerated_files=60000';\
	echo 'opcache.save_comments=1'; \
	echo 'opcache.enable_file_override=1'; \
	echo 'opcache.consistency_checks=0'; \
	echo 'opcache.fast_shutdown=1'; \
	} > /usr/local/etc/php/conf.d/opcache-recommended.ini 

#these opcache settings can better be added after installation:
#echo 'opcache.revalidate_freq=0'; \
#echo 'opcache.validate_timestamps=0'; \

RUN { \
	echo 'extension=apcu.so'; \
	echo 'apc.enable = 1'; \
	echo 'apc.enable_cli=1'; \
	} > /usr/local/etc/php/conf.d/apcu.ini 

WORKDIR $INSTALL_DIR

# Add cron job
ADD crontab /etc/cron.d/magento2-cron
RUN chmod 0644 /etc/cron.d/magento2-cron \
    && crontab -u www-data /etc/cron.d/magento2-cron

VOLUME $INSTALL_DIR