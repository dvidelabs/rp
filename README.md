Overview
========

Loads environment key/values from multiple sources and runs a tool of
choice within the isolated environment of that tool. It also provides
a structure for storing specific files such as SSH keys that differ in
different scenarios (different profiles and systems).

The tool is intended to load access and encryption keys into a devops
tool like Ansible, cloud host provider libraries such as python boto
for AWS, or simply to use ssh more conveniently on multiple different
host systems.

It is especially useful to help separate system configuration from
user specific configuration where only the system configuration is
under shared source control, and it makes it easy to switch between
heterogenous systems such as multiple cloud hosting providers, or
datacenters.

This tool is no way tied to any particular other tool, but to
exemplify: Ansible easily handles different setups, but, assuming the
playbook rules are written as generic as possible, it still needs to
be told which configuration to use, and then have the right access
keys to do the job.

See 'rp --help' for brief description.


Introduction to the 'rp' or 'run-profile' tool
======================================================================

The 'rp' tool works a bit similar to the 'env' tool be setting up
a customized environment before running a command.


Installation
------------

The 'rp' file should be executable and placed in an executable path
such as '/usr/local/bin'.

Currently there are some associated files that must be located in the
same directory. These are named: 'run-profile*'.

It can also be copied directly into, say, ./bin of devops repository
and run as 'bin/rp'. The user specific setup evolves around
configuring ~/.devops/profiles - this is discussed at length below.


Maintainence
------------

This tool is essentially viewed as complete and will only be updated
if significant bugs are found, or new features become relevant.


Use Case
--------

A typical (simplified) use case could be to give AWS_SECRET_KEY and
friends to the python boto library, give network topology to ansible,
give ssh.config to ssh while allowing for multiple system
configurations and multiple possible user profiles, making core
deployment scripts reusable.


Motivation
----------

It is convenient to write a few scripts to lauch a tool in different
configurations. For example ssh into different cloud providers are
using ansible roles in different contexts. This in itself can quickly
explode into many unmaintable scripts, even if using a very structured
approach such as ansible overall.

Furthermore, some data such as host IP addresses may need frequent
and expedient synchronization across teams by using a source code
repository such as git, while other data such
user account data does belong there, and may provide a security risk.

Even when these user specific data are separated out, the lack of
structure and standardization makes it difficult to set up a
consistent framework and typically requires a lot of written
instructions and easily broken scripts.

In the end, a job needs to done, and the end result may be that data
are hacked into the wrong places, which then easily leads to the other
meaning of the word hacked as critical data are exposed in hosted
source code repositories.

Using this tool will not solve all problems, but it becomes easier to
structure things right, for example by documenting that a user profile
must define "SSH_PRIVATE_KEY" and have it point to a valid key. This
can then by done by dropping the key into the profile home and store
the name without being concerned with global path conventions. Here
is an arbitrary example:

    $ cat ~/.devops/profiles/examples/dc01/profile.env

    # copy to ~/.devops/profiles/<myname>/dc01
    # log in to management console to get key and credentials
    # fill in below...
    
    SSH_PRIVATE_KEY={{ profile_home }}/mykey.pem


    # IMPORTANT: only set this to yes if you need to maintain system:
    # dc-polar/bonzai with database backup jobs (TODO: we need to
    # clean this up and create a new system config with the proper
    # setting).
    # FIXME_SSH_FORWARD_AGENT=yes
    FIXME_SSH_FORWARD_AGENT=no

    # Remove this for internal datacenters such as dc-tribal and dc-polar 
    AWS_SECRET_KEY=...
    AWS_ACCESS_KEY=...
    AWS_REGION=...
    
    # This must be a valid path in trainee-devops/systems:
    DEVOPS_SYSTEM=aws/test-01
    # DEVOPS_SYSTEM=polar-bonzai


By sharing an example profile or a tree of such, documentation
requirements are kept to a minimum, and users can update their own
credentials as they change or become available.

The user of the above might then run:

    $ cp -r ~/.devops/profiles/examples/dc01 ~/.devops/profiles/doe/test-dc01
    (edit and add key)
    $ cd trainee-devops
    $ ls systems
    aws/
    polar-bonazai

    $ rp -p doe/test-dc01 ansible -m ping all

    (systems/aws/test-01/system.env would have settings needed by ansible)


Setup
-----

Before running a configuration management (or any other) tool, the
environment must be setup.

