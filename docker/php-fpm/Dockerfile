FROM  centos:latest

#install enviroment

ENV PHP_DEPENDS gcc gcc-c++ autoconf libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel zlib zlib-devel glibc glibc-devel glib2 glib2-devel bzip2 bzip2-devel ncurses ncurses-devel curl curl-devel e2fsprogs e2fsprogs-devel krb5 krb5-devel libidn libidn-devel openssl openssl-devel openldap openldap-devel nss_ldap openldap-clients openldap-servers gd gd2 gd-devel gd2-devel perl-CPAN pcre-devel libmcrypt libmcrypt-devel readline-devel recode-devel git unzip

ENV PHP_INI_DIR /usr/local/php/etc
ENV PHP_VERSION 5.6.23
ENV PHP_FILENAME php-5.6.23.tar.gz
ENV PHP_SHA256 5f2274a13970887e8c81500c2afe292d51c3524d1a06554b0a87c74ce0a24ffe
ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --enable-pcntl --enable-mysqlnd --enable-opcache --enable-sockets --enable-sysvmsg --enable-sysvsem  --enable-sysvshm --enable-shmop --enable-zip --enable-ftp --enable-soap --enable-xml --enable-mbstring --disable-rpath --disable-debug --disable-fileinfo --with-mysql=mysqlnd --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-pcre-regex --with-iconv --with-zlib --with-mcrypt --with-openssl --with-mhash --with-xmlrpc --with-curl --with-imap-ssl 

ENV GPG_KEYS 0BD78B5F97500D450838F95DFE857D9A90D90EC1 6E4F6AB321FDC07F2C332E3AC2BF0BC433CFC8B3

RUN set -xe \
	&& for key in $GPG_KEYS; do \
		gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
	done

RUN yum -y localinstall http://rpms.famillecollet.com/enterprise/remi-release-7.rpm \
	&& yum -y install $PHP_DEPENDS

RUN curl -fSL "http://cn2.php.net/distributions/$PHP_FILENAME" -o "$PHP_FILENAME" \
	&& echo "$PHP_SHA256 *$PHP_FILENAME" | sha256sum -c - \
	&& curl -fSL "http://php.net/get/$PHP_FILENAME.asc/from/this/mirror" -o "$PHP_FILENAME.asc" \
	&& gpg --verify "$PHP_FILENAME.asc" \
	&& mkdir -p /usr/src/php \
	&& tar -zxf "$PHP_FILENAME" -C /usr/src/php --strip-components=1 \
	&& rm "$PHP_FILENAME"* 

RUN cd /usr/src/php \
	&& ./configure \
		--with-config-file-path="$PHP_INI_DIR" \
		--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
		$PHP_EXTRA_CONFIGURE_ARGS \
		--enable-mysqlnd \
		--with-curl \
		--with-openssl \
		--with-readline \
		--with-recode \
		--with-zlib \
	&& make -j"$(nproc)" \
	&& make install \
	&& make clean 

RUN mkdir $PHP_INI_DIR \
	&& mkdir "$PHP_INI_DIR/conf.d" \
	&& cp /usr/src/php/php.ini-production "$PHP_INI_DIR/php.ini" \
	&& cp /usr/local/etc/php-fpm.conf.default /usr/local/etc/php-fpm.conf

#extension 

#gd
RUN cd /usr/src/php/ext/gd \
	&& phpize \
	&& ./configure --prefix=/usr/local/gd --with-jpeg-dir=/usr/local --with-png-dir=/usr/local --with-freetype-dir=/usr/local \
	&& make && make install \
	&& echo "extension = gd.so" |tee "$PHP_INI_DIR/conf.d/gd.ini" 

RUN curl -fSL "http://pecl.php.net/get/redis-2.2.8.tgz" -o "redis-2.2.8.tgz" \
	&& tar zxf redis-2.2.8.tgz \
	&& rm -Rf  redis-2.2.8.tgz\
	&& cd redis-2.2.8 \
	&& phpize \
	&& ./configure --with-php-config=/usr/local/bin/php-config \
	&& make && make install \
	&& echo "extension = redis.so" |tee "$PHP_INI_DIR/conf.d/redis.ini" \
	&& rm -Rf  /redis-2.2.8/

#phalcon
RUN	curl -fSL "https://codeload.github.com/phalcon/cphalcon/zip/master" -o "phalcon.zip" \
	&& unzip phalcon.zip \
	&& cd cphalcon-master/build/ \
	&& ./install \
	&& echo "extension = phalcon.so" |tee "$PHP_INI_DIR/conf.d/phalcon.ini" \
	&& rm -Rf /cphalcon-master/

RUN set -ex \
	&& cd /usr/local/etc \
	&& if [ -d php-fpm.d ]; then \
		# for some reason, upstream's php-fpm.conf.default has "include=NONE/etc/php-fpm.d/*.conf"
		sed 's!=NONE/!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
		cp php-fpm.d/www.conf.default php-fpm.d/www.conf; \
	else \
		# PHP 5.x don't use "include=" by default, so we'll create our own simple config that mimics PHP 7+ for consistency
		mkdir php-fpm.d; \
		cp php-fpm.conf.default php-fpm.d/www.conf; \
		{ \
			echo '[global]'; \
			echo 'include=etc/php-fpm.d/*.conf'; \
		} | tee php-fpm.conf; \
	fi \
	&& { \
		echo '[global]'; \
		echo 'error_log = /proc/self/fd/2'; \
		echo; \
		echo '[www]'; \
		echo '; if we send this to /proc/self/fd/1, it never appears'; \
		echo 'access.log = /proc/self/fd/2'; \
		echo; \
		echo 'clear_env = no'; \
		echo; \
		echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
		echo 'catch_workers_output = yes'; \
	} | tee php-fpm.d/docker.conf \
	&& { \
		echo '[global]'; \
		echo 'daemonize = no'; \
		echo; \
		echo '[www]'; \
		echo 'listen = [::]:9000'; \
	} | tee php-fpm.d/zz-docker.conf

RUN sed -i 's/variables_order = "GPCS"/variables_order = "EGPCS"/g' "$PHP_INI_DIR/php.ini"

EXPOSE 9000
CMD ["php-fpm"]