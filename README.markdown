rails-nginx-passenger-centos
============================

My notes on setting up a simple production server with centos, nginx, passenger and mysql for rails.

Aliases
-------

    echo "alias ll='ls -l'" >> ~/.bash_aliases
    
edit .bashrc and uncomment the loading of .bash_aliases

If you have trouble with PATH that changes when doing sudo, see http://stackoverflow.com/questions/257616/sudo-changes-path-why then add the following line to the same file

    echo "alias sudo='sudo env PATH=$PATH'" >> ~/.bash_aliases
    

Update and upgrade the system
-------------------------------

    sudo yum update

Configure timezone
-------------------

	yum install tzdata system-config-date redhat-config-date
	yum install ntp
    sudo ntpdate ntp.ubuntu.com # Update time
    
Verify that you have to correct date and time with

    date

Configure hostname
-------------------

    sudo hostname your-hostname

Add 127.0.0.1 your-hostname

    sudo vim /etc/hosts
    
Write your-hostname in 
    
    sudo vim /etc/hostname
    
Verify that hostname is set
    
    hostname

Install mysql
---------------

This should be installed before Ruby Enterprise Edition becouse that will install the mysql gem.

    sudo yum install mysql mysql-devel
    
    
Gemrc
-------

Add the following lines to ~/.gemrc, this will speed up gem installation and prevent rdoc and ri from being generated, this is not nessesary in the production environment.

    ---
    :sources:
    - http://gems.rubyforge.org
    - http://gems.github.com
    gem: --no-ri --no-rdoc


Ruby Enterprise Edition
------------------------

Check for newer version at http://www.rubyenterpriseedition.com/download.html

Install package required by ruby enterprise, C compiler, Zlib development headers, OpenSSL development headers, GNU Readline development headers

	yum groupinstall "Development Tools"
	yum install zlib-devel wget openssl-devel pcre pcre-devel readline-devel


Download and install Ruby Enterprise Edition

    wget http://rubyforge.org/frs/download.php/66162/ruby-enterprise-X.X.X-ZZZZ.ZZ.tar.gz
    tar xvfz ruby-enterprise-X.X.X-ZZZZ.ZZ.tar.gz 
    rm ruby-enterprise-X.X.X-ZZZZ.ZZ.tar.gz 
    cd ruby-enterprise-X.X.X-ZZZZ.ZZ/
    sudo ./installer
    
    
Change target folder to /opt/ruby for easier upgrade later on

Add Ruby Enterprise bin to PATH

    echo "export PATH=/opt/ruby/bin:$PATH" >> ~/.profile && . ~/.profile
    
Verify the ruby installation

    ruby -v
    rruby 1.8.7 (2010-04-19 patchlevel 253) [i686-linux], MBARI 0x8770, Ruby Enterprise Edition 2010.02


Installing git
----------------

    yum install gettext-devel expat-devel curl-devel zlib-devel openssl-devel
	cd /usr/local/src
	wget http://kernel.org/pub/software/scm/git/git-1.7.1.tar.gz
	tar xzvf git-1.7.1.tar.gz
	cd git-1.7.1
	make prefix=/usr/local all
	make prefix=/usr/local install
	
	ref: http://stackoverflow.com/questions/2318999/installing-git-on-centos

Nginx
-------

    sudo /opt/ruby/bin/passenger-install-nginx-module

Select option 1. Yes: download, compile and install Nginx for me. (recommended)

When finished, verify nginx source code is located under /tmp

    $ ll /tmp/
    drwxr-xr-x 8 deploy deploy    4096 2009-04-18 17:48 nginx-0.6.36
    -rw-r--r-- 1 root   root    528425 2009-04-02 08:49 nginx-0.6.36.tar.gz
    drwxrwxrwx 7   1169   1169    4096 2009-04-18 17:56 pcre-7.8
    -rw-r--r-- 1 root   root   1168513 2009-04-18 17:51 pcre-7.8.tar.gz
    
Run the passenger-install-nginx-module once more if you want to add --with-http_ssl_module 

    $ sudo /opt/ruby/bin/passenger-install-nginx-module
    
Select option 2. No: I want to customize my Nginx installation. (for advanced users)

When installation script ask, "Where is your Nginx source code located?" Enter:

    /tmp/nginx-0.6.36

On, extra arguments to pass to configure script add

     --with-http_ssl_module
     
     
Nginx init script
-------------------

More information on http://wiki.nginx.org/Nginx-init-ubuntu

    cd
    git clone git://github.com/jnstq/rails-nginx-passenger-ubuntu.git
    sudo mv rails-nginx-passenger-ubuntu/nginx/nginx /etc/init.d/nginx
    sudo chown root:root /etc/init.d/nginx
    
Verify that you can start and stop nginx with init script

    sudo /etc/init.d/nginx start
    
      * Starting Nginx Server...
      ...done.
    
    sudo /etc/init.d/nginx status
    
      nginx found running with processes:  11511 11510
    
    sudo /etc/init.d/nginx stop
    
      * Stopping Nginx Server...
      ...done.
    
    sudo /sbin/chkconfig nginx on
    
If you want, reboot and see so the webserver is starting as it should.

Installning ImageMagick and RMagick
-----------------------------------

If you want to install the latest version of ImageMagick. I used MiniMagick that shell-out to the mogrify command, worked really well for me.

    # If you already installed imagemagick from yum
	sudo yum remove imagemagick

    yum install tcl-devel libpng-devel libjpeg-devel ghostscript-devel bzip2-devel freetype-devel libtiff-devel
	yum install libjpeg-devel libpng-devel glib2-devel fontconfig-devel zlib-devel libwmf-devel freetype-devel

Use wget to grab the source from ImageMagick.org.

    wget ftp://ftp.imagemagick.org/pub/ImageMagick/ImageMagick.tar.gz

Once the source is downloaded, uncompress it:
	
    tar xvfz ImageMagick.tar.gz

Now configure and make:

    cd ImageMagick-6.6.3-10
    ./configure --prefix=/usr --with-bzlib=yes --with-fontconfig=yes --with-freetype=yes --with-gslib=yes --with-gvc=yes --with-jpeg=yes --with-jp2=yes --with-png=yes --with-tiff=yes
    make
    sudo make install

To avoid an error such as:

convert: error while loading shared libraries: libMagickCore.so.2: cannot open shared object file: No such file or directory

    sudo /sbin/ldconfig

Install RMagick
 
    sudo /opt/ruby/bin/ruby /opt/ruby/bin/gem install rmagick

Test a rails applicaton with nginx
----------------------------------

    rails -d mysql testapp
    cd testapp
    
Enter your mysql password
    
    vim database.yml
    rake db:create:all
    ruby script/generate scaffold post title:string body:text
    rake db:migrate RAILS_ENV=production
    
Check so the rails app start as normal
    
    ruby script/server

    sudo vim /opt/nginx/conf/nginx.conf
    
Add a new virutal host

    server {
        listen 80;
        # server_name www.mycook.com;
        root /home/deploy/testapp/public;
        passenger_enabled on;
    }
    
Restart nginx

    sudo /etc/init.d/nginx restart
    
Check you ipaddress and see if you can acess the rails application
        