Tools using this setup will load the environment with items such as
secret keys and also choose the associated system configuration just
before running the tool Thus, keys are not lingering in the shell
environment, and are kept in a single controlled location outside of
shared source repositories.

We thus have two configuration parts: systems and profiles.

The systems define network topology, service provider, firewall rules
etc.  The profiles define access credentials and customized tooling
options. Often there a profile close match one system, and thus a
system is selected by selecting a profile.

Profiles are stored outside of the primary devops source control
system and holds access keys and local encryption keys. Ideally, the
profile can be reconstructed using two factor login to a management
console to retrieve access keys, define them in a small profile.env
file and define which system the keys apply to - and possibly an ssh
key and encryption key shared by gpg email or similar, but of course
it depends.

Systems are defined inside devops repositories and may by synchronized
very frequently, for example changing the number of load-balanced web
servers.  Perhaps some settings are encrypted - we might for example
want to hide the IP of our primary access hosts, but that depends on
the tooling used.

In the basic form, we have

    ~/.devops/profiles/profile.env

or, if DEVOPS_PROFILE=dc01_shared (as an example)

    ~/.devops/profiles/dc01_shared/profile.env

or

    ~/.devops/profiles/user-doe/dc01_test/profile.env

Here DEVOPS_PROFILE_ROOT defaults to '~/.devops/profiles', and in the
exampels, DEVOPS_PROFILE was one of

    DEVOPS_PROFILE=""
    DEVOPS_PROFILE=dc01_shared
    DEVOPS_PROFILE=user-doe/dc01_test

We can override the profile root path so we find the profile with

    $DEVOPS_PROFILE_ROOT/$DEVOPS_PROFILE/profile.env

The file $DEVOPS_PROFILE_ROOT/common_profile.env is optionally read
first and may be used to select the currently active profile by
defining DEVOPS_PROFILE.

As an example, profile.env may hold tool and system specific
credentials such as:

    AWS_SECRET_KEY=xxx
    AWS_ACCESS_KEY=xxx
    AWS_REGION=xxx

Variables are entirely specific to the system tool chain, the python
boto library seems to take a preference for the above. The DEVOPS_
prefix is reserved for our environment setup use.

With the profile configured, a system can be identified. This is
normally a directory in a checked out configuration management system
and decoupled from the profile, but this is merely a convention:

After 'profile.env' has been read, 'system.env' is read (when present)
in the path:

    $DEVOPS_SYSTEM_ROOT/$DEVOPS_SYSTEM/system.env

with the system root path default to './systems' and the system
defaults to empty, so we the default path becomes:

    ./systems/system.env

Like with profiles, if a common_profile.env file is found in the
system root, it is read first, and may be used to select the active
system.  For example, we might have the following files read in that
order:

    ~/.devops/profiles/common_profile.env
    ~/.devops/profiles/dc01_shared/profile.env
    ./systems/common_system.env
    ./systems/datacenter_01/system.env

NOTE: this order allows for critical security parameters to be
confidently set in the systems.env setting regardless of what
individual users define in their profile or environment. This could,
as an example, be shutting off an SSH_FORWARD_AGENT setting that was
previously configured in user profiles. If the opposite order is
needed, use different names and deal with it in a script using
'OPTION_X=${PROFILE_X:-SYSTEM_X}'.

More generally we have:

    $DEVOPS_PROFILE_ROOT/common_profile.env
    $DEVOPS_PROFILE_ROOT/$DEVOPS_PROFILE/profile.env
    $DEVOPS_SYSTEM_ROOT/common_system.env
    $DEVOPS_SYSTEM_HOME/$DEVOPS_SYSTEM/system.env

with

    DEVOPS_PROFILE_HOME=
        $DEVOPS_PROFILE_ROOT/$DEVOPS_PROFILE

    DEVOPS_SYSTEM_HOME=
        $DEVOPS_SYSTEM_ROOT/$DEVOPS_SYSTEM

If a value is defined in multiple places, the last value read will
win, except the following specific variables take precedence from the
command line arguments, then from the existing environment:

    DEVOPS_PROFILE_ROOT
    DEVOPS_PROFILE
    DEVOPS_SYSTEM_ROOT
    DEVOPS_SYSTEM

Defaults summary:

    DEVOPS_PROFILE_ROOT=~/.devops/profiles
    DEVOPS_SYSTEM_ROOT=./systems

