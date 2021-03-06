==========
User Guide
==========

Configuration
-------------

Anteaters configuration exists witin ``anteater.conf``::

    [config]
    anteater_files = anteater_files/
    reports_dir = %(anteater_files)s.reports/
    anteater_log = %(anteater_files)s/.reports/anteater.log
    flag_list =  %(anteater_files)s/flag_list.yaml
    ignore_list = %(anteater_files)s/ignore_list.yaml
    vt_rate_type = public

* ``anteater_files``: Main location to store anteater ``flag_list``,
  ``ignore_list`` and reports. This location is ignored by anteater when
  performing scans.
* ``reports_dir``: location for anteater to send reports
* ``anteater_log``: anteater application logging output file.
* ``flag_list``: Regular Expressions to flag. See RegExp Framework.
* ``ignore_list``: Regular Expressions to overwrite / cancel ``flag_list``.
* ``vt_rate_type``: ``public`` or ``private`` VirusTotal API limiting.

The ``anteater.conf`` file should always be in the directory from where the
anteater command is run from. ``anteater`` will look for ``anteater.conf``
in the present working directory.

Methods of Operation
--------------------

Anteater uses a simple argument system in the standard POSIX format.

The main usage  parameters are ``--project`` and either ``---path`` or
``--patchset``.

