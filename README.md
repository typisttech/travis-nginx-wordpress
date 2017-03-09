# Travis CI Nginx WordPress Test

[![Latest Stable Version](https://poser.pugx.org/typisttech/travis-nginx-wordpress/v/stable)](https://packagist.org/packages/typisttech/travis-nginx-wordpress)
[![Total Downloads](https://poser.pugx.org/typisttech/travis-nginx-wordpress/downloads)](https://packagist.org/packages/typisttech/travis-nginx-wordpress)
[![Build Status](https://travis-ci.org/TypistTech/travis-nginx-wordpress.svg?branch=master)](https://travis-ci.org/TypistTech/travis-nginx-wordpress)
[![PHP Versions Tested](http://php-eye.com/badge/typisttech/travis-nginx-wordpress/tested.svg)](https://travis-ci.org/TypistTech/travis-nginx-wordpress)
[![Latest Unstable Version](https://poser.pugx.org/typisttech/travis-nginx-wordpress/v/unstable)](https://packagist.org/packages/typisttech/travis-nginx-wordpress)
[![Dependency Status](https://gemnasium.com/badges/github.com/TypistTech/travis-nginx-wordpress.svg)](https://gemnasium.com/github.com/TypistTech/travis-nginx-wordpress)
[![License](https://poser.pugx.org/typisttech/travis-nginx-wordpress/license)](https://packagist.org/packages/typisttech/travis-nginx-wordpress)


A basic template for Nginx and WordPress running on Travis CI's container based infrastructure.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [What is the purpose of this repo?](#what-is-the-purpose-of-this-repo)
- [Installation and Usage](#installation-and-usage)
- [Customization](#customization)
- [How does it works?](#how-does-it-works)
  - [WordPress](#wordpress)
  - [Nginx](#nginx)
  - [Codeception](#codeception)
  - [Sauce Labs](#sauce-labs)
  - [Scrutinizer CI](#scrutinizer-ci)
- [Known Issues](#known-issues)
- [See Also](#see-also)
- [Change Log](#change-log)
- [Credit](#credit)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## What is the purpose of this repo?

Do you need to run some automated tests that rely on Nginx on Travis CI? Do you want those tests to run on the Docker
[container-based infrastructure](http://blog.travis-ci.com/2014-12-17-faster-builds-with-container-based-infrastructure/)?
Are you pulling your hair out trying to get this all to work? Then this repo is for you.

Travis CI does not come with Nginx pre-installed so the install needs to be scripted. Since Travis CI's container-based
infrastructure doesn't allow sudo privileges this installation is non-trivial. Hopefully, by providing this repo
I can save someone some hassle trying to get Nginx set up for their project.

## Installation and Usage

The script is tailored on Travis CI PHP build images and might not work in every situation. Below is an basic example of `.travis.yml`:

```

# .travis.yml
language: php

sudo: false

services:
  - mysql

cache:
  apt: true
  directories:
    - $HOME/.composer/cache/files

addons:
  apt:
    packages:
      - jq
      - nginx

php:
  - 7.0
  - 7.1

env:
  - WP_VERSION=latest
  - WP_VERSION=4.7.2

before_install:
  # Install helper scripts
  - composer global require -n --prefer-dist "typisttech/travis-nginx-wordpress:^1.0.0"
  - export PATH=$HOME/.composer/vendor/bin:$PATH
  - tnw-install-nginx
  - tnw-install-wordpress
  - tnw-prepare-codeception

install:
  # Build the test suites
  - cd $TRAVIS_BUILD_DIR
  - composer install -n --prefer-dist
  - vendor/bin/codecept build -n

script:
	# Run the tests
  - cd $TRAVIS_BUILD_DIR
  - vendor/bin/codecept run -n --coverage --coverage-xml

after_script:
 - tnw-send-result-to-saucelabs
 - tnw-upload-coverage-to-scrutinizer
 - tnw-upload-coverage-to-codecov

```

And, this is an basic example of `codeception.dist.yml` which compatible with the above Travis settings:

```

# codeception.dist.yml
actor: Tester
paths:
  tests: tests
  log: tests/_output
  data: tests/_data
  helpers: tests/_support
settings:
  bootstrap: _bootstrap.php
  colors: true
  memory_limit: 1024M
coverage:
  enabled: true
  include:
    - src/my-plugin/*
  exclude:
    - src/my-plugin/partials/*
params: [env] # get parameters from environment vars
modules:
  config:
    WPDb:
      dsn: 'mysql:host=localhost;dbname=wordpress'
      user: 'root'
      password: ''
      dump: 'tests/_data/dump.sql'
      url: 'http://wp.dev:8080'
    WPBrowser:
      url: 'http://wp.dev:8080'
      adminUsername: 'admin'
      adminPassword: 'password'
      adminPath: '/wp-admin'
    WordPress:
      depends: WPDb
      wpRootFolder: '/tmp/wordpress'
      adminUsername: 'admin'
      adminPassword: 'password'
    WPLoader:
      wpRootFolder: '/tmp/wordpress'
      dbName: 'wordpress_int'
      dbHost: 'localhost'
      dbUser: 'root'
      dbPassword: ''
      tablePrefix: 'int_wp_'
      domain: 'wordpress.dev'
      adminEmail: 'admin@wordpress.dev'
    WPWebDriver:
      url: 'http://wp.dev:8080'
      port: 4444
      window_size: '1024x768'
      adminUsername: 'admin'
      adminPassword: 'password'
      adminPath: '/wp-admin'
      # Sauce Labs
      host: '%SAUCE_USERNAME%:%SAUCE_ACCESS_KEY%@ondemand.saucelabs.com'
      browser: firefox
      capabilities:
          platform: 'Windows 10'
          version: '50.0'
          screenResolution: '1024x768'
          tunnel-identifier: '%TRAVIS_JOB_NUMBER%'
          name: 'WPBS %TRAVIS_JOB_NUMBER% on WP-%WP_VERSION% with PHP-%TRAVIS_PHP_VERSION%'
          build: '%TRAVIS_JOB_NUMBER%'

```

## Customization

The default scripts install WordPress core on `/tmp/wordpress` and serve it at `http://wp.dev:8080`.

You can customize the build via environment variables. Check the variables of the functions in the [bin](./bin) director for available configuration.

## How does it works?

All of the setup scripts are located in the [bin](./bin) directory and template files are in the [tpl](./tpl) directory. They are short and basic so it should be relatively easy to follow. The repository defines 6 setup scripts:

1. `tnw-install-wordpress`
    - Install WordPress
1. `tnw-install-nginx`
    - Setup Nginx to serve a website from a folder on a local domain
1. `tnw-prepare-codeception`
    - Install [PHP_CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer) and [WordPress coding standard](https://github.com/WordPress-Coding-Standards/WordPress-Coding-Standards)
1. `tnw-send-result-to-saucelabs`
    - Send Travis test result to [Sauce Labs](https://saucelabs.com/)
1. `tnw-upload-coverage-to-codecov`
    - Upload test coverage to [codecov.io](https://codecov.io)
1. `tnw-upload-coverage-to-scrutinizer`
    - Upload test coverage to [Scrutinizer](https://scrutinizer-ci.com)

### WordPress

The WordPress installation is done through the
[tnw-install-wordpress](./bin/tnw-install-wordpress) bash script. The basic install process goes as follows:

1. Create the WordPress database.
1. Download WordPress core.
1. Generate the `wp-config.php` file.
    - Note that `define( 'AUTOMATIC_UPDATER_DISABLED', true );` is added.
1. Install the WordPress database.

### Nginx

The Nginx installation is done through the
[tnw-install-nginx](./bin/tnw-install-nginx) bash script. The basic install process goes as follows:

1. Install Nginx using the [apt addon](https://docs.travis-ci.com/user/installing-dependencies/#Installing-Packages-with-the-APT-Addon) via entries in the [.travis.yml](./.travis.yml) file.
1. Copy the configuration templates to a new directory while replacing placeholders with environment variables.
1. Start php-fpm and Nginx with our custom configuration file instead of the default.

### Codeception

The Codeception preparation is done through the
[tnw-prepare-codeception](./bin/tnw-prepare-codeception) bash script. The basic install process goes as follows:

1. Replace `phantomjs` path to the TravisCI one in `codeception.yml` and `codeception.dist.yml`.
1. Create an extra database for testing.
1. Import database dump to WordPress.
1. Upgrade the WordPress database.
1. Export the WordPress database dump for later use.

Note: The `phantomjs` path must be wrapped in single quotes.

```
extensions:
  enabled:
    - Codeception\Extension\Phantoman
  config:
    Codeception\Extension\Phantoman:
      path: '/usr/bin/phantomjs'
      port: 4444
      suites: ['acceptance']
```

### Sauce Labs

The WordPress coding standard installation is done through the
[tnw-send-result-to-saucelabs](./bin/tnw-send-result-to-saucelabs) bash script. The basic process goes as follows:

1. Find Sauce Labs job id by `$TRAVIS_JOB_NUMBER`.
1. Determine test result by `$TRAVIS_TEST_RESULT`.
1. Send test result to Sauce Labs via REST API.

Note: You must annotate Sauce Labs job's `build` equals to `$TRAVIS_JOB_NUMBER` like so:

```
modules:
	config:
		WPWebDriver:
			capabilities:
				build: '%TRAVIS_JOB_NUMBER%'
```

### Codecov.io

The Codecov.io test coverage uploading is done through the
[tnw-upload-coverage-to-codecov](./bin/tnw-upload-coverage-to-codecov) bash script. The basic install process goes as follows:

1. Run the [codecov-bash](https://github.com/codecov/codecov-bash) script.

### Scrutinizer CI

The Scrutinizer CI test coverage uploading is done through the
[tnw-upload-coverage-to-scrutinizer](./bin/tnw-upload-coverage-to-scrutinizer) bash script. The basic install process goes as follows:

1. Download [ocular](https://github.com/scrutinizer-ci/ocular).
1. Upload the test coverage file to Scrutinizer CI.

## Known Issues

* Nginx gives alert messages during start which is safe to ignore.

	```
	$ install-nginx
  [26-Dec-2046 00:00:00] NOTICE: [pool travis] 'user' directive is ignored when FPM is not running as root
  nginx: [alert] could not open error log file: open() "/var/log/nginx/error.log" failed (13: Permission denied)
	```

## See Also

* [Running Nginx as a Non-Root User](https://www.exratione.com/2014/03/running-nginx-as-a-non-root-user/)
* [Travis CI Nginx Test (the original repo)](https://github.com/tburry/travis-nginx-test)
* [Travis CI Apache Virtualhost configuration script](https://github.com/lucatume/travis-apache-setup)

## Change Log

See [CHANGELOG.md](./CHANGELOG.md).

## Credit

[Travis CI Nginx WordPress Test](https://github.com/TypistTech/travis-nginx-wordpress) is originally forked from the [Travis CI Nginx Test](https://github.com/tburry/travis-nginx-test) project. Special thanks to its author [Todd Burry](https://github.com/tburry).

## License

[Travis CI Nginx WordPress Test](https://github.com/TypistTech/travis-nginx-wordpress) is released under the [MIT License](https://opensource.org/licenses/MIT).
