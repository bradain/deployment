deployment
==========

Make deployment to an OpenBSD server easy (for me, anyway)

### The Problem
How can OpenBSD be re-installed onto a machine in a reliably consistent way, that includes all of the settings
and adjustments that have been made over time?

### My Solution
* Keep a record of all critical configuration files in a repository.
* Install a clean operating system, add necessary packages (including git etc)
* Get the repository files onto the machine in a staging area
* Run a shell script to deploy the configuration files to their correct locations
* Make it idempotent so we can run it as often as desired - it is a specification of how things should be.
* Explain what changes are being made each time the deployment runs

This script defines several ksh functions that make it easier to write the shell script that deploys the
staged files to the right locations. The focus is on making the deployment script succinct and readable.
Obviously it needs to run as root.

### Sample deployment script
The deployment script is a shell script so the usual rules apply. These functions just make it more succinct.

In this example, we pulled our repository files into `/var/staging/mymachine`, and we load the definitions of the helper functions:
```bash
# define where your staged files are found
DIR="/var/staging/mymachine"
# load the subroutines
. /var/staging/deploy.subr 

# need to define a group before a user
Group 1000 bradain
# create a user - before you deploy files that will belong to that user
User bradain 1000 bradain "Bradain Foley" Home=/home/bradain \
  Class=staff Shell=/bin/ksh Groups=wheel
```

Now let's get to the meat of installing files. Set some defaults for ownership and permissions.
You can change the defaults at any time, or override for a single folder. Symlinks can also be 
created:
```bash
export Owner="user" Group="user" Mode="0750"
Folder ~/
Folder ~/.ssh Mode=0700 # overrides the mode for this folder only
Folder ~/bin
export Owner="user" Group="user" Mode="0770"
cd $DIR/home/user
File ~/.profile
File ~/.xsession
Link /usr/local/bin/firefox \
  ~/.config/rox.sourceforge.net/SendTo/.application_pdf/firefox
```
Note that we change to the staging directory where the files are found. The `File` function looks in
the current directory (`$DIR/home/user`) for a file with the basename (`.profile`) and copies it
to the target location (`~/.profile`). It then sets the correct ownership and permissions on that file.

You can override for a **single** file in a set by appending `Mode`, `Group`, or `User` variables.

### Logs
What about logs files? We don't want to clobber those with some version from the repository.
We need to preserve the contents, but still ensure the ownerships and permissions are correct.
But there is no need to store an empty file in the repository - just append `Empty` to ensure that
the file is present but cleared. If `Empty Preserve` are both specified, then we'll create the file empty if its missing, 
or else we'll preserve the contents if its found.
```bash
File /var/www/logs/http-access.log Empty Preserve
```

`Empty` by itself will always clear the file, each time the deployment is run.
`Preserve` by itself requires an initial file to be kept in the repository, but won't change the deployed
file's contents each time you run the deployment.

### One caveat
There is one caveat: when specifying folders to be created, specify **all** intermediate folders, or else they 
will be created but may have the wrong ownership due to the nature of `install(8)`. For example:
```bash
export User="bradain" Group="bradain" Mode="0700"
# Wrong: ~/bin will be created automatically but owned by root!
Folder ~/bin/special
# Correct: creates ~/bin first then ~/bin/special 
Folder ~/bin
Folder ~/bin/special
```

### Other features
* check if a `cron` job is set up (does not change crontabs, just reports on missing jobs)
*	check if a `newsyslog(8)` entry exists (just reports missing items)
* there are some other functions (see `deploy.subr` for more details) that help in building a `chroot` gaol



