Motivation
==========

The  main motivation for this patch is to address the problem of segfaults in PHP 8.x when opening dlopen() plugins with
the  RTLD_DEEPBIND  flag,  introduced  under  a  far-fetched  pretext,  which prevents the use of any custom allocators,
including in cases where their use is obviously desirable.

This  flag,  the use of which is strongly discouraged in principle, prevents the forwarding of LD_PRELOAD interpositions
to loaded modules. Which leads to an immediate segfault in case of preloading an allocator other than the system one.

The  original  idea  was  taken from here https://github.com/php/php-src/issues/10670, but the diff file is difficult to
use. For this reason, it was converted to a patch format suitable for use on all PHP 8.x subversions.

Unfortunately,  there  is no easy way to solve the RTLD_DEEPBIND flag problem other than rebuilding the patched PHP from
source and using it instead of the versions from the repositories.

How to make it so
=================

Preparation
-----------

Get current PHP config from installed binary:

php -i | grep 'Configure Command'

Then download the tarball and unzip it. Put the patch on top of your source tree and apply it:

patch -p0 < php8.x_deepbind_on_off.patch


Configuration and build
-----------------------

Configuration template for 64-bit build with GCC (example, configure PHP options in accordance with the configuration of
the installed build from the repository):

./configure 'CFLAGS=-O3 -m64 -s'

Note: Do NOT use -flto, it is incompatible with PHP.

Example for libphp.so and mysql support (zabbix 7.x+ with Apache2):

./configure 'CFLAGS=-O3 -m64 -s' \
--with-apxs2=/usr/bin/apxs \
--with-mysqli \
--with-pdo-mysql \
--enable-mbstring \
--enable-sockets \
--enable-session \
--with-gettext \
--with-libxml \
--with-zlib \
--with-openssl \
--with-curl \
--enable-fpm \
--with-bz2 \
--with-xsl \
--enable-ctype \
--enable-fileinfo \
--enable-bcmath \
--enable-gd \
--with-webp \
--with-jpeg \
--with-xpm \
--with-freetype

Then

make && make install

Note:  DO  NOT REMOVE the PHP 8.x version installed from the repositories, to avoid breaking dependencies. DO NOT remove
the  installed  version  of PHP 8.x from the repositories, to avoid breaking dependencies. Do not modify package version
files,  as  they  may  be  overwritten  by  updates,  and  do  not set immutable flags. This may lead to difficulties in
maintaining the configuration.

Switch alternatives
-------------------

Example for PHP 8.2:

update-alternatives --display php

update-alternatives --remove php /usr/bin/php8.2

update-alternatives --install /usr/bin/php php /usr/local/bin/php 200

update-alternatives --auto php

update-alternatives --display php

Make sure php is calling the correct version from /usr/local.

Using custom build
------------------

By default, if no prefix is specified, PHP will be built and installed in /usr/local.

The php.ini template, if the configuration path has not been customized, will be placed in /usr/local/etc.

It  must  be brought into line with the PHP configuration of the installed package version, adjusting the loaded plugins
if  necessary,  and,  in  addition,  find the new zend.dlopen_deepbind option in it and disable it (set it to Off; it is
enabled by default).

If a workaround with libC preload was used for the web server and PHP scripts, it must be disabled.

Now you can safely preload a custom allocator globally or per-service.

Final words about Zabbix
========================

For  Zabbix,  doing  all of the above is not enough. It contains a PHP session cleaning service phpsessionclean.service,
initiated by a timer. Unfortunately, the path to the PHP executable is hardcoded in the service script.

A  supported  solution  is  to  create  an alternative script /usr/lib/php/sessionclean2 (located in the phpsessionclean
directory)  and create an alternative method and timer (located there). After that, the original phpsessionclean.service
and phpsessionclean.timer can be disabled. Do not delete them.
