# JSON Protocol specs

[![Documentation Status](https://readthedocs.org/projects/glpi-json-protocol/badge/?version=latest)](https://glpi-json-protocol.readthedocs.io/en/latest/?badge=latest)

Current specs is built on top of [Sphinx documentation generator](http://sphinx-doc.org/). 

Specs are released under the terms of the [MIT license](LICENSE).

## Read it online!

[JSON Protocol specs](http://glpi-json-protocol.rtfd.io/).

## Run it!

You'll have to install [Python Sphinx](http://sphinx-doc.org/) 1.3 minimum.

If your distribution does not provide this version, you could use a `virtualenv`:
```
$ virtualenv /path/to/virtualenv/files
$ source /path/to/virtualenv/bin/activate
$ pip install -r requirements.txt
```

Once all has been successfully installed, just run the following to build the documentation:
```
$ make html
```

Results will be avaiable in the `build/html` directory :)

Note that it actually uses the readthedocs theme installed in your virtual environment and it can differ locally from the one on readthedocs system.

## Autobuild

Autobuild automatically rebuild and refresh the current page on edit.
To use it, you need the `sphinx-autobuild` module:
```
$ pip install sphinx-autobuild
```

You can then use the `sphinx-autobuild` command:
```
$ sphinx-autobuild source build/html
```

And access your documentation on the proposed link: `http://127.0.0.1:8000`
