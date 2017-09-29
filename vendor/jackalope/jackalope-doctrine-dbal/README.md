# Jackalope [![Build Status](https://secure.travis-ci.org/jackalope/jackalope-doctrine-dbal.png?branch=master)](http://travis-ci.org/jackalope/jackalope-doctrine-dbal)

Implementation of the PHP Content Repository API ([PHPCR](http://phpcr.github.io))
using a relational database to persist data.

Jackalope uses Doctrine DBAL to abstract the database layer. It is currently
tested to work with MySQL, PostgreSQL and SQLite.

For the moment, it is less feature complete, performant and robust than
[Jackalope-Jackrabbit](http://github.com/jackalope/jackalope-jackrabbit) but it
can run on any server with PHP and an SQL database.

Discuss on jackalope-dev@googlegroups.com or visit #jackalope on irc.freenode.net


## License

This code is dual licensed under the MIT license and the Apache License Version
2.0. Please see the file LICENSE in this folder.


# Preconditions

* php >= 5.3
* One of the following databases, including the PDO extension for it:
    * MySQL >= 5.1.5 (we need the ExtractValue function)
    * PostgreSQL
    * SQLite


# Installation

The recommended way to install jackalope is through [composer](http://getcomposer.org/).
You can of course do without, but then you will need to resolve the dependencies
manually.

```sh
$ mkdir my-project
$ cd my-project
$ curl -s http://getcomposer.org/installer | php
$ ./composer.phar init
$ ./composer.phar require jackalope/jackalope-doctrine-dbal
$ ./composer.phar install
```

## Create a repository

Set up a new database supported by Doctrine DBAL. You can use your favorite GUI frontend or just do something like this:

### MySQL

Note that you need at least version 5.1.5 of MySQL, otherwise you will get ``SQLSTATE[42000]: Syntax error or access violation: 1305 FUNCTION cmf-app.EXTRACTVALUE does not exist``

```sh
$ mysqladmin -u root -p  create jackalope
$ echo "grant all privileges on jackalope.* to 'jackalope'@'localhost' identified by '1234test'; flush privileges;" | mysql -u root -p
```
### PostgreSQL

```sh
$ psql -c "CREATE ROLE jackalope WITH ENCRYPTED PASSWORD '1234test' NOINHERIT LOGIN;" -U postgres
$ psql -c "CREATE DATABASE jackalope WITH OWNER = jackalope;" -U postgres
```

### SQLite
    Database is created automatically if you specify driver and path ("pdo_sqlite", "jackalope.db"). Database name is not needed.

For further details, please see the [Doctrine configuration page](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/configuration.html#connection-details).


## Enable the commands

There are a couple of useful commands to interact with the repository.

To use the console, copy ``cli-config.php.dist`` to ``cli-config.php`` and configure
the connection parameters.
Then you can run the commands from the jackalope directory with ``./bin/jackalope``

NOTE: If you are using PHPCR inside of **Symfony**, the DoctrinePHPCRBundle
provides the commands inside the normal Symfony console and you don't need to
prepare anything special.

There is the Jackalope specific command ``jackalope:init:dbal`` which you need
to run to initialize a database before you can use it.

You have many useful commands available from the phpcr-utils. To get a list of
all commands, type:

```sh
$ ./bin/jackalope
```

To get more information on a specific command, use the `help` command. To learn
more about the `phpcr:workspace:export` command for example, you would type:

```sh
$ ./bin/jackalope help phpcr:workspace:export
```

# Bootstrapping

Jackalope relies on autoloading. Namespaces and folders are compliant with
PSR-0. You should use the autoload file generated by composer:
``vendor/autoload.php``

If you want to integrate jackalope into other PSR-0 compliant code and use your
own classloader, find the mapping in ``vendor/composer/autoload_namespaces.php``

Before you can use jackalope with a database, you need to set the database up.
Create a database as described above, then make sure the command line utility
is set up (see above "Enable the commands"). Now you can run:

```sh
$ bin/jackalope jackalope:init:dbal
```

Once these steps are done, you can bootstrap the library. A minimalist
sample code to get a PHPCR session with the doctrine-dbal backend:

```php
// For further details, please see Doctrine configuration page.
// http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/configuration.html#connection-details
$driver    = 'pdo_mysql'; // pdo_pgsql | pdo_sqlite
$host      = 'localhost';
$user      = 'jackalope';
$pass      = '';
$database  = 'jackalope'; // $path = 'jackalope.db'; // for SQLite
$workspace = 'default';

// Bootstrap Doctrine
$dbConn = \Doctrine\DBAL\DriverManager::getConnection(array(
  'driver'    => $driver,
  'host'      => $host,
  'user'      => $user,
  'password'  => $pass,
  'dbname'    => $database,
  // 'path'   => $path, // for SQLite
));

$factory = new \Jackalope\RepositoryFactoryDoctrineDBAL();
$repository = $factory->getRepository(
  array('jackalope.doctrine_dbal_connection' => $dbConn)
);
// dummy credentials to comply with the API
$credentials = new \PHPCR\SimpleCredentials(null, null);
$session = $repository->login($credentials, $workspace);
```

To use a workspace different than ``default`` you need to create it first. The
easiest is to run the command ``bin/jackalope phpcr:workspace:create <myworkspace>``
but you can of course also use the PHPCR API to create workspaces from your code.


# Usage

The entry point is to create the repository factory. The factory specifies the
storage backend as well. From this point on, there are no differences in the
usage (except for supported features, that is).

```php
// see Bootstrapping for how to get the session.

$rootNode = $session->getNode("/");
$whitewashing = $rootNode->addNode("www-whitewashing-de");
$session->save();

$posts = $whitewashing->addNode("posts");
$session->save();

$post = $posts->addNode("welcome-to-blog");
$post->addMixin("mix:title");
$post->setProperty("jcr:title", "Welcome to my Blog!");
$post->setProperty("jcr:description", "This is the first post on my blog! Do you like it?");

$session->save();
```

See [PHPCR Tutorial](http://phpcr.readthedocs.org/en/latest/book/index.html)
for a more detailed tutorial on how to use the PHPCR API.


# Performance tweaks

If you know that you will need many child nodes of a node you are about to
request, use the depth hint on Session::getNode.  This will prefetch the
children to reduce the round trips to the database. It is part of the PHPCR
standard. You can also globally set a fetch depth, but that is Jackalope
specific: Call Session::setSessionOption with Session::OPTION_FETCH_DEPTH
to something bigger than 1.

Use Node::getNodeNames if you only need to know the names of child nodes, but
don't need the actual nodes. Note that you should not use the typeFilter on
getNodeNames with jackalope. Using the typeFilter with getNodes to only fetch
the nodes of types that interest you can make a lot of sense however.


# Advanced configuration

## Logging

Jackalope supports logging, for example to investigate the number and type of
queries used. To enable logging, provide a logger instance to the repository
factory:

```php
$factory = new \Jackalope\RepositoryFactoryDoctrineDBAL();
$logger = new Jackalope\Transport\Logging\DebugStack();
$options = array(
  'jackalope.doctrine_dbal_connection' => $dbConn,
  'jackalope.logger' => $logger,
);
$repository = $factory->getRepository($options);

//...

// at the end, output debug information
var_dump($logger->calls);
```

You can also wrap a [PSR-3](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md)
compatible logger like [monolog](https://github.com/Seldaek/monolog) with the
Psr3Logger class.

Note that when using jackalope in Symfony2, the logger is integrated in the
debug toolbar.

## Custom UUID generator

By default, Jackalope uses the UUIDHelper class from phpcr-utils. If you want
to use something else, you can provide a closure that returns UUIDs as option
`jackalope.uuid_generator` to `$factory->getRepository($options)`


# Implementation notes

See [doc/architecture.md](https://github.com/jackalope/jackalope/blob/master/doc/architecture.md)
for an introduction how Jackalope is built. Have a look at the source files and
generate the phpdoc.

# Running the tests

Jackalope-doctrine-dbal is integrated with the phpcr-api-tests suite that tests
all PHPCR functionality.

If you want to run the tests, please see the [README file in the tests folder](https://github.com/jackalope/jackalope-doctrine-dbal/blob/master/tests/README.md).


# Things left to do

The best overview of what needs to be done are the skipped API tests.
Have a look at [ImplementationLoader](https://github.com/jackalope/jackalope-doctrine-dbal/blob/master/tests/ImplementationLoader.php)
to see what is currently not working and start hacking :-)

Also have a look at the issue trackers of this project and the base jackalope/jackalope.


# Contributors

* Christian Stocker <chregu@liip.ch>
* David Buchmann <david@liip.ch>
* Tobias Ebnöther <ebi@liip.ch>
* Roland Schilter <roland.schilter@liip.ch>
* Uwe Jäger <uwej711@googlemail.com>
* Lukas Kahwe Smith <smith@pooteeweet.org>
* Benjamin Eberlei <kontakt@beberlei.de>
* Daniel Barsotti <daniel.barsotti@liip.ch>
* [and many others](https://github.com/jackalope/jackalope-doctrine-dbal/contributors)