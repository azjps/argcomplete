argcomplete - Bash tab completion for argparse
==============================================
*Tab complete all the things!*

Argcomplete provides easy, extensible command line tab completion of arguments for your Python script.

It makes two assumptions:

* You're using bash as your shell (limited support for zsh, fish, and tcsh is available)
* You're using `argparse <http://docs.python.org/3/library/argparse.html>`_ to manage your command line arguments/options

Argcomplete is particularly useful if your program has lots of options or subparsers, and if your program can
dynamically suggest completions for your argument/option values (for example, if the user is browsing resources over
the network).

Installation
------------
::

    pip3 install argcomplete
    activate-global-python-argcomplete

See `Activating global completion`_ below for details about the second step (or if it reports an error).

Refresh your bash environment (start a new shell or ``source /etc/profile``).

Synopsis
--------
Python code (e.g. ``my-awesome-script``):

.. code-block:: python

    #!/usr/bin/env python
    # PYTHON_ARGCOMPLETE_OK
    import argcomplete, argparse
    parser = argparse.ArgumentParser()
    ...
    argcomplete.autocomplete(parser)
    args = parser.parse_args()
    ...

Shellcode (only necessary if global completion is not activated - see `Global completion`_ below), to be put in e.g. ``.bashrc``::

    eval "$(register-python-argcomplete my-awesome-script)"

argcomplete.autocomplete(*parser*)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This method is the entry point to the module. It must be called **after** ArgumentParser construction is complete, but
**before** the ``ArgumentParser.parse_args()`` method is called. The method looks for an environment variable that the
completion hook shellcode sets, and if it's there, collects completions, prints them to the output stream (fd 8 by
default), and exits. Otherwise, it returns to the caller immediately.

.. admonition:: Side effects

 Argcomplete gets completions by running your program. It intercepts the execution flow at the moment
 ``argcomplete.autocomplete()`` is called. After sending completions, it exits using ``exit_method`` (``os._exit``
 by default). This means if your program has any side effects that happen before ``argcomplete`` is called, those
 side effects will happen every time the user presses ``<TAB>`` (although anything your program prints to stdout or
 stderr will be suppressed). For this reason it's best to construct the argument parser and call
 ``argcomplete.autocomplete()`` as early as possible in your execution flow.

.. admonition:: Performance

 If the program takes a long time to get to the point where ``argcomplete.autocomplete()`` is called, the tab completion
 process will feel sluggish, and the user may lose confidence in it. So it's also important to minimize the startup time
 of the program up to that point (for example, by deferring initialization or importing of large modules until after
 parsing options).

Specifying completers
---------------------
You can specify custom completion functions for your options and arguments. Two styles are supported: callable and
readline-style. Callable completers are simpler. They are called with the following keyword arguments:

* ``prefix``: The prefix text of the last word before the cursor on the command line.
  For dynamic completers, this can be used to reduce the work required to generate possible completions.
* ``action``: The ``argparse.Action`` instance that this completer was called for.
* ``parser``: The ``argparse.ArgumentParser`` instance that the action was taken by.
* ``parsed_args``: The result of argument parsing so far (the ``argparse.Namespace`` args object normally returned by
  ``ArgumentParser.parse_args()``).

Completers should return their completions as a list of strings. An example completer for names of environment
variables might look like this:

.. code-block:: python

    def EnvironCompleter(**kwargs):
        return os.environ

To specify a completer for an argument or option, set the ``completer`` attribute of its associated action. An easy
way to do this at definition time is:

.. code-block:: python

    from argcomplete.completers import EnvironCompleter

    parser = argparse.ArgumentParser()
    parser.add_argument("--env-var1").completer = EnvironCompleter
    parser.add_argument("--env-var2").completer = EnvironCompleter
    argcomplete.autocomplete(parser)

