Performance Tips
================

The performance of ownCloud, like any `LAMP application <http://wikipedia.org/wiki/LAMP_%28software_bundle%29>`_,
is dependent on all components of the stack.
Maximizing performance can be achieved by optimizing the operations and interactions
of the underlying network, hardware, operating systems, webservers, databases, and storage.

This guide cannot cover all possible configurations and will instead
cover tips that are specific to ownCloud or give the greatest benefit.

SSL / Encryption App
--------------------

SSL (HTTPS) and file encryption/decryption can be offloaded to a processor's AES-NI extension.
This can both speed up these operations while lowering processing overhead.
This requires a processor with the `AES-NI instruction set <http://wikipedia.org/wiki/AES_instruction_set>`_.

OPcache Extension
-----------------

OPcache improves PHP performance by storing precompiled script bytecode in shared memory,
thereby removing the need for PHP to load and parse scripts on each request.
This extension is bundled with PHP 5.5.0 and later, and is available in PECL for PHP versions 5.2, 5.3, and 5.4.

Object Caching
--------------

ownCloud is written to take advantage of object caching.
Object caching can be done locally with the APCu extension,
or for distributed PHP environments using Memcached.
Memcached servers must be specified in the "memcached_servers" array in ownCloud's config file.

`Serving static files via web server <serving_static_files_configuration.rst>`_
------------------------------------

`Using cron to perform background jobs <background_jobs_configuration.rst#cron>`_
--------------------------------------

`Using MySQL instead of SQLite <database_configuration.rst#configuring-a-mysql-or-mariadb-database>`_
------------------------------

MySQL or MariaDB should be preferred because of the `performance limitations of SQLite with highly concurrent applications <http://www.sqlite.org/whentouse.html>`_, like ownCloud.

On large instances you could consider `running MySQLTuner <https://github.com/major/MySQLTuner-perl/>`_ to optimize the database.

-----------------------------------

Nginx: caching ownCloud gallery thumbnails with fastcgi_cache_purge
===================================================================

| One of the optimisations for ownCloud when using Nginx as webserver is to combine FastCGI caching with "Cache Purge", a `3rdparty Nginx module <http://wiki.nginx.org/3rdPartyModules>`_  that adds the ability to purge content from `FastCGI`, `proxy`, `SCGI` and `uWSGI` caches. This mechanism speeds up thumbnail presentation as it shifts requests to Nginx and minimizes php invocations which else would take place for every thumbnail presented every time.
| The following procedure is based on an Ubuntu 14.04 system. You may need to adopt it according your OS type and release.
| **Note I:** 
|    Unlike Apache, Nginx does not dynamically load modules. All modules needed, must be compiled into Nginx. This is one of the reasons for Nginx´s performance.
| **Note II:**
|    It is expected to have an already running Nginx installation with a working configuration set up like described in the ownCloud documentation.

Nginx module check
------------------
| As a first step, it is necessary to check if your Nginx installation has the ``nginx cache purge`` module compiled in.
| 
| ``nginx -V 2>&1 | grep ngx_cache_purge -o``
| 
| If your output contains ``ngx_cache_purge``, you can continue with the configuration, else you need to manually compile Nginx with the module needed.

Compile Nginx with the ``nginx-cache-purge`` module
---------------------------------------------------

1. | *Preparation*
   | ``cd /opt``
   | ``wget http://nginx.org/keys/nginx_signing.key``
   | ``sudo apt-key add nginx_signing.key``
   | ``sudo vi /etc/apt/sources.list.d/nginx.list``
   | 
   | Add following lines (in case, replace ``{trusty}`` by your distribution name):
   | ``deb http://nginx.org/packages/mainline/ubuntu/ trusty nginx``
   | ``deb -src http://nginx.org/packages/mainline/ubuntu/ trusty nginx``     
   |
   | Then do a
   | ``sudo apt-get update``
   |
   | **Note:**
   |   If you're not overly cautious and wish to install the latest and greatest Nginx packages and features, you may have to install Nginx from its mainline repository.
   |   From the Nginx homepage: "In general, you should deploy Nginx from its mainline branch at all times."
   |
   | If you would like to use standard Nginx from the latest mainline branch but without compiling in any additional modules, just run ``sudo apt-get install nginx``.
   |

2. | *Download the Nginx source from the ppa repository*
   | ``cd /opt``
   | ``sudo apt-get build-dep nginx``
   | ``sudo apt-get source nginx``
   |