The following variables are temporarily overwritten, or rewritten with
absolute paths when running the command given: 

    DEVOPS_PROFILE_ROOT
    DEVOPS_SYSTEM_ROOT
    DEVOPS_PROFILE_HOME
    DEVOPS_SYSTEM_HOME

These values are also avaible to some of the .env files as template
arguments; for example {{ system_home }} is available to 'system.env'.
See support matrix below.

All .env files are always only looked up in exactly one place each,
and skipped when absent.

Both the profile and system directories may contain additional setup
files, and the .env files may be used to identify these with the
help of template arguments.


Example
-------

Lets run an example with the python boto library on OS-X to list
hosts on the AWS cluster.

    $ easy_install pip
    $ pip install boto
    (possibly also set up a python path)

Now the tool 'list_instances' should be able to list stopped and
running AWS EC2 instances, except we have to define a number of
AWS keys first.

We edit common_profile.env for a quick test:

    $ cat ~/.devops/profiles/common_profile.env
    AWS_SECRET_KEY=xxx
    AWS_ACCESS_KEY=xxx
    AWS_REGION=xxx
    DEVOPS_SYSTEM=aws/test

We can now do:

    $ rp list_instances

Notably without AWS_SECRET_KEY floating around in your shell
environment.

So far this only details the profile part. But we also wants an
ssh.config file that can be shared in a devops repository with
quickly chaning host names so we can easily log in. This is not
suitable for the profiles directory which we keep private.

Without going into specifics about ssh, we create a devops repository,
using git or similar. For simplicity we assume we operate from the
root of this repository and add the systems subdirectory:

    $ mkdir -p devops/systems/aws/test
    $ cd devops
    $ touch systems/aws/test/ssh.config
    $ touch systems/aws/test/system.env

Edit ssh.config to your specifics.
Edit system.env so we get:
    
    $ cat systems/test/system.env
    SSH_CONFIG={{ system_home }}/ssh.config
    SSH_PRIVATE_KEY={{ profile_home }}/key.pem
    
Copy you ssh key in place so we have:

    $ ls ~/.devops/profiles/
    common_profile.env
    key.pem

And finally we create a login script in ./bin/login

    $ cat ./bin/login 
    #!/bin/sh
    ssh-add $SSH_PRIVATE_KEY
    ssh -f $SSH_CONFIG $1

Note that we use ssh-add so ssh.config might use ForwardAgent=yes if
necessary. The script can be enhanced to deregister they key from the
agent when done.

Assuming ssh.config lists a host named 'test-server-01' we login wit:

    $ rp bin/login test-server-01

As an excercise, rename the common_profile.env to profile.env and
place it in a profile named 'test-01' in 'devops', and another named
'production' using a different key.  With that configured, and
ssh.config updated, we can now do:

    $ rp --profile test-01 bin/login test-server-01
    $ rp --profile production bin/login production-host

Set up a new AWS account with a new profile. This requires a new
systems dir 'aws/2', and a new profile to hold the AWS keys. With that
configured, we can name the new profile 'admin-02' that sets
DEVOPS_SYSTEM=aws/2.

    $ rp -p admin-02 list_instances
    $ rp -p test-01 list_instances

Next steps:

Use a tool like Ansible and call it with a shell script that feeds
--extra-vars with a file from $DEVOPS_SYSTEM_HOME, and possible
configures an encryption key given in the profile.env file.

Assuming we have a basic ansible setup

Finally, create a third system aws/lock-down which uses the same
account so we can reuse keys and profile, but change some ssh.config
parameters, and perhaps later some firewall rules. We also add
a hosts file for use with ansible.

Gotcha:

    $ rp -p admin-02 -s aws/3 ansible \
        -i $DEVOPS_SYSTEM_HOME/hosts -m ping all

Note that we avoid variable expansion on the command line where they
are not defined, and that run has added its own location to the
executable path temporarily.

We can work around this by placing the command in a shell script,
but since this is an ad-hoc job, we can work around it with:

    $ rp -p admin-02 -s aws/3 -i systems/aws/3/hosts -m ping all

For ad-hoc testing we can also drop into a sub-shell:

    $ rp -p admin-2 -s aws-3 bash

(advanced user might want to change the prompt at the same time)

All the secrets exposed, but we can run all commands directly:

    $ ansible -i $DEVOPS_SYSTEM_HOME/hosts -m ping all
    ...
    