If you specify the ``choices`` keyword for an argparse option or argument (and don't specify a completer), it will be
used for completions.

A completer that is initialized with a set of all possible choices of values for its action might look like this:

.. code-block:: python

    class ChoicesCompleter(object):
        def __init__(self, choices):
            self.choices = choices

        def __call__(self, **kwargs):
            return self.choices

The following two ways to specify a static set of choices are equivalent for completion purposes:

.. code-block:: python

    from argcomplete.completers import ChoicesCompleter

    parser.add_argument("--protocol", choices=('http', 'https', 'ssh', 'rsync', 'wss'))
    parser.add_argument("--proto").completer=ChoicesCompleter(('http', 'https', 'ssh', 'rsync', 'wss'))

Note that if you use the ``choices=<completions>`` option, argparse will show
all these choices in the ``--help`` output by default. To prevent this, set
``metavar`` (like ``parser.add_argument("--protocol", metavar="PROTOCOL",
choices=('http', 'https', 'ssh', 'rsync', 'wss'))``).

The following `script <https://raw.github.com/kislyuk/argcomplete/master/docs/examples/describe_github_user.py>`_ uses
``parsed_args`` and `Requests <http://python-requests.org/>`_ to query GitHub for publicly known members of an
organization and complete their names, then prints the member description:

.. code-block:: python

    #!/usr/bin/env python
    # PYTHON_ARGCOMPLETE_OK
    import argcomplete, argparse, requests, pprint

    def github_org_members(prefix, parsed_args, **kwargs):
        resource = "https://api.github.com/orgs/{org}/members".format(org=parsed_args.organization)
        return (member['login'] for member in requests.get(resource).json() if member['login'].startswith(prefix))

    parser = argparse.ArgumentParser()
    parser.add_argument("--organization", help="GitHub organization")
    parser.add_argument("--member", help="GitHub member").completer = github_org_members

    argcomplete.autocomplete(parser)
    args = parser.parse_args()

    pprint.pprint(requests.get("https://api.github.com/users/{m}".format(m=args.member)).json())

`Try it <https://raw.github.com/kislyuk/argcomplete/master/docs/examples/describe_github_user.py>`_ like this::

    ./describe_github_user.py --organization heroku --member <TAB>

If you have a useful completer to add to the `completer library
<https://github.com/kislyuk/argcomplete/blob/master/argcomplete/completers.py>`_, send a pull request!

Readline-style completers
~~~~~~~~~~~~~~~~~~~~~~~~~
The readline_ module defines a completer protocol in rlcompleter_. Readline-style completers are also supported by
argcomplete, so you can use the same completer object both in an interactive readline-powered shell and on the bash
command line. For example, you can use the readline-style completer provided by IPython_ to get introspective
completions like you would get in the IPython shell:

.. _readline: http://docs.python.org/3/library/readline.html
.. _rlcompleter: http://docs.python.org/3/library/rlcompleter.html#completer-objects
.. _IPython: http://ipython.org/

.. code-block:: python

    import IPython
    parser.add_argument("--python-name").completer = IPython.core.completer.Completer()

``argcomplete.CompletionFinder.rl_complete`` can also be used to plug in an argparse parser as a readline completer.

Printing warnings in completers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Normal stdout/stderr output is suspended when argcomplete runs. Sometimes, though, when the user presses ``<TAB>``, it's
appropriate to print information about why completions generation failed. To do this, use ``warn``:

.. code-block:: python

    from argcomplete import warn

    def AwesomeWebServiceCompleter(prefix, **kwargs):
        if login_failed:
            warn("Please log in to Awesome Web Service to use autocompletion")
        return completions

Using a custom completion validator
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
By default, argcomplete validates your completions by checking if they start with the prefix given to the completer. You
can override this validation check by supplying the ``validator`` keyword to ``argcomplete.autocomplete()``:

.. code-block:: python

    def my_validator(completion_candidate, current_input):
        """Complete non-prefix substring matches."""
        return current_input in completion_candidate

    argcomplete.autocomplete(parser, validator=my_validator)

Global completion
-----------------
In global completion mode, you don't have to register each argcomplete-capable executable separately. Instead, bash
will look for the string **PYTHON_ARGCOMPLETE_OK** in the first 1024 bytes of any executable that it's running
completion for, and if it's found, follow the rest of the argcomplete protocol as described above.

Additionally, completion is activated for scripts run as ``python <script>`` and ``python -m <module>``.
This also works for alternate Python versions (e.g. ``python3`` and ``pypy``), as long as that version of Python has
argcomplete installed.

.. admonition:: Bash version compatibility

 Global completion requires bash support for ``complete -D``, which was introduced in bash 4.2. On OS X or older Linux
 systems, you will need to update bash to use this feature. Check the version of the running copy of bash with
 ``echo $BASH_VERSION``. On OS X, install bash via `Homebrew <http://brew.sh/>`_ (``brew install bash``), add
 ``/usr/local/bin/bash`` to ``/etc/shells``, and run ``chsh`` to change your shell.

 Global completion is not currently compatible with zsh.

.. note:: If you use setuptools/distribute ``scripts`` or ``entry_points`` directives to package your module,
 argcomplete will follow the wrapper scripts to their destination and look for ``PYTHON_ARGCOMPLETE_OK`` in the
 destination code.

If you choose not to use global completion, or ship a bash completion module that depends on argcomplete, you must
register your script explicitly using ``eval "$(register-python-argcomplete my-awesome-script)"``. Standard bash
completion registration roules apply: namely, the script name is passed directly to ``complete``, meaning it is only tab
completed when invoked exactly as it was registered. In the above example, ``my-awesome-script`` must be on the path,
and the user must be attempting to complete it by that name. The above line alone would **not** allow you to complete
``./my-awesome-script``, or ``/path/to/my-awesome-script``.


Activating global completion
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The script ``activate-global-python-argcomplete`` will try to install the file
``bash_completion.d/python-argcomplete`` (`see on GitHub`_) into an appropriate location on your system
(``/etc/bash_completion.d/`` or ``~/.bash_completion.d/``). If it
fails, but you know the correct location of your bash completion scripts directory, you can specify it with ``--dest``::

    activate-global-python-argcomplete --dest=/path/to/bash_completion.d

Otherwise, you can redirect its shellcode output into a file::

    activate-global-python-argcomplete --dest=- > file

The file's contents should then be sourced in e.g. ``~/.bashrc``.

.. _`see on GitHub`: https://github.com/kislyuk/argcomplete/blob/master/argcomplete/bash_completion.d/python-argcomplete

Zsh Support
------------
To activate completions for zsh you need to have ``bashcompinit`` enabled in zsh::

    autoload -U bashcompinit
    bashcompinit

Afterwards you can enable completion for your scripts with ``register-python-argcomplete``::

    eval "$(register-python-argcomplete my-awesome-script)"

Tcsh Support
------------
To activate completions for tcsh use::

    eval `register-python-argcomplete --shell tcsh my-awesome-script`

The ``python-argcomplete-tcsh`` script provides completions for tcsh.
The following is an example of the tcsh completion syntax for
``my-awesome-script`` emitted by ``register-python-argcomplete``::

    complete my-awesome-script 'p@*@`python-argcomplete-tcsh my-awesome-script`@'

Fish Support
------------
To activate completions for fish use::

    register-python-argcomplete --shell fish my-awesome-script | source

or create new completion file, e.g::

    register-python-argcomplete --shell fish my-awesome-script > ~/.config/fish/completions/my-awesome-script.fish

Completion Description For Fish
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
By default help string is added as completion description.

.. image:: docs/fish_help_string.png

You can disable this feature by removing ``_ARGCOMPLETE_DFS`` variable, e.g::

    register-python-argcomplete --shell fish my-awesome-script | grep -v _ARGCOMPLETE_DFS | source

Absolute Path Completion
~~~~~~~~~~~~~~~~~~~~~~~~
If script is not in path you still can register it's completion by specifying absolute path::

    register-python-argcomplete --shell fish /home/awesome-user/my-awesome-script | source

then you can complete it by using ``/home/awesome-user/my-awesome-script`` or ``./my-awesome-script``.

Unfortunately ``~/my-awesome-script`` would not work, to fix it you can run::

    register-python-argcomplete --shell fish my-awesome-script -e /home/awesome-user/my-awesome-script | source

This would enable completion for any ``my-awesome-script`` in any location.

Git Bash Support
----------------
Due to limitations of file descriptor inheritance on Windows,
Git Bash not supported out of the box. You can opt in to using
temporary files instead of file descriptors for for IPC
by setting the environment variable ``ARGCOMPLETE_USE_TEMPFILES``,
e.g. by adding ``export ARGCOMPLETE_USE_TEMPFILES=1`` to ``~/.bashrc``.

For full support, consider using Bash with the
Windows Subsystem for Linux (WSL).

External argcomplete script
---------------------------
To register an argcomplete script for an arbitrary name, the ``--external-argcomplete-script`` argument of the ``register-python-argcomplete`` script can be used::

    eval "$(register-python-argcomplete --external-argcomplete-script /path/to/script arbitrary-name)"

This allows, for example, to use the auto completion functionality of argcomplete for an application not written in Python. 
The command line interface of this program must be additionally implemented in a Python script with argparse and argcomplete and whenever the application is called the registered external argcomplete script is used for auto completion.

This option can also be used in combination with the other supported shells.

Python Support
--------------
Argcomplete requires Python 3.6+.

Common Problems
---------------
If global completion is not completing your script, bash may have registered a
default completion function::

    $ complete | grep my-awesome-script
    complete -F _minimal my-awesome-script

You can fix this by restarting your shell, or by running
``complete -r my-awesome-script``.

Debugging
---------
Set the ``_ARC_DEBUG`` variable in your shell to enable verbose debug output every time argcomplete runs. This will
disrupt the command line composition state of your terminal, but make it possible to see the internal state of the
completer if it encounters problems.

Acknowledgments
---------------
Inspired and informed by the optcomplete_ module by Martin Blais.

.. _optcomplete: http://pypi.python.org/pypi/optcomplete

Links
-----
* `Project home page (GitHub) <https://github.com/kislyuk/argcomplete>`_
* `Documentation <https://kislyuk.github.io/argcomplete/>`_
* `Package distribution (PyPI) <https://pypi.python.org/pypi/argcomplete>`_
* `Change log <https://github.com/kislyuk/argcomplete/blob/master/Changes.rst>`_
* `xontrib-argcomplete <https://github.com/anki-code/xontrib-argcomplete>`_ - support argcomplete in `xonsh <https://github.com/xonsh/xonsh>`_ shell

Bugs
~~~~
Please report bugs, issues, feature requests, etc. on `GitHub <https://github.com/kislyuk/argcomplete/issues>`_.

License
-------
Licensed under the terms of the `Apache License, Version 2.0 <http://www.apache.org/licenses/LICENSE-2.0>`_.

.. image:: https://github.com/kislyuk/argcomplete/workflows/Python%20package/badge.svg
        :target: https://github.com/kislyuk/argcomplete/actions
.. image:: https://codecov.io/github/kislyuk/argcomplete/coverage.svg?branch=master
        :target: https://codecov.io/github/kislyuk/argcomplete?branch=master
.. image:: https://img.shields.io/pypi/v/argcomplete.svg
        :target: https://pypi.python.org/pypi/argcomplete
.. image:: https://img.shields.io/pypi/l/argcomplete.svg
        :target: https://pypi.python.org/pypi/argcomplete
