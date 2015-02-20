Cartodb Ubuntu 14.04
=============

CartoDB VM on Ubuntu Server 14.04 LTS
based upon https://gist.githubusercontent.com/tomay/7779546/raw/5479e0be07e80b1cc5b4341c180d675afff5b427/cartodb+install+notes+12.04.md

### Cartodb install on Digital Ocean ubuntu 12.04 64-bit
based on https://github.com/CartoDB/cartodb with additions as necessary

Install git

    sudo apt-get install git-core

Clone project

    git clone --recursive https://github.com/CartoDB/cartodb.git

Some dependencies
py
    apt-get update
    sudo apt-get install python-software-properties
    **sudo apt-get install software-properties-common**

More dependencies

    sudo apt-get install unp
    sudo apt-get install zip

GDAL

    sudo apt-get install gdal-bin libgdal1-dev

GEOS

    # note: seems all ok from above; 0 upgraded, 0 newly installed, 0 to remove and 20 not upgraded.
    sudo apt-get install libgeos-c1 libgeos-dev

JSON-C

    sudo apt-get install libjson0 python-simplejson libjson0-dev

PROJ-4

    sudo apt-get install proj-bin proj-data libproj-dev

POSTGRESQL

    # Note: 12.04 so not using carto ppa here: sudo add-apt-repository ppa:cartodb/postgresql 
    ***sudo apt-get install postgresql-9.1 postgresql-client-9.1 postgresql-contrib-9.1 postgresql-server-dev-9.1
    sudo apt-get install postgresql-9.3 postgresql-client-9.3 postgresql-contrib-9.3 postgresql-server-dev-9.3

PLP-PYTHON

    ***sudo apt-get install postgresql-plpython-9.1
    sudo apt-get install postgresql-plpython-9.3

POST-GIS    
    PostGIS 2.1 is included in 14.04
    # sudo apt-get install postgis postgresql-9.3-postgis-2.1

Configure spatial-db tempate

    # create and run setup script
    touch /var/lib/postgresql/config_db.sh

    nano /var/lib/postgresql/config_db.sh

    #!/usr/bin/env bash
    POSTGIS_SQL_PATH=/usr/share/postgresql/9.3/contrib/postgis-2.1
    createdb -E UTF8 template_postgis
    createlang -d template_postgis plpgsql
    psql -d postgres -c \ "UPDATE pg_database SET datistemplate='true' WHERE datname='template_postgis'"
    psql -d template_postgis -f $POSTGIS_SQL_PATH/postgis.sql
    psql -d template_postgis -f $POSTGIS_SQL_PATH/spatial_ref_sys.sql
    psql -d template_postgis -f $POSTGIS_SQL_PATH/legacy.sql
    psql -d template_postgis -f $POSTGIS_SQL_PATH/rtpostgis.sql
    psql -d template_postgis -f $POSTGIS_SQL_PATH/topology.sql
    psql -d template_postgis -c "GRANT ALL ON geometry_columns TO PUBLIC;"
    psql -d template_postgis -c "GRANT ALL ON spatial_ref_sys TO PUBLIC;"

    sudo su - postgres 
    bash config_db.sh

    # confirm template exists with
    psql --command="\l"
    exit

Install Ruby 1.9.2 (using RVM)

    sudo su
    \curl -L https://get.rvm.io | bash
    source /etc/profile.d/rvm.sh
    rvm install 1.9.2

Node

    sudo apt-get install nodejs npm

Redis

    sudo apt-get install redis-server


ImageMagik 

    # something to do with windshaft and comparing images
    sudo apt-get install imagemagick

Install Python dependencies

    sudo apt-get install python-setuptools
    sudo apt-get install python-dev
    sudo? easy_install pip
    sudo? pip install -r cartodb/python_requirements.txt

    # when python-gdal bindings fail to install using above
    sudo apt-add-repository ppa:ubuntugis/ubuntugis-unstable
    sudo apt-get update
    sudo apt-get install python-gdal

    # leaving this ppa in for now??
    sudo apt-add-repository --remove ppa:ubuntugis/ubuntugis-unstable
    apt-get update

Varnish
    
    # RealGeeks version recommended in Cartodb docs
    sudo pip install -e git+https://github.com/RealGeeks/python-varnish.git@0971d6024fbb2614350853a5e0f8736ba3fb1f0d#egg=python-varnish

