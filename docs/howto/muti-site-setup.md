---
title: Running Multiple Bolt sites from one source directory
---
Running Multiple Bolt sites from one source directory
=====================================================

When running multiple sites on the one host, it is sometimes advantageous to
only maintain a single directory for Bolt itself, and have each site in a
separate directory using the same Bolt install. This is known as a multi-site
set up.

This approach may have several advantages:
  * Easy Bolt upgrades for all sites at the same time
  * Security is slightly enhanced, as it keeps all executable code out of the
    web root directory
  * OpCache configuration is greatly simplified for multiple Bolt sites
  * A small amount of disk space is saved by not requiring multiple copies of the
    same Bolt and vendor libraries.

Set up base install
-------------------

First thing that needs to be done is the set up of the core Bolt files in a
central location, outside of the web root.

In this HOWTO we're going to use the location of `/var/www/bolt-private/`
but that is entirely up to your own needs.

Create and navigate to `/var/www/bolt-private/`

```
$ mkdir /var/www/bolt-private/
$ cd /var/www/bolt-private/
```

In this example, we're going to clone the Bolt git repository into the
`/var/www/bolt-private/` directory and install the vendor libraries with
Composer:

```
$ git clone https://github.com/bolt/bolt.git .
$ git checkout v2.2.9
$ composer install --no-dev --optimize-autoloader
```

**Note:** The above example assumes that you're wanting to use v2.2.9, the
latest version of Bolt at the time of writing, but you can change that version
tag to whatever version you desire.

Individual site set up
----------------------

Each virtual host site you set up will need a few things:
  * Site directories:
    * app
    * app/database
    * app/cache
    * app/config
    * extensions
    * files
    * theme
  * Site files:
    * index.php
    * app/bootstrap.php
    * app/nut
  * A template

### Site directory set up

In this HOWTO we're going to install each Bolt site in a directory under
`/var/www/sites/`, the first one in `/var/www/sites/my-sites-1`.

Create and navigate to `/var/www/sites/my-sites-1`:
```
$ mkdir /var/www/sites/my-sites-1
$ cd /var/www/sites/my-sites-1
```
Now create the required directories:

```
$ mkdir -p app/cache app/config app/database extensions files theme
```

Create a symbolic link to Bolt's templates:

```
ln -s /var/www/bolt-private/app/view/ app/view
```

### Install a theme

For this HOWTO, we'll just use the default `base-2014` theme and files. Copy
the `base-2014` directory into your site's `theme/` directory:

```
$ cp -a /var/www/bolt-private/theme/base-2014/ /var/www/sites/my-sites-1/theme/
$ cp -a /var/www/bolt-private/files/* /var/www/sites/my-sites-1/files/
```

### Site root index file

In the root of the sites web directory, create the `index.php` file and add
the following:

```php
<?php
$app = require_once __DIR__ . '/app/bootstrap.php';
$app->run();
```

### Create the application bootstrap

Next, create the site specific `app/bootstrap.php` file and add the following:

```php
<?php
return call_user_func(
    function () {
        // The full path to the core Bolt files
        $source = '/var/www/bolt-private';

        // The full path of the individual site
        $site = '/var/www/sites/my-sites-1';

        require_once $source . '/vendor/autoload.php';

        $configuration = new Bolt\Configuration\Standard(dirname(__DIR__));

        $configuration->setPath('root',    $source . '/');
        $configuration->setPath('app',     $source . '/app');
        $configuration->setPath('apppath', $source . '/app');

        $configuration->setPath('config',           $site . '/app/config');
        $configuration->setPath('cache',            $site . '/app/cache');
        $configuration->setPath('database',         $site . '/app/database');
        $configuration->setPath('extensionsconfig', $site . '/app/config/extensions');
        $configuration->setPath('extensionspath',   $site . '/extensions');
        $configuration->setPath('files',            $site . '/files');
        $configuration->setPath('themebase',        $site . '/theme/');
        $configuration->setPath('web',              $site . '/');

        $configuration->getVerifier()->disableApacheChecks();
        $configuration->verify();

        $app = new Bolt\Application(array('resources' => $configuration));
        $app->initialize();

        return $app;
    }
);
```

You just need to edit the `$source` and `$site` parameters to set them to the
correct directory locations for your individual site.

This is the one file you need to change for each site you set up using this
configuration.

### Setting up Nut

As a final step, each site will require a specific Nut command file should you
wish to use Nut.

To enable this, create the `app/nut` file and add the following:

```php
#!/usr/bin/env php
<?php

$app = require_once __DIR__ . '/bootstrap.php';

$nut = $app['nut'];
$nut->run();
```

Finally make the Nut file executable:
```
$ chmod +x app/nut
```