3. | *Download module(s) to be compiled in and configure compiler arguments*
   | ``ls -la``
   | 
   | Please replace ``{release}`` with the release downloaded
   | ``cd /opt/nginx-{release}/debian``
   | 
   | If folder "modules" is not present, do:
   | ``sudo mkdir modules``
   | ``cd modules``
   | ``sudo git clone https://github.com/FRiCKLE/ngx_cache_purge.git``
   | ``sudo vi /opt/nginx-{release}/debian/rules``
   | 
   | If not present, add the following line at the top under ``#export DH_VERBOSE=1``:
   | ``MODULESDIR = $(CURDIR)/debian/modules``
   |
   | And the end of `every` ``./configure`` command add:
   | ``--add-module=$(MODULESDIR)/ngx_cache_purge``
   | 
   | Don't forget to escape preceeding lines with a backslash ``\``.
   | The parameters may now look :
   |   ``$(WITH_SPDY) \``
   |   ``--with-cc-opt="$(CFLAGS)" \``
   |   ``--with-ld-opt="$(LDFLAGS)" \``
   |   ``--with-ipv6 \``
   |   ``--add-module=$(MODULESDIR)/ngx_cache_purge``
   |

4. | *Compile and install Nginx*
   | ``cd /opt/nginx-{release}``
   | ``sudo dpkg-buildpackage -uc -b``
   | ``ls -la /opt``
   | ``sudo dpkg --install /opt/nginx_{release}~{distribution}_amd64.deb``
   |

5. | *Check if the compilation and installation of the* ``ngx_cache_purge`` *module was successful*
   | ``nginx -V 2>&1 | grep  ngx_cache_purge -o``
   | 
   | It should show now: ``ngx_cache_purge``
   | 
   | Show Nginx version including all features compiled and installed:
   | ``nginx -V 2>&1 | sed s/" --"/"\n\t--"/g``
   |

6. | *Mark Nginx to be blocked from further updates via* ``apt-get``
   | ``sudo dpkg --get-selections | grep nginx``
   | 
   | For eyery nginx component listed do a:
   | ``sudo apt-mark hold <component>``
   |

7. | *Regular checks for nginx updates*
   | Do a regular visit on the `Nginx news page <http://nginx.org>`_ and proceed in case of updates with item 2 to 5

Configure Nginx with the ``nginx-cache-purge`` module
-----------------------------------------------------
1. | *Preparation*
   | Create a directory where Nginx will save the cached thumbnails. Use any path that fits to your environment. Replace ``{path}`` with the path used, example path below:
   | ``sudo mkdir -p /usr/local/tmp/cache``
   |

2. | *Configuration*
   | ``sudo vi /etc/nginx/sites-enabled/{your-ownCloud-nginx-config-file}``
   | 
   | Note: the ``keys_zone`` / ``fastcgi_cache`` name and the ``{path}`` must be unique to each instance of ownCloud serverd with Nginx !
   | 
   | Add at the *beginning*, but *outside* the ``server{}`` block:
   | ``fastcgi_cache_path {path} levels=1:2 keys_zone=OWNCLOUD:100m inactive=60m;``
   |
   | Add *inside* the ``server{}`` block, as an example of a configuration:
   |
   | ``set $skip_cache 1;``
   |    
   | ``# POST requests and urls with a query string should always go to PHP``
   | ``if ($request_uri ~* "thumbnail.php") {``
   |      ``set $skip_cache 0;``
   |   ``}``
   |    
   |  ``fastcgi_cache_key "$scheme$request_method$host$request_uri";``
   |  ``fastcgi_cache_use_stale error timeout invalid_header http_500;``
   |  ``fastcgi_ignore_headers Cache-Control Expires Set-Cookie;``
   |    
   |  ``location ~ \.php(?:$|/) {``
   |      ``fastcgi_split_path_info ^(.+\.php)(/.+)$;``
   |    
   |      ``include fastcgi_params;``
   |      ``fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;``
   |      ``fastcgi_param PATH_INFO $fastcgi_path_info;``
   |      ``fastcgi_param HTTPS on;``
   |      ``fastcgi_pass php-handler;``
   |    
   |      ``fastcgi_cache_bypass $skip_cache;``
   |      ``fastcgi_no_cache $skip_cache;``
   |      ``fastcgi_cache OWNCLOUD;``
   |      ``fastcgi_cache_valid  60m;``
   |    ``}``
   |
   | Note regarding the ``fastcgi_pass`` parameter:
   | Use whatever fits your configuration. In the example above, a ``upstream`` was defined in an Nginx global configuration file.
   | This then can look like:
   |    
   |  ``upstream php-handler {``
   |      ``server 127.0.0.1:9000;``
   |      ``# or``
   |      ``#server unix:/var/run/php5-fpm.sock;``
   |    ``}``
   |
   
3. | *Test the configuration*
   |  ``sudo service nginx restart``
   
   * | Open your browser and clear your cache.
   
   * | Logon to your ownCloud instance, open the gallery app, move thru your folders
     | and watch while the thumbs are generated for the first time.
   * | You may also watch with eg. ``htop`` your system load while the thumbnails are processed.
   * | Goto another app or logout and relogon.
   * | Open the gallery app again and browse to the folders you accessed before.
     | Your thumbnails should appear more or less immediately.
   * | ``htop`` will not show up additional load while processing, compared to the high load before. 