Optional parameters are ``--binaries`` which is the binary check system. When
this argument is passed, all binaries / blobs will result in a VirusTotal scan 
- unless a sha256 checksum of the binary is listed in one of the exeception
files (``ignore_list`` or a ``project_exceptions`` file. ``--ips`` peforms a
scan of IP addresses, and ``--urls`` for any URL's found within file contents.

Refer to `binary exceptions`_ for more details on the binary blocking feature of
anteater.

The ``--project`` argument
--------------------------

Anteater always requires a project name passed with the ``--project`` argument.
This should be the same as the name as your repository. So for example, if your
git repository and its root folder are named 'acme', then you
pass ``--project acme``.

Having a project parameter allows for a scenario of multiple projects (for
example when using gerrit).

The ``--project`` parameter maps to several areas:

* Reports naming convention (for example ``contents_<project>.log``)

* dealing with a relative path (we strip out the full path, to allow people to
  enter filenames with a path relative to the repository). This is useful for
  when running locally (where every user will have their own unique ``$HOME``).

* project exceptions::

    project_exceptions:
      - myrepo: anteater_files/myrepo.yaml

.. Note::

    See `Exceptions`_ for more details.

The ``--patchset`` and ``--path`` arguments
-------------------------------------------

Anteater can be run with two methods, ``--patchset`` or ``--path``.

When ``--patchset`` is passed as an argument, it is expected that a text file be
provided that consists of a list of files, using a relative or full path.
Anteater will then iterate scans over each file, with the files seperated by
a new line. For example::

    % cat /tmp/patchset
    /path/to/repos/myrepo/fileone.sh
    /path/to/repos/myrepo/filetwo.sh
    /path/to/repos/myrepo/filethree.txt

The patchset is typically generated by another system, with git being a good
example and allowing a complete pull request to be iterated over::

    git diff --name-only HEAD^ > /tmp/patchset

This would then be called with::

    anteater --project myrepo --patchset /tmp/patchset

When ``--path`` is  provided, the argument should be a single relative or full
path to your repositories folder. Anteater will then perform a recursive walk
through all files in the respository folder. For example::

    anteater --project myrepo --path /path/to/repos/myrepo

Having these two methods allows anteater to scan individual pull requests /
patch sets or perform a complete audit on existing files.

RegExp Framework
================

The RegExp Framework is set of a YAML formatted files which are declared in
``anteater.conf`` under the directives ``flag_list`` and ``ignore_list``, as
well as ``project_exceptions`` embedded within ``ignore_list``.

There is a simple hierarchy with these files, with ``ignore_list`` and the
contents within ``project_exceptions`` "stacking" on top.

All RegExp files should be stored in the  set location of ``anteater_files``
that is declared in ``anteater.conf`` - this is important, as ``anteater_files``
is ignored by anteater during all scanning operations, thereby stopping anteater
falsely flagging its own strings set within ``flag_list``.

flag_list
---------

``flag_list`` is a complete list of all regular expressions, that if matched
within any file content or binary / file name, will cause anteater to exit with
a sys code of ``1``, thereby causing a build failure within a CI system (such as
jenkins / Travis CI).

``flag_list`` should be considered a list of strings or object namings that you
do not want anyone to merge into a repository, a blacklist essentially. This
could include security objects such as private keys, binaries or depreciated
functions, modules, libaries. Basically anything that can be matched using
standard regular expression syntax.

Within ``flag_list`` are several parameters set within YAML list formats.

file_names
-----------

``file_names`` is a list of full file names to flag. For example, the following
would flag someone's shell history if included in a pull request / patch::

    file_audits:
        file_names:
          - (irb|plsq|mysql|bash|zsh)_history

So if a user then accidentally checks in a ``zsh_history`` then anteater will
flag this, the build will fail and prevent an oversight from happening and the
file being merged into main branches.

file_contents
-------------

``file_contents`` is a list of regular expression strings that will be searched
for within any file that is not a binary / blob - this could be text files,
documentation, shell scripts, source code etc.

The structure of the file is as follows::

    file_audits:
      file_contents:
        unique_name:
            regex: <Regular Expression to Match>
            desc: <Line of text to describe the rationale for flagging the string>

The following would be examples for ensuring no insecure cryptos are used and
a depreciated function is also flagged::

  file_contents:
    md245:
      regex: md[245]
      desc: "Insecure hashing algorithm"

    depreciated_function:
      regex: depreciated_function\(.*\)
      desc: This function was depreciated in release X, use Y function.

So the above would match and flag the following lines::

    hashlib.md5(password)

    dothis = thing.depreciated_function(some_value):

Exceptions
----------

Exceptions are essentially a regular expression that provides a waiver to
strings that are flagged as false postives.

Exceptions can be made in two locations ``ignore_list`` or ``project_exceptions``
set within ``ignore_list`` and allows you to overule a string set within the
``flag_list`` file with a more unique regular expression.

There are main three sections within ``ignore_list.yaml`` and
``project_exceptions``

* ``file_contents`` - ignore matching regex if matched in a certain file.

* ``file_names`` -  ignore matching regex when it matches a file name.

* ``binaries`` - allow binaries, when they have a matching sha256 checksum set.

Project Exceptions
------------------

If you're a single project, then you can place all of the above three sections
into ``ignore_list.yaml``. If you have to manage multiple projects, then use
``ignore_list.yaml`` as a global master list, and use a ``project_exceptions``
entry for each individual project. For example, within your ``ignore_list.yaml``
you can declare each projects exeception list as follows::

    project_exceptions:
      - acme:   anteater_files/acme.yaml
      - bravo   anteater_files/bravo.yaml
      - charlie anteater_files/charlie.yaml


file_contents exceptions
------------------------

``file_contents`` exceptions are used to cancel out a ``flag_list`` entry by
using a regular expression that matches a unique string that has been
incorrectly flagged and is a false positive.

Let's say we wish to have some control over git repositories that can be cloned
in shell scripts present in out repository and used to automate our builds.

First we make an entry in the ``flag_list`` around git clone::

    file_contents:
      clone:
        regex: git.*clone
        desc: "Clone blocked as using an non approved external source"

The above would flag any instance of a clone, for example::

    git clone http://github.com/no_longer_around/some_unmaintained_repo.git

Now let's assume we want to allow all clones from a specific github org called
'acme' which we trust, but no other github repositories.

We could do this by using the following Exception::

    file_contents:
      - git clone https:\/\/github\.com\\acme\\.+

This would then allow the following strings::

    git clone https://github.com/acme/repository
    git clone https://github.com/acme/another_repository

Let's look at an example again using the md5 flag::

    file_contents:
      md245:
        regex: md[245]
        desc: "Insecure hashing algorithm"

The above ``file_contents`` expression would incorrectly match the following
string::

    mystring = int(md500) * 4

In this case ``md500` is incorrectly matched against ``md5``.

We can cancel out this false postive with a regular expression unique to the
incorrectly flagged false positive::

    file_contents:
      - mystring.=.int\(md500\).*

.. Note::
    You can test strings out on an regex site such as https://regex101.com

file_names exceptions
---------------------

As with ``file_contents``, ``file_names`` incorrectly flagged as false postives may
also be removed using a regular expression.

Public IP Addresses
-------------------

If `--ips` is passed as arguments, anteater will perform a scan for 
public / external IP Addresses. Once an address is found, the IP is sent to 
the Virus Total API and if the IP Address has past assocations with malicous 
or malware hosting domains, a failure is registered and a report is provided.

An example report can be seen `here <https://www.virustotal.com/#/ip-address/90.156.201.27>`_.

URLs
----

If ``--urls`` is passed as arguments, anteater will perform a scan for URL's. 
If an URL is found, the URL is sent to the Virus Total API which then 
compares the URL to a large list of URL blacklisting services.

An example report can be seen `here <https://www.virustotal.com/#/url/fb69ecad84eb86b1afddcca17aec38daea196e7c883b22ff88a7c39fd8fbdf1a/detection>`_.

binary exceptions
-----------------

If the ``--binaries`` argument is passed to anteater, anteater blocks (CI build
failure) all binary files unless a sha256 checksum of the file is entered as an
exeception. If no checksum is present, the binary (hash) is also sent to 
the VirusTotal API.

This is done using the relative path from the root of the respository.

For example::

  media/images/weather-storm.png:
    - 48f38bed00f002f22f1e61979ba258bf9006a2c4937dde152311b77fce6a3c1c
  media/images/stop_light.png:
    - 5a1101e8b1796f6b40641b90643d83516e72b5b54b1fd289cf233745ec534ec9

Examples of files can be found here_.
.. _here: https://github.com/anteater/tree/master/examples
