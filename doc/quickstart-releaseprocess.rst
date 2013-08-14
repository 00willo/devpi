.. include:: links.rst

.. _quickstart-releaseprocess:

Quickstart: uploading, testing, pushing releases 
------------------------------------------------

This quickstart document walks you through setting up a self-contained
pypi release upload, testing and staging system for your Python packages.

Installing devpi client and server
++++++++++++++++++++++++++++++++++

We want to run the full devpi system on our laptop::

	$ pip install --pre -U -q devpi

This will install ``devpi-client`` and ``devpi-server`` pypi packages.

devpi quickstart: initializing basic scenario
+++++++++++++++++++++++++++++++++++++++++++++

The ``devpi quickstart`` command performs basic initialization step
of the devpi system on your local machine:

- start a background devpi-server at ``http://localhost:3141``

- configure the client-side tool ``devpi`` to connect to the newly
  started server

- create and login a user, using as defaults your current login name 
  and an empty password.

- create an index and directly use it.

Let's run the quickstart command which will trigger
a series of other devpi commands::

    $ devpi quickstart
    --> $ devpi-server --start
    starting background devpi-server at http://localhost:3141
    /home/hpk/p/devpi/doc/.devpi/server/.xproc/devpi-server$ /home/hpk/venv/0/bin/devpi-server
    process 'devpi-server' started pid=13370
    devpi-server process startup detected
    logfile is at /home/hpk/p/devpi/doc/.devpi/server/.xproc/devpi-server/xprocess.log
    --> $ devpi use http://localhost:3141
    using server: http://localhost:3141/ (not logged in)
    no current index: type 'devpi use -l' to discover indices
    
    --> $ devpi user -c testuser password=
    user created: testuser
    
    --> $ devpi login testuser --password=
    logged in 'testuser', credentials valid for 10.00 hours
    
    --> $ devpi index -c dev
    dev:
      type=stage
      bases=root/pypi
      volatile=True
      uploadtrigger_jenkins=None
      acl_upload=testuser
    
    --> $ devpi use dev
    using index: http://localhost:3141/testuser/dev/ (logged in as testuser)
    COMPLETED!  you can now work with your 'dev' index
      devpi install PKG   # install a pkg from pypi
      devpi upload        # upload a setup.py based project
      devpi test PKG      # download and test a tox-based project 
      devpi PUSH ...      # to copy releases between indexes
      devpi index ...     # to manipulate/create indexes
      devpi user ...      # to manipulate/create users
      devpi CMD -h        # help for a specific command
      devpi -h            # general help
    docs at http://doc.devpi.net

.. _`quickstart_release_steps`:

devpi install: installing a package
+++++++++++++++++++++++++++++++++++

We can now use the ``devpi`` command line client to trigger a ``pip
install`` of a pypi package (here ``pytest`` as an example) through an
auto-started caching devpi-server::

    $ devpi install pytest
    --> $ docenv/bin/pip install --pre -U -i http://localhost:3141/testuser/dev/+simple/ pytest
    Downloading/unpacking pytest
      Running setup.py egg_info for package pytest
        
    Requirement already up-to-date: py>=1.4.13dev6 in ./docenv/lib/python2.7/site-packages (from pytest)
    Installing collected packages: pytest
      Running setup.py install for pytest
        
        Installing py.test script to /home/hpk/p/devpi/doc/docenv/bin
        Installing py.test-2.7 script to /home/hpk/p/devpi/doc/docenv/bin
    Successfully installed pytest
    Cleaning up...

The ``devpi install`` command configured a pip call, using the
pypi-compatible ``+simple/`` page on our ``testuser/dev`` index for
finding and downloading packages.  The ``pip`` executable was searched
in the ``PATH`` and found in ``docenv/bin/pip``.

Let's check that ``pytest`` was installed correctly::

    $ py.test --version
    This is py.test version 2.3.5, imported from /home/hpk/p/devpi/doc/docenv/local/lib/python2.7/site-packages/pytest.pyc

You may invoke the ``devpi install`` command a second time which will
even work when you have no network.

devpi upload: uploading one or more packages
++++++++++++++++++++++++++++++++++++++++++++

Let's verify we are logged in to the correct index::

    $ devpi use
    using index: http://localhost:3141/testuser/dev/ (logged in as testuser)