Mapnik
    
    # following not working, note these seem to be v 2.1.1 
    ## sudo apt-get install libmapnik-dev python-mapnik mapnik-utils

    # so remove the cartodb ppa
    sudo apt-add-repository --remove ppa:cartodb/mapnik
    
    # install mapnik ppa
    sudo add-apt-repository ppa:mapnik/nightly-2.1 
    apt-get update

    # install boost (q: with suggests? --install-suggests, a: so far seems ok without)
    sudo apt-get install libboost1.55-tools-dev

    # then install mapnik
    sudo apt-get install libmapnik-dev python-mapnik mapnik-utils

    # can test with
    python
    import mapnik

### CartoDB SQL API

    # nodejs-legacy fixes 'WARN This failure might be due to the use of legacy binary "node"' error
    sudo apt-get install nodejs-legacy
    git clone git://github.com/CartoDB/CartoDB-SQL-API.git
    cd CartoDB-SQL-API
    git checkout master 
    npm install

    mv config/environments/test.js.example config/environments/test.js

    # run test
    export PGUSER=postgres # note see config below
    make check # 143 tests pass

### Windshaft-cartodb

    # note about g++ memory error (only 512 MB RAM for me: http://stackoverflow.com/questions/14800609/phusion-passenger-nginx-module-installation-error-on-ubuntu-server-12-04-2-64)
    git clone git://github.com/CartoDB/Windshaft-cartodb.git
    cd Windshaft-cartodb
    git checkout master
    npm install

    mv config/environments/development.js.example config/environments/development.js
    # set mapnik version "2.1.1"
    # create millstone directory writable
    mkdir -p /tmp/cdb-tiler-dev/millstone-dev

    # testing
    # 1. setup test db and user


    make check


### Additional pg config

    ## pg
    #In /etc/postgresql/9.3/main/pg_hba.conf, set auth method to "trust":
    local   all             postgres                                trust
    host    all             all             127.0.0.1/32            trust

    # restart psql
    sudo service postgresql restart
    
    # set pw (also add to database.yml -- optional/redundant given above??
    sudo -u postgres psql
    \password
    (Set password) 
    \q

### CartoDB Schema Triggers

    git clone https://github.com/CartoDB/pg_schema_triggers.git && \
      cd pg_schema_triggers && \
      sudo make all install && \
      sudo sed -i \
      "/#shared_preload/a shared_preload_libraries = 'schema_triggers.so'" \
      /etc/postgresql/9.3/main/postgresql.conf

## Cartodb Postgres Extension

    # version is 0.5.2 but it keeps changing ...
    git clone --branch 0.5.2 https://github.com/CartoDB/cartodb-postgresql && \
      cd cartodb-postgresql && \
      PGUSER=postgres sudo make install
    
### Setup (see https://github.com/CartoDB/cartodb for details)

    export SUBDOMAIN=carto # unset SUBDOMAIN to undo
    cd cartodb

    # bundle --> carto seems to require ruby 1.9.3 so installed this first
    rvm install ruby-1.9.3
    rvm use 1.9.3@cartodb --create && bundle install # from cartodb/
    # sudo bundle install # not needed? anyway bundle not found for sudo

    mv config/app_config.yml.sample config/app_config.yml
    nano config/app_config.yml
    # only change so far: developers_host:    'http://carto.localhost.lan:3000'

    mv config/database.yml.sample config/database.yml
    nano config/database.yml

    echo "127.0.0.1 ${SUBDOMAIN}.localhost.lan" | sudo tee -a /etc/hosts
    # add similar entry to local /etc/hosts
    xxx.xxx.xxx.xxx carto.localhost.lan
    xxx.xxx.xxx.xxx admin.localhost.lan

### Create user script and alter limits
    # start redis-server with redis-server
    # create user script (edit to avoid error): sh script/create_dev_user ${SUBDOMAIN}
    bundle exec rake cartodb:db:set_user_quota["myuser",10240] 
    bundle exec rake cartodb:db:set_unlimited_table_quota["myuser"]
    bundle exec rake cartodb:db:set_user_private_tables_enabled["myuser",'true']
    bundle exec rake cartodb:db:set_user_account_type["myuser",'[DEDICATED]'] 

### Start all servers
    
    # foreman not working for me - but starting services individually is ...
    bundle exec foreman start -p 3000
    
### Useful miscellany
    
    apt-cache policy mapnik-utils

    # drop db
    sudo su - postgres
    dropdb carto_db_development

    # drop roles
    sudo -u postgres psql
    DROP ROLE publicuser
    DROP ROLE tileuser

    # node tests
    export PGUSER = "postgres"
    make check # 143 tests complete

    # restart pg
    sudo service postgresql restart
    
    # references
    ref to 12.04 setup: https://gist.github.com/arjendk/63a0603d9dd278503edb
