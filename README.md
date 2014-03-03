# Composer Manager

[Composer Manager](https://drupal.org/project/composer_manager) provides a
gateway to the larger PHP community by enabling [Drupal](http://drupal.org)
modules to more easily use best-in-breed libraries that are managed by
[Composer](https://getcomposer.org/).

There are [many challenges](#why-cant-you-just--) when using Composer with
Drupal, so the primary goal of this module is to work around the challenges by
wrapping common Drupal workflows around Composer so that so that module
developers can use the [thousands of standards-compliant, platform agnostic PHP
libraries](https://packagist.org/statistics) with as little friction as
possible.

## Installation

* Follow the [standard module installation guide](https://drupal.org/documentation/install/modules-themes).
* Refer to the [Usage For Site Builders](#usage-for-site-builders) section for
  installing and updating third-party libraries required by contributed modules.

## Usage For Site Builders

As modules are enabled and disabled, Composer Manager maintains a list of their
requirements. There are two ways to install and update the contributed
modules' dependencies:

### Automatically With Drush (Recommended)

Using `drush en` and `drush dis` to enable and disable modules respectively will
automatically generate the consolidated `composer.json` file and run the
appropriate Composer commands to install and update the required dependencies.

This technique introduces the least amount of friction with existing workflows
and is strongly recommended.

### Manually With Composer

If you do not wish to use Drush, you must manually use Composer's command line
tool to install and update dependencies whenever modules are enabled or
disabled. The following steps illustrate the workflow to maintain the
dependencies required by contributed module:

* Visit `admin/modules` and enable / disable the modules that have dependencies
* Change into the directory that the composer.json file was generated in as
  noted in the UI
* If necessary, [download and install](https://github.com/composer/composer/blob/master/doc/01-basic-usage.md#installation)
  the Composer tool
* Run `php composer.phar install --no-dev` on the command line, replace
  `install` with `update` when updating dependencies

Refer to [Composer's documentaton](https://getcomposer.org/doc/) for more
details on how Composer works.

## Why can't you just ... ?

The problems that Composer Manager solves tend to be more complex than they
first appear. This section addresses some of the common questions that are asked
as to why Composer Manager works the way it does.

### Why can't you just run "composer install" in each module's root directory?

If a module contains a `composer.json` file, running `composer install` in its
root directory will download all requirements and dependencies to `vendor/`
directories with their own autoloaders. Relying on this technique poses multiple
challenges:

* Duplicate library code when modules have the same dependencies
* Unexpected classes being sourced depending on which autoloader is registered
  first
* Potential version conflicts that aren't detected since each installation is
  run in isolation

To highlight the challenges, let's say `module_a` requires
`"guzzle/http": "3.7.*"` and `module_b` requires `"guzzle/service": ">=3.7.0"`.
At the time of this post, running `composer install` in each module's directory
will result in version 3.7.4 of `guzzle/http` being installed in `module_a`'s
`vendor/` directory and version 3.8.1 of `guzzle/service` being installed in
`module_b`'s `vendor/` directory.

Because `guzzle/service` depends on `guzzle/http`, you now have duplicate
installs of `guzzle/http`. Furthermore, each installation uses different
versions of the `guzzle/http` component (3.7.4 for `module_a` and 3.8.1 for
`module_b`). If `module_a`'s autoloader is registered first then you have a
situation where version 3.8.1 of `\Guzzle\Service\Client` extends version 3.7.4
of `\Guzzle\Http\Client`.

#### Composer Manager's Solution

Composer Manager finds all `composer.json` files in each enabled module's root
directory and attempts to gracefully merge them into a consolidated
`composer.json` file. This results in a single vendor directory shared across
all modules which prevents code duplication and version mismatches. For the use
case above, Composer will resolve both `guzzle/http` and `guzzle/service` to
version 3.7.4 which is a more consistent, reliable environment.

#### Challenges With Composer Manager

The challenge of Composer Manager's technique is when multiple modules require
different version of the same package, e.g. `"guzzle/http": "3.7.*"` and
`"guzzle/http": "3.8.*"`. Composer Manager will use the version defined in the
`composer.json` file that is associated with the module with the heaviest weight.
Composer Manager will also flag the potential version conflict in the UI so the
site builder is aware of the inconsistency.

The story at https://drupal.org/node/1931200 aims to provide manual resolution
via the UI, and in the future projects such as
https://github.com/dflydev/dflydev-embedded-composer might provide a better
solution to eliminate the need for a merging strategy in Composer Manager.

### Why can't you just manually maintain a composer.json file?

Manually maintaining a `composer.json` file provides a single library space that
all modules can share, however relying on this technique poses multiple
challenges:

* Site builder responsible for updating file when modules are enabled or updated
* Dependencies are decoupled from the module's codebase
* Multiple files must be maintained for each multisite installation

#### Composer Manager's Solution

Composer Manager automatically generates the `composer.json` file when modules
are enabled and disabled, and there is an option in the UI with a corresponding
Drush command that can rebuild the consolidated `composer.json` file on demand.
Furthermore, using a Drush based workflow will automatically run the appropriate
composer commands whenever modules are enabled or disabled, so the need to run
Composer commands outside of normal workflows is reduced to module updates.

For Drupal 8, Composer Manager also prevents the packages included in Drupal
core from being installed in the contributed vendor directory. It also ensures
that the dependencies are compatible with the versions included in Drupal core.
For example, if a module requires `"guzzle/service": "~3.0"`, version 3.7.1 will
be installed which is the version of the Guzzle components in core that
`guzzle/service` depends on.

@todo Provide technical details, reference https://drupal.org/node/2128353

#### Challenges With Composer Manager

There are multiple challenges posed by Composer Manager's technique:

* Web server needs write access to the composer file directory
* Sane multisite configuration requires environment-specific `settings.php`
  configurations
* Must implement `hook_composer_json_alter()` in a module to modify
  `composer.json`

### Why can't you just modify Drupal core's composer.json file (D8)?

Modifying Drupal core's `composer.json` file provides a single library space and
uses the autoloader that is registered in index.php. Relying on this technique
poses multiple challenges:

* Difficult to manage Drupal upgrades that have package updates
* Site builder responsible for updating file when modules are enabled or updated
* Dependencies are decoupled from the module's codebase
* Challenging in multisite environments where different packages / version are
  required

#### Composer Manager's Solution

Refer to the [Why can't you just manually maintain a composer.json file?](#composer-managers-solution-1)
section.

#### Challenges With Composer Manager

Refer to the [Why can't you just manually maintain a composer.json file?](#challenges-with-composer-manager-1)
section.
