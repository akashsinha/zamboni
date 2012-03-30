.. _installation:

==================
Installing Zamboni
==================

We're going to use all the hottest tools to set up a nice environment.  Skip
steps at your own peril. Here we go!

.. note::

    For less manual work, you can build Zamboni in a
    :doc:`virtual machine using vagrant <install-with-vagrant>`.


Requirements
------------
To get started, you'll need:
 * Python 2.6 (greater than 2.6.1)
 * MySQL
 * libxml2 (for building lxml, used in tests)

:ref:`OS X <osx-packages>` and :ref:`Ubuntu <ubuntu-packages>` instructions
follow.

There are a lot of advanced dependencies we're going to skip for a fast start.
They have their own :ref:`section <advanced-install>`.

If you're on a Linux distro that splits all its packages into ``-dev`` and
normal stuff, make sure you're getting all those ``-dev`` packages.


.. _ubuntu-packages:

On Ubuntu
~~~~~~~~~
The following command will install the required development files on Ubuntu or,
if you're running a recent version, you can `install them automatically
<apt:python-dev,python-virtualenv,libxml2-dev,libxslt1-dev,libmysqlclient-dev,libmemcached-dev>`_::

    sudo aptitude install python-dev python-virtualenv libxml2-dev libxslt1-dev libmysqlclient-dev libmemcached-dev libssl-dev swig


.. _osx-packages:

On OS X
~~~~~~~
The best solution for installing UNIX tools on OSX is Homebrew_.

The following packages will get you set for zamboni::

    brew install python libxml2 mysql libmemcached openssl swig jpeg

.. _Homebrew: http://github.com/mxcl/homebrew#readme


MySQL
~~~~~

You'll probably need to :ref:`configure MySQL after install <configure-mysql>`
(especially on Mac OS X) according to advanced installation.


Use the Source
--------------

Grab zamboni from github with::

    git clone --recursive git://github.com/mozilla/zamboni.git
    cd zamboni
    svn co http://svn.mozilla.org/addons/trunk/site/app/locale locale

``zamboni.git`` is all the source code.  ``zamboni-lib.git`` is all of our
pure-Python dependencies.  :ref:`updating` is detailed later on.

``locale`` contains all of the localizations of the site.  Unless you are
specifically working with locales, you probably don't need to touch this again
after you check it out.


virtualenv
----------

`virtualenv <http://pypi.python.org/pypi/virtualenv>`_ is a tool to create
isolated Python environments.  We don't want to install packages system-wide
because that can create quite a mess. ::

    sudo easy_install virtualenv

virtualenv is the only Python package I install system-wide.  Everything else
goes in a virtual environment.


virtualenvwrapper
~~~~~~~~~~~~~~~~~

`virtualenvwrapper <http://www.doughellmann.com/docs/virtualenvwrapper/>`_
is a set of shell functions that make virtualenv easy to work with.

Install it like this::

    wget http://bitbucket.org/dhellmann/virtualenvwrapper/raw/f31869779141/virtualenvwrapper_bashrc -O ~/.virtualenvwrapper
    mkdir ~/.virtualenvs

Then put these lines in your ``~/.bashrc``::

    export WORKON_HOME=$HOME/.virtualenvs
    source $HOME/.virtualenvwrapper

``exec bash`` and you're set.

.. note:: If you didn't have a ``.bashrc`` already, you should make a
          ``~/.profile`` too::

            echo 'source $HOME/.bashrc' >> ~/.profile


virtualenvwrapper Hooks (optional)
**********************************

virtualenvwrapper lets you run hooks when creating, activating, and deleting
virtual environments.  These hooks can change settings, the shell environment,
or anything else you want to do from a shell script.  For complete hook
documentation, see
http://www.doughellmann.com/docs/virtualenvwrapper/hooks.html.

You can find some lovely hooks to get started at http://gist.github.com/536998.
The hook files should go in ``$WORKON_HOME`` (``$HOME/.virtualenvs`` from
above), and ``premkvirtualenv`` should be made executable.


Getting Packages
----------------

Now we're ready to go, so create an environment for zamboni::

    mkvirtualenv --no-site-packages zamboni

That creates a clean environment named zamboni.  You can get out of the
environment by restarting your shell or calling ``deactivate``.

To get back into the zamboni environment later, type::

    workon zamboni  # requires virtualenvwrapper

.. note:: If you want to use a different Python binary, pass the path to
          mkvirtualenv with ``--python``::

            mkvirtualenv --python=/usr/local/bin/python2.6 --no-site-packages zamboni


Finish the install
~~~~~~~~~~~~~~~~~~

From inside your activated virtualenv, run::

    pip install -r requirements/compiled.txt

pip installs a few packages into our new virtualenv that we can't distribute in
``zamboni-lib``.  These require a C compiler.


.. _example-settings:

Settings
--------

.. note::

    Also see the Multiple Sites section below for using settings files to run
    the Add-ons and Marketplace sites side by side.

