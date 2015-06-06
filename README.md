# vagrant-spk

`vagrant-spk` is a tool designed to help app developers package apps for [Sandstorm](https://sandstorm.io).

See also https://github.com/sandstorm-io/sandstorm/wiki/Porting-an-app-with-vagrant-spk for more details.

## Example usage:

    git clone git://github.com/zarvox/vagrant-spk
    git clone git://github.com/paulproteus/php-app-to-package-for-sandstorm
    export PATH=$(pwd)/vagrant-spk:$PATH
    cd php-app-to-package-for-sandstorm
    vagrant-spk setupvm lemp
    vagrant-spk up
    vagrant-spk init
    # edit .sandstorm/sandstorm-pkgdef.capnp in your editor of choice
    vagrant-spk dev
    # visit http://local.sandstorm.io:6080 in a web browser
    # log in as Alice, the admin account
    # launch an instance of the example app, play around with it
    # then, press Ctrl-C to stop the tracing vagrant-spk dev
    vagrant-spk pack example.spk
    # You now have an .spk file.  Yay!
    # Verify it works by going to http://local.sandstorm.io:6080,
    # select "My Files" -> "Upload an app", select your .spk file,
    # upload it, install it, and create a new instance of your app.

## What the files are for
`vagrant-spk`will create a `.sandstorm/` folder in your repo and set up some
files with some defaults for your app stack.  You will likely need to modify
some of these to adapt their behavior to make the most sense for your app.

### global-setup.sh
This installs Sandstorm using the official installer script, enables developer
accounts, and stops unneeded services on the VM.  It caches the sandstorm
bundle to speed up subsequent runs. You should not need to change this script.

### setup.sh
This script runs when you setup the VM by `vagrant-spk up` the first time and
does stack-specific setup, like installing tools or daemons needed to build and
run your app.  `vagrant-spk` provides some defaults based on commonly-used platform
stacks, but you'll likely want to modify this script as suitable for your package.

This is the ideal place to `apt-get install` system packages your app relies on,
like:

  - language runtimes (PHP, Node, Python, etc.)
  - database engines (MySQL, PostgreSQL, Redis, etc.)
  - frontend web servers (nginx, Apache)

and so on.  Please note that this script is run only once for the lifetime of
the VM, and will not be run again when running `vagrant-spk dev` or starting a
grain.

If you make changes to this script, you should destroy your VM (`vagrant-spk destroy`)
and rebuild it cleanly from scratch (`vagrant-spk up`) to make sure your
changes take effect and work as desired.

If you find yourself destroying and creating VMs from scratch frequently,
consider running an instance of `apt-cacher-ng` on your host to speed up
package downloads.

### build.sh
This script runs each time you run `vagrant-spk dev` before exposing your app
to the Sandstorm server, so you can run it in "dev mode".  Again, `vagrant-spk`
provides some defaults based on commonly-used patterns in the supported stacks,
but you'll likely need to modify this script to run your package's usual build
flow, since packages use many different workflows and directory structures.

This is the ideal place to invoke anything which is normally part of *your app's
build process*: anything that you need to transform your project's source code
into a runnable deployment, but explicitly *not* the project's deployment,
configuration, or user data.

Usually you put things here which should be run again as the result of changes
to your project's source code.  Examples of things you might put here are:

- Compiling your project from source, for projects written in compiled languages.
- Calling `composer` to install or update PHP dependencies described in your app's `composer.json`
- Calling `pip` to install your app's Python-specific dependencies from the `requirements.txt` in your app's repository
- Calling `npm install` to install or update npm dependencies from your app's `package.json`
- Calling `bower install` to install or update web/css dependencies described in the app's `bower.json`
- Calling `gulp` to compile and minify SASS/LESS into CSS, or collect javascript into bundles
- Minifying dependencies
- Collecting various build artifacts or assets into a deployment-ready directory structure

### launcher.sh
This script will be run every time an instance of your app - aka grain - starts
in Sandstorm.  It is run inside the Sandstorm sandbox.  This script will be run
both when a grain first launches, and when a grain resumes after being
previously shut down.  This script is responsible for *launching everything that
your app needs to run*.  The thing it should do *last* is:

- start a process in the foreground listening on port 8000 for HTTP requests.

Frequently this is something like `nginx` serving static files and reverse
proxying for some other backend service.  You want to run this last because
accepting requests on port 8000 is how you signal to the Sandstorm platform
that your application is completely up and ready for use.  If you do this
before your backend is ready to go, users could get e.g. 502 errors or see a
broken page on first load - a poor first experience.

Other things you probably want to do in this script include:

- Building folder structures in `/var`.  `/var` is the only non-tmpfs folder mounted R/W, and when a grain is first launched, it will start out empty.  It will persist between runs of the same grain, but be unique per app instance.
- Preparing a database and running migrations.  You can also manually generate some tables once, place them somewhere under `/opt/app`, and copy them to `/var/lib/mysql` if your app takes a while to do migrations, at the potential cost of producing a larger `.spk`.
- Launching other daemons that your app uses (`mysqld`, `redis-server`, `php5-fpm`, `uwsgi`, etc.)

For apps which need the ability to self-modify code or configuration, or which
expect to be able to write data underneath their source tree, you
should create a dangling symlink from
`/opt/app/where-your-app-keeps-its-self-modifiable-config.conf` to somewhere
under `/var`, then copy or generate a default configuration to that symlink target under `/var` in
`launcher.sh` so your app will find it at runtime.

There's an example of this in the paperwork repository -
[`build.sh`](https://github.com/JamborJan/paperwork/blob/cf4b11631e9cda9d45196b1a545a116376e630af/.sandstorm/build.sh#L35)
removes the folder `frontend/app/storage` and replaces it with a symlink
pointing to `/var/storage`.  Then,
[`launcher.sh`](https://github.com/JamborJan/paperwork/blob/cf4b11631e9cda9d45196b1a545a116376e630af/.sandstorm/launcher.sh#L16-24)
makes sure that `/var/storage` exists and is populated with the appropriate
subdirectories.

These tend to be unique per-app, so again, `vagrant-spk` provides appropriate
defaults for common stacks, but you'll likely need to make adjustments for your
app.

### sandstorm-files.list
This file is generated by running `vagrant-spk dev` and using the app.  It contains a list
of all files that your app used at runtime.  This is used to construct a
minimal package.  See
https://github.com/sandstorm-io/sandstorm/wiki/Porting-an-app-with-vagrant-spk

In the fullness of time, we'd like to support a method of generating
`sandstorm-files.list` that doesn't require the developer to carefully use
every app feature to make sure that e.g. default plugins get included in the
package.

### sandstorm-pkgdef.capnp
See https://github.com/sandstorm-io/sandstorm/wiki/Porting-an-app-with-vagrant-spk

### Vagrantfile
See https://github.com/sandstorm-io/sandstorm/wiki/Porting-an-app-with-vagrant-spk

## Example setups

### Default setup
Repo: https://github.com/paulproteus/php-app-to-package-for-sandstorm

This example shows how to setup a php + mysql app.

`setup.sh` installs PHP, nginx, and MySQL from the distribution's repository,
then modifies default config files to support the `/opt/app` layout and run
in the Sandstorm sandbox.

`build.sh` installs/updates composer, and uses composer to install PHP
dependencies.

`launcher.sh` creates a folder structure in `/var` for MySQL, nginx, and
php5-fpm, creates MySQL tables, then launches the three daemons, checking that
`mysqld` and `php5-fpm` are ready to accept requests before launching `nginx`,
which will listen for requests on port 8000.

### Paperwork (php, mysql, composer, npm)
Repo: https://github.com/JamborJan/paperwork

[`setup.sh`](https://github.com/JamborJan/paperwork/blob/master/.sandstorm/setup.sh)
installs PHP, nginx, nodejs, and npm.  Additionally, it installs some
system-global tools (`gulp` and `bower`) with `npm`.

[`build.sh`](https://github.com/JamborJan/paperwork/blob/master/.sandstorm/build.sh)
does several things: it installs and updates `composer`, installs app-specific
`npm` and `bower` dependencies from `package.json` and `bower.json` manifests
in the repo, and runs `gulp` to build static assets.

[`launcher.sh`](https://github.com/JamborJan/paperwork/blob/master/.sandstorm/launcher.sh)
creates the storage folders for notes in `/var/storage`, which the app will
find because `/opt/app/frontend/app/storage` (the standard storage location for
Paperwork) is a symlink to `/var/storage`.  Additionally, the script sets up
the default database, grants permissions, and runs migrations.