and exit when done with ad-hoc testing

    $ exit

To get a more concise workflow, still with flexible profiles
we add a wrapper script for key tasks such as ansible-playbook:

    $ cat bin/play
    #!/bin/sh

    ansible-playbook -i ${DEVOPS_SYSTEM_HOME}/hosts "$@" \
        --extra-vars="@${DEVOPS_SYSTEM_HOME}/vault/vars.yml

    $ ls roles
    myplay.yml

    $ rp bin/play roles/myplay.yml
    $ rp -p test bin/play roles/myplay.yml
    ...

where the vault encryption key might have been set in the user profile
to protect the system configuration. The "$@" is shell speak for
passing the arguments from run to ansible-playbook.


Evaluation Order
----------------

The following values are decided as .env files are being read
and cannot be changed once decided. The table lists the places
they can be defined, and the first place to set it, wins.

    DEVOPS_PROFILE_ROOT    <env>
    DEVOPS_PROFILE         <arg>, <env>, <common_profile.env>
    DEVOPS_SYSTEM_ROOT     <env>, <common_profile.env>, <profile.env>
    DEVOPS_SYSTEM          <arg>, <env>, <all other env files>

Where <env> is the existing shell environment and <arg> is a command
line option given the 'rp' tool, such as '--system=mystem'.

To see have this works, use the debug facilities:

    $ rp -d
    $ rp -d -e

Other values may be set repeatedly, and the last value will win,
which gives predictability, but isn't very useful when looking up
which profile or system to use.

The behavior of other DEVOPS_ variables are reserved for future use.

The values:

    DEVOPS_PROFILE_HOME
    DEVOPS_SYSTEM_HOME

are derived from the above and can never be set.

Note that:j;w

    $ rp --system=online-services sh

will start a new shell with the profiled environment. Therefore, if
the 'rp' tool is used again, it will inherit the settings from the
parent call, which can be very useful, but when it is not, the
following example shows different ways to override settings:

    $ unset DEVOPS_PROFILE_ROOT
    $ DEVOPS_SYSTEM_ROOT=../staging rp --profile=test


File Syntax
-----------

The file consists of blank lines, comments and key value pairs:

    # Comment
       # comment

    Key=Value
    sentence_1=More than { on e word
    login_file={{ profile_home }}/login.sh

Blank lines, and comments are ignored.
Keys must start at the beginning of the line.

Comments cannot appear on key=value lines - they would be part of
the value. This allows for passwords containing special characters.

Trailing space is stripped from values, but otherwise they can
contain spaces and special characters. If you have a file name
with a trailing space, or a password, you have to define your
delimiter and strip it later, or store the value in a file.

Template arguments are mustache and Jinja2 style. The convention
is one space before and after keyword. Unsupported template
arguments are passed through.


Template Arguments
------------------

Each .env file may access a few variables when they do not define
the values within their own file:

    variable                valid for 
    -------------------------------------------------------
    {{ project_home }}      all
    {{ profile_root }}      common_profile.env, profile.env
    {{ profile_home }}      profile.env
    {{ profile }}           profile.env
    {{ system_root }}       common_system.env, system.env  
    {{ system_home }}       system.env
    {{ system }}            system.env

No other template arguments are allowed - they are passed unchanged.

Don't confuse {{ project_home }} with {{ profile_home }}. The project
home is the current directory unless a parent directory contains the
file `.devops.conf` in which case that directory becomes the project
directory. Various default paths rely on the project home path. Like
all home paths, the project home path is computed and cannot be changed.

The reason why systems do not see profiles is to separate concerns and
ease maintenance. Tools have access to all paths via the environment.

Sometimes it useful to have shared settings across systems or
profiles. In that case one can use the root paths, or the more
fragile home paths such as:

    AWS_SHARED_VARS_1={{ system_home }}/../shared.vars
    AWS_SHARED_VARS_2={{ system_root }}/aws/shared.vars


The .env files have no leading space and consists of lines of key
value pairs.  Blank lines and lines starting with # (only at line
start), are valid and ignored. Variables have exactly one space before
and after the name.

    # this is a comment

    DEVOPS_SYSTEM=datacenter_01
    DEVOPS_USER_NAME="Shared Devops User"
    DEVOPS_PROFILE=dc01_shared
    SSH_PRIVATE_KEY={{ profile_home }}/{{ profile }}/dc01_shared.pem
    SSH_DEFAULT_PRIVATE_KEY={{ profile_home }}/shared.pem

    # end of example