Now go to the directory of a ``setup.py`` file of one of your projects  
(we assume it is named ``example``) to build and upload your package
to our ``testuser/dev`` index::

    example $ devpi upload
    created workdir /tmp/devpi0
    --> $ hg st -nmac .
    hg-exported project to <Exported /tmp/devpi0/upload/example>
    --> $ /home/hpk/p/devpi/doc/docenv/bin/python /home/hpk/p/devpi/client/devpi/upload/setuppy.py /tmp/devpi0/upload/example http://localhost:3141/testuser/dev/ testuser testuser-bd4468df5e43c72876b29ad1bc2ad587e08e3575b1ee7de11cdf42cb0bb6834e.BO025g.1W_T1A6K4F0IhHLz_z_zNcD4OkM register -r devpi
    release registered to http://localhost:3141/testuser/dev/
    --> $ /home/hpk/p/devpi/doc/docenv/bin/python /home/hpk/p/devpi/client/devpi/upload/setuppy.py /tmp/devpi0/upload/example http://localhost:3141/testuser/dev/ testuser testuser-bd4468df5e43c72876b29ad1bc2ad587e08e3575b1ee7de11cdf42cb0bb6834e.BO025g.1W_T1A6K4F0IhHLz_z_zNcD4OkM sdist --formats gztar upload -r devpi
    submitted dist/example-1.0.tar.gz to http://localhost:3141/testuser/dev/

There are three triggered actions:

- detection of a mercurial repository, leading to copying all versioned
  files to a temporary work dir.  If you are not using mercurial,
  the copy-step is skipped and the upload operates directly on your source
  tree.

- registering the ``example`` release as defined in ``setup.py`` to 
  our current index

- building and uploading a ``gztar`` formatted release file from the
  workdir to the current index (using a ``setup.py`` invocation under
  the hood).

We can now install the freshly uploaded package::

    $ devpi install example
    --> $ docenv/bin/pip install --pre -U -i http://localhost:3141/testuser/dev/+simple/ example
    Downloading/unpacking example
      Downloading example-1.0.tar.gz
      Running setup.py egg_info for package example
        
    Installing collected packages: example
      Running setup.py install for example
        
    Successfully installed example
    Cleaning up...

This installed your just uploaded package from the ``testuser/dev``
index where we previously uploaded the package.

.. note::

    ``devpi upload`` allows to simultanously upload multiple different 
    formats of your release files such as ``sdist.zip`` or ``bdist_egg``.
    The default is ``sdist.tgz``.



devpi test: testing an uploaded package
+++++++++++++++++++++++++++++++++++++++

If you have a package which uses tox_ for testing you may now invoke::

    $ devpi test example  # package needs to contain tox.ini
    received http://localhost:3141/testuser/dev/example/1.0/example-1.0.tar.gz
    unpacking /tmp/devpi-test0/downloads/example-1.0.tar.gz to /tmp/devpi-test0
    /tmp/devpi-test0/example-1.0$ /home/hpk/venv/0/bin/tox --installpkg /tmp/devpi-test0/downloads/example-1.0.tar.gz -i ALL=http://localhost:3141/testuser/dev/+simple/ --result-json /tmp/devpi-test0/toxreport.json -v
    using tox.ini: /tmp/devpi-test0/example-1.0/tox.ini
    using tox-1.6rc2 from /home/hpk/p/tox/tox/__init__.pyc
    python create: /tmp/devpi-test0/example-1.0/.tox/python
      /tmp/devpi-test0/example-1.0/.tox$ /home/hpk/venv/0/bin/python /home/hpk/venv/0/local/lib/python2.7/site-packages/virtualenv.py --setuptools --python /home/hpk/venv/0/bin/python python >/tmp/devpi-test0/example-1.0/.tox/python/log/python-0.log
    python installdeps: pytest
      /tmp/devpi-test0/example-1.0/.tox/python/log$ /tmp/devpi-test0/example-1.0/.tox/python/bin/pip install -i http://localhost:3141/testuser/dev/+simple/ pytest >/tmp/devpi-test0/example-1.0/.tox/python/log/python-1.log
    python inst: /tmp/devpi-test0/downloads/example-1.0.tar.gz
      /tmp/devpi-test0/example-1.0/.tox/python/log$ /tmp/devpi-test0/example-1.0/.tox/python/bin/pip install -i http://localhost:3141/testuser/dev/+simple/ /tmp/devpi-test0/downloads/example-1.0.tar.gz >/tmp/devpi-test0/example-1.0/.tox/python/log/python-2.log
    python runtests: commands[0] | py.test
      /tmp/devpi-test0/example-1.0$ /tmp/devpi-test0/example-1.0/.tox/python/bin/py.test >/tmp/devpi-test0/example-1.0/.tox/python/log/python-3.log
    ___________________________________ summary ____________________________________
      python: commands succeeded
      congratulations :)
    wrote json report at: /tmp/devpi-test0/toxreport.json
    posting tox result data to http://localhost:3141/+tests
    successfully posted tox result data

Here is what happened:

- devpi got the latest available version of ``example`` from the current
  index

- it unpacked it to a temp dir, found the ``tox.ini`` and then invoked
  tox, pointing it to our ``example-1.0.tar.gz``, forcing all installations
  to go through our current ``testuser/dev/+simple/`` index and instructing
  it to create a ``json`` report.

- after all tests ran, we send the ``toxreport.json`` to the devpi server
  where it will be attached precisely to our release file.
 
We can verify that the test status was recorded via::

    $ devpi list example
    list result: http://localhost:3141/testuser/dev/
    testuser/dev/example/1.0/example-1.0.tar.gz
      teta linux2 python 2.7.3 tests passed

devpi push: staging a release to another index
++++++++++++++++++++++++++++++++++++++++++++++

Once you are happy with a release file you can push it either
to another devpi-managed index or to an outside pypi index server.

Let's create another ``staging`` index::

    $ devpi index -c staging volatile=False
    staging:
      type=stage
      bases=root/pypi
      volatile=False
      uploadtrigger_jenkins=None
      acl_upload=testuser

We created a non-volatile index which means that one can not 
overwrite or delete release files. See :ref:`non_volatile_indexes` for more info
on this setting.

We can now push the ``example-1.0.tar.gz`` from above to
our ``staging`` index::

    $ devpi push example-1.0 testuser/staging
    200 register example 1.0 -> testuser/staging
    200 store_releasefile example-1.0.tar.gz -> testuser/staging

This will determine all files on our ``testuser/dev`` index belonging to
the specified ``example-1.0`` release and copy them to the
``testuser/staging`` index. 

devpi push: releasing to an external index
++++++++++++++++++++++++++++++++++++++++++

Let's check again our our current index::

    $ devpi use
    using index: http://localhost:3141/testuser/dev/ (logged in as testuser)

Let's now use our ``testuser/staging`` index::

    $ devpi use testuser/staging
    using index: http://localhost:3141/testuser/staging/ (logged in as testuser)

and check the test result status again::

    $ devpi list example
    list result: http://localhost:3141/testuser/staging/
    testuser/staging/example/1.0/example-1.0.tar.gz
      teta linux2 python 2.7.3 tests passed

Good, the test result status is still available after the push
from the last step.

We may now decide to push this release to an external
pypi-style index which we have configured in the ``.pypirc`` file::

    $ devpi push example-1.0 pypi:testrun
    using pypirc /home/hpk/.pypirc
    200 register example 1.0
    200 upload testuser/staging/example/1.0/example-1.0.tar.gz

this will push all release files of the ``example-1.0`` release
to the external ``testrun`` index server, using credentials
and the URL found in the ``pypi`` section in your
``.pypirc``.

index inheritance re-configuration
++++++++++++++++++++++++++++++++++

At this point we have the ``example-1.0`` release and release file
on both the ``testuser/dev`` and ``testuser/staging`` indices.
If we rather want to always use staging packages in our development
index, we can reconfigure the inheritance 
``bases`` for ``testuser/dev``::

    $ devpi index testuser/dev bases=testuser/staging
    testuser/dev changing bases: testuser/staging
    testuser/dev:
      type=stage
      bases=testuser/staging
      volatile=True
      uploadtrigger_jenkins=None
      acl_upload=testuser

If we now switch back to using ``testuser/dev``::

    $ devpi use testuser/dev
    using index: http://localhost:3141/testuser/dev/ (logged in as testuser)

and look at our example release files::

    $ devpi list example
    list result: http://localhost:3141/testuser/dev/
    testuser/dev/example/1.0/example-1.0.tar.gz
      teta linux2 python 2.7.3 tests passed
    testuser/staging/example/1.0/example-1.0.tar.gz
      teta linux2 python 2.7.3 tests passed

we'll see that ``example-1.0.tar.gz`` is contained in both
indices.  Let's remove the ``testuser/dev`` ``example`` release::

    $ devpi remove -y example
    About to remove the following release files and metadata:
       testuser/dev/example/1.0/example-1.0.tar.gz
    Are you sure (yes/no)? yes (autoset from -y option)

If you don't specify the ``-y`` option you will be asked to confirm
the delete operation interactively.

The ``example-1.0`` release remains accessible through ``testuser/dev``
because it inherits all releases from its ``testuser/staging`` base::

    $ devpi list example
    list result: http://localhost:3141/testuser/dev/
    testuser/staging/example/1.0/example-1.0.tar.gz
      teta linux2 python 2.7.3 tests passed


running devpi-server permanently
+++++++++++++++++++++++++++++++++

If you want to configure a permanent devpi-server install,
you can go to :ref:`quickstart-server` to learn more.