Most of zamboni is already configured in ``settings.py``, but there's some
things you need to configure locally.  All your local settings go into
``settings_local.py``.  The settings template for
developers, included below, is at :src:`docs/settings/settings_local.dev.py`.

.. literalinclude:: /settings/settings_local.dev.py

I'm overriding the database parameters from ``settings.py`` and then extending
``INSTALLED_APPS`` and ``MIDDLEWARE_CLASSES`` to include the `Django Debug
Toolbar <http://github.com/robhudson/django-debug-toolbar>`_.  It's awesome,
you want it.

Any file that looks like ``settings_local*`` is for local use only; it will be
ignored by git.

Database
--------

Instead of running ``manage.py syncdb`` your best bet is to grab a snapshot of
our production DB which has been redacted and pruned for development use.
Development snapshots are hosted over at
https://landfill.addons.allizom.org/db/

Here are some shell commands to pull down the latest snapshot and set it up::

    export DB_NAME=zamboni
    export DB_USER=zamboni
    mysqladmin -uroot create $DB_NAME
    mysql -uroot -B -e'GRANT ALL PRIVILEGES ON $DB_NAME.* TO $DB_USER@localhost'
    wget --no-check-certificate -P /tmp https://landfill.addons.allizom.org/db/landfill-`date +%Y-%m-%d`.sql.gz
    zcat /tmp/landfill-`date +%Y-%m-%d`.sql.gz | mysql -u$DB_USER $DB_NAME
    # Optionally, you can remove the landfill site notice:
    mysql -uroot -e"delete from config where \`key\`='site_notice'" $DB_NAME

Database Migrations
-------------------

Each incremental change we add to the database is done with a versioned SQL
(and sometimes Python) file. To keep your local DB fresh and up to date, run
migrations like this::

    ./vendor/src/schematic/schematic migrations

More info on schematic: https://github.com/jbalogh/schematic


Multiple sites
--------------

We now run multiple sites off the zamboni code base. The current sites are:

- *default* the Add-ons site at https://addons.mozilla.org/

- *mkt* the Mozilla Marketplace at https://marketplace.mozilla.org/

There are modules in zamboni for each of these base settings to make minor
modifications to settings, url, templates and so on. Start by copying the
template from ``settings/settings_local.dev.py`` into a custom file.

To run the Add-ons site, make a ``settings_local_amo.py`` file with this import
header::

    from default.settings import *

Or to run the Marketplace site, make a ``settings_local_mkt.py`` file with
these imports::

    from mkt.settings import *


Run the Server
--------------

If you've gotten the system requirements, downloaded ``zamboni`` and
``zamboni-lib``, set up your virtualenv with the compiled packages, and
configured your settings and database, you're good to go.

To choose which site you want to run, use the `settings` command line
argument to pass in a local settings file you created above.

Run The Add-ons Server
~~~~~~~~~~~~~~~~~~~~~~

::

    ./manage.py runserver --settings=settings_local_amo 0.0.0.0:8000

Run The Marketplace Server
~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    ./manage.py runserver --settings=settings_local_mkt 0.0.0.0:8000


Create an Admin User
--------------------

To log into your dev site, you can click the login / register link and login
with Browser ID just like on the live site. However, if you want to grant
yourself admin privileges there are some additional steps. After registering,
find your user record::

    mysql> select * from auth_user order by date_joined desc limit 1\G

Then make yourself a superuser like this::

    mysql> update auth_user set is_superuser=1, is_staff=1 where id=<id from above>;

Next, you'll need to set a password. Do that by clicking "I forgot my password"
on the login screen then go back to the shell you started your dev server in.
You'll see the email message with the password reset link in stdout.


Testing
-------

The :ref:`testing` page has more info, but here's the quick way to run
zamboni's marketplace tests::

    ./manage.py test --settings=settings_local_mkt

Or to run AMO's tests::

    ./manage.py test --settings=settings_local_amo

.. _updating:

Updating
--------

This updates zamboni::

    git checkout master && git pull && git submodule update --init

This updates zamboni-lib in the ``vendor/`` directory::

    pushd vendor && git pull && git submodule update --init && popd

We use `schematic <http://github.com/jbalogh/schematic/>`_ to run migrations::

    ./vendor/src/schematic/schematic migrations

The :ref:`contributing` page has more on managing branches.

If you want to pull in the latest locales::

    pushd locale && svn up && popd


Contact
-------

Come talk to us on irc://irc.mozilla.org/amo if you have questions, issues, or
compliments.


Submitting a Patch
------------------

See the :ref:`contributing` page.


.. _advanced-install:

Advanced Installation
---------------------

In production we use things like memcached, rabbitmq + celery, sphinx, redis,
and LESS.  Learn more about installing these on the
:doc:`./advanced-installation` page.

.. note::

    Although we make an effort to keep advanced items as optional installs
    you might need to install some components in order to run tests or start
    up the development server.