# Vagrant Heroku cedar-16 box for Python and Django

A Vagrant box for python/Django development, mimicking a Heroku cedar-16 dyno. Based on the older [cedar-14 version](https://github.com/philgyford/vagrant-heroku-cedar-14-python).

* Ubuntu 16.04 (bento/ubuntu-16.04)
* PostgreSQL 9.6
* Python 2.7, 3.5 or 3.6
* pip, virtualenv, virtualenvwrapper
* Requirements for the python image processing module Pillow
* foreman (optional, and sometimes problematic; see below)
* GeoDjango requirements (optionally)

If a `requirements.txt` file is found, packages in it will be installed into the virtualenv. (The filename can be configured, see below.)

When you ssh into the VM the virtualenv will automatically be activated.

If a `Procfile` is found, and the `start_foreman` setting is `'true'`, foreman
will be started. (The Procfile filename can be configured, see below.)

The project directory (containing the `Vagrantfile`) will be availble in the VM at `/vagrant/`.

If there's a `manage.py` file in the root of the project, it will be used to run Django's `migrate` (optional) and `collectstatic` commands.


## Running it

1. Install [Vagrant](https://www.vagrantup.com/) and [VirtualBox](https://www.virtualbox.org/).

2. In your project, make a `config/` directory, if it doesn't already have one.

3. Make a copy of `config/vagrant.template.yml` and put it at `config/vagrant.yml` in *your* project.

4. If you have a Procfile, and want foreman to run, you *must* change the Django `settings_module` in `config/vagrant.yml` to whatever you want the `DJANGO_SETTINGS_MODULE` environment variable in the virtual machine to be. Feel free to change any of the other config options if appropriate.

    You can also change the name of the Procfile foreman should use in `config/vagrant.yml`, in case you want to use a different one in Vagrant versus production (see below).

5. Either copy, move or symlink `Vagrantfile` and the `config/vagrant/` directory into your Django project. So it will be something like:

        config/
            vagrant/            # a copy or symlink
                bash_setup.sh
                ...
            vagrant.yml
        myproject/
            ...
        manage.py
        Procfile
        requirements.txt
        runtime.txt             # optional, see below
        Vagrantfile             # a copy or symlink

    This will vary slightly depending on your Django project's layout.

6. Run `vagrant up` from the same directory that the copy/symlink of `Vagrantfile` is in.

7. Go to [http://localhost:5000](http://localhost:5000) or [http://0.0.0.0:5000](http://0.0.0.0:5000) in your browser.

If you change or update any of the Vagrant stuff, then do `vagrant provision` to have it run through and update the box with changes.

### Subsequently

If you `vagrant halt` the box, you'll need to do `vagrant up --provision` to get everything running again. Just doing `vagrant up` won't currently start foreman etc.

## Configuring

Edit the `config/vagrant.yml` file to change the following settings:

### `db`

`name`, `user` and `password` will be used to for the Postgres database name, and to create a database user. These should be the same as in your Django settings file.

### `django`

`settings_module`: Specify the dotted path to the Django settings module. e.g. `myproject.settings.development` will use the settings file at `myproject/settings/development.py`.

`run_migrations`: This should be `'true'` if you want the Django migrations to
be run. If you're going to import an existing database dump (see below) you
should set this to `'false'` (or anything other than `'true'`).

### `foreman`

`procfile`: As described above, if you want to use a specific Procfile for use in the Vagrant VM, set its name here. e.g. `Procfile.dev`.

`start_foreman`: This should be `'true'` to use foreman to run the webserver.
Set it to `'false'` (or anything other than `'true'`) if you want to use the
Django `runserver` command manually (see below).

### `virtualbox`

`name`: This will be the name of the box as seen in the Virtualbox UI.

`memory`: How much RAM should the virtual machine have in MB? e.g. `'1024'`.

### `virtualenv`

`envname`: This will be used to create a python virtualenv in which all the project's requirements, and python version, will be installed. e.g. `myvirtualenv` will result in a virtualenv being created at `/home/vagrant/.virtualenvs/myvirtualenv/`.

`requirements_file`: Specify the name of the pip requirements file to use when setting up the virtualenv. e.g. `requirements_dev.txt`.

### `use_geodjango`

Set this to `'true'` if you want to install the necessary packages for GeoDjango. See below.


## Foreman or the Django development server?

By default foreman sends output to stdout and stderr. This prevents Vagrant from exiting nicely, even though we run foreman as `foreman .. &`. To ensure a smooth exit from foreman, and to be able to see its output in future, you should send the output of processes in your Procfile to a file. eg:

    web: gunicorn --reload --log-level debug myproject.wsgi > /home/vagrant/gunicorn.log 2>&1

Then you can just `tail -f /home/vagrant/gunicorn.log` to see its output. For this reason you might want to use a different Procfile for use in Vagrant than you do with your live server (use the setting in `config/vagrant.yml` to specify the filename).

Also note: We use the `--reload` option with gunicorn so that it reloads when code changes. Otherwise you'll never see changes when you make them!

**However:** This still [seems to get stuck](http://stackoverflow.com/questions/38208840/restart-gunicorn-run-with-foreman-on-error) sometimes, with gunicorn's log showing "Worker failed to boot". To avoid using foreman at all then set the `start_foreman` setting to `'false'`, then you can run the Django dev server manually:

    $ vagrant ssh
    vagrant$ /vagrant/manage.py runserver 0.0.0.0:5000


## Python versions

By default the virtualenv will use python 2.7.

To specify python 3.6 [Heroku requires you](https://devcenter.heroku.com/articles/python-runtimes) to place a `runtime.txt` file in your repository's root containing one line:

    python-3.6.2

When setting up the Vagrant box, if this file is present, and contains `python-3.6*`, then python 3.6 will be installed and used for the virtualenv. It will do similar for `python-3.5*`, although Heroku don't support this on Cedar 16.


## GeoDjango

If you want to use GeoDjango, set the ``use_geodjango`` variable in your `config/vagrant.yml` file to the string ``true``:

    use_geodjango: 'true'

This will install the requirements for using GeoDjango: GEOS, PROJ.4, GDAL, PostGIS.

### Note

The install script should do this, but I had an occasion when I was getting this error while running Django migrations:

	django.db.utils.ProgrammingError: permission denied to create extension "postgis"

So I had to do this (replace `DB_NAME` with your database name):
	
	$ vagrant ssh
	vagrant$ sudo -u postgres psql DB_NAME
	=# CREATE EXTENSION postgis;
	=# \q


## Database

To change the version of Postgres (and PostGIS, if you're using GeoDjango), edit the variable(s) in `config/vagrant/postgresql_setup.sh`.

The process above will set up a postgres database and user, and run Django migrations, but not populate the database. Database name, username and password are set in `config/vagrant.yml`.

If you have a `pg_dump` dump file, put it in the same directory as your Django project and then:

    $ vagrant ssh
    vagrant$ cd /vagrant
    vagrant$ pg_restore --verbose --clean --no-acl --no-owner -h localhost -U your_pg_username -d your_db_name your-dump-name.dump

You'll be prompted for the postgres user's password, and then it should import.

NOTE: You might get errors when importing due to the database tables existing.
If you want to undo all those Django migrations, and delete all the tables, this
seems to work (use at your own risk):

	$ vagrant ssh
	vagrant$ psql -U your_pg_username -h localhost -d your_db_name
	=> select 'drop table "' || tablename || '" cascade;' from pg_tables where schemaname = 'public';
	=> \q

Then you can do the `pg_restore`.


## Environment variables

The simplest way to set up environment variables in the virtual environment is to put a `.env` file in the root of of your project (i.e., the same level as `requirements.txt`, `config/`, etc.).

If present, this file will be symlinked in the VM to the virtual environment's `postactivate` hook, so it will be run whenever the environment is activated. An example file:

    #!/bin/bash

    export MY_ENVIRONMENT_VARIABLE=hello-there
    export MY_OTHER_VAR="Hello there"


## Virtualenvwrapper

If you need more control over the virtual environment than setting environment variables (above), then you can do the following. NOTE: doing this *and* using a `.env` file is probably a bad idea.

To use virtualenvwrapper's [user-defined hooks](http://virtualenvwrapper.readthedocs.org/en/latest/scripts.html#scripts) then create a directory at `config/virtualenvwrapper/vagrant/` containing them. For example, continuing our example structure above:

        config/
            vagrant/        # a copy or symlink
            vagrant.yml
            virtualenvwrapper/
                vagrant/
                    postactivate
                    postdeactivate
                    preactivate
        ...

If this directory is present, the `VIRTUALENVWRAPPER_HOOK_DIR` environment variable will be set, and these files will be used instead of the defaults.

So, to set environment variables for your virtualenv you might add something like this to your `postactivate` file:

    #!/bin/bash
    # This hook is run after this virtualenv is activated.

    export MY_ENVIRONMENT_VARIABLE=hello-there
    export MY_OTHER_VAR="Hello there"

These files are only created on initial set-up, so to change their contents subsequently, do it manually. They can be found in the vagrant box at `/home/vagrant/.virtualenvs/your-venv-name/bin/`.


## Potential problems

If you try to access your Django site in a browser but get a "Peer authentication failed for user" error, then ensure you've set the `HOST` value in your Django settings file to `localhost` or (*for development only*) `"*"`. An empty string will not work.


## Useful / Inspired by:

* https://github.com/kiere/vagrant-heroku-cedar-14/
* https://github.com/ejholmes/vagrant-heroku/
* https://github.com/torchbox/vagrant-django-base/
* https://github.com/torchbox/vagrant-django-template
* https://github.com/jackdb/pg-app-dev-vm/
* https://github.com/maigfrga/django-vagrant-chef/
* https://devcenter.heroku.com/articles/cedar-ubuntu-packages


## Credits

* [Phil Gyford](https://github.com/philgyford) - Initial creation
* [Steven Day](https://github.com/stevenday) - [cedar-14](https://github.com/philgyford/vagrant-heroku-cedar-14-python) v1.1 updates and fixes