The convention says one space before the keyword such as {{
profile_home }}, by spaces are ignored. During export (see below)
without expansion, template arguments are rewritten with exactly
one space on either side of the keyword, making it easier to parse.


SSH Agent
---------

This is not strictly related to this tool, but it is mentioned as it
is closely related. For example in the choise of where to put keys
and whether key paths should be configured at all.

SSH keys are normally, or at least sometimes, expected to be added to
ssh agent, and configuration management systems may forward keys to
remote hosts via ssh ForwardAgent=yes. Therefore it makes sense to
password protect keys even when remote systems cannot have access
to that password. On OS-X one can add keys to the keychain with:

    $ ssh-add -K mykey.pem 

and in general without -K to add it to the running ssh-agent during
local login.  Some tools may script this using the variables such as
in the above.


rcm and git for profiles
------------------------

As a final note, the profile directory structure is designed to be
compatible with the rcm tool:

    github.com/thoughtbot/rcm

rcm synchronizes dot dirs across machines for different users and
system types, but it is not necessarily recommended unless the
security implications of multiple key locations are well understood.
Note that 'man' pages are currently the best source of documentation
for that tool.

We may also define a git repository with only example profiles and
explicitly .gitignore the actual profiles. This makes it easy to share
configuration templates, and changes to those, in relative safety.


Default Command
---------------

If no command argument is given, 'env' is called to dump the profiled
environment, unless an export option is given.

In this way it is easy to find, say, a db password:

    $ rp -p backup | grep DB_PASSWORD
    DB_PASSWORD=not-secret-anymore

or the AWS key setup:

    $ rp | grep AWS_
    AWS_ACCESS_KEY=...
    AWS_SECRET_KEY=...
    AWS_REGION=...


Export
------

It is possible to export values from the various sources. The output
is in ini style format, but is easily converted to a shell source
that will read the same environment as rp configures.

If any export option is used, it is not valid to supply a command. No
environment will be created during export.

Here is an example:


    $ rp -p doe/production-dc01 -e
    [common_profile.env]
    DEVOPS_PROFILE=doe/test-dc01
    
    [profile.env]
    AWS_ACCESS_KEY=...
    AWS_SECRET_KEY=...
    AWS_REGION=...
    SSH_PRIVATE=...../doedc01key.pem
    DEVOPS_SYSTEM=dc01/production

    [system.env]
    SYS_STATUS=TEST

    [meta]
    DEVOPS_PROFILE_ROOT=/home/doe/.devops/profiles
    DEVOPS_PROFILE_HOME=/home/doe/.devops/profiles/doe/production-dc01
    DEVOPS_PROFILE=doe/production-dc01
    DEVOPS_SYSTEM_ROOT=/home/doe/devopswork/dcx-project/systems
    DEVOPS_SYSTEM_HOME=/home/doe/devopswork/\
        dcx-project/systems/dc01/production
    DEVOPS_SYSTEM=dc01/producion

The last [meta] section is the actual settings used, regardless of
source. Here we have used the command line to override the
DEVOPS_PROFILE in common_profile.env. Also note that the
[common_system.env] section is absent because in this case there
were no such file.

The format is stripped for comments and blank lines and have exactly
one blank line between sections.

The section headers and blank lines can be stripped with:

    $ rp -x

This is almost enough to use as a shell source file, but special
characters are not escaped.

Output can also be directed to a file with the -o option, and this
option implies the -e, so we can have:

    $ rp -o "myenv.ini"
    $ rp -x -o "myenv.sh"

Last, but not least, we can disable template expansion which is useful
for sharing and reviewing configurations, but of course, mask
sensitive data first.

The following example only shows a few lines where relevant:

    $ rp -n
    [profile.env]
    ...
    SSH_PRIVATE={{ profile_home }}/keys/doedc01key.pem
    ...

    [meta]
    DEVOPS_PROFILE_ROOT={{ profile_root }}
    DEVOPS_PROFILE_HOME={{ profile_home }}
    DEVOPS_PROFILE=doe/production-dc01
    DEVOPS_SYSTEM_ROOT={{ system_root }}
    DEVOPS_SYSTEM_HOME={{ system_home }}
    DEVOPS_SYSTEM=dc01/producion



