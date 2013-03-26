.. _developing:

===========
Development
===========

Heka is an actively developed open-source project being worked on by
the Mozilla Services team. Contributions are welcome.

* Heka project `github page <https://github.com/mozilla-services/heka>`_
* Heka project
  `issue tracker <https://github.com/mozilla-services/heka/issues>`_

* Heka documentation project
  `source code <https://github.com /mozilla-services/heka-docs>`_
* Heka documentation project
  `issue tracker <https://github.com/mozilla-services/heka-docs/issues>`_

Documentation Guidelines
========================

Heka the project encompasses multiple components that may have various
release schedules. Terminology defined in the :ref:`glossary` should be
used for consistency and to avoid ambiguities.

Documentation for the components is organized as follows.

hekad
-----

Documentation intended for:

* Running / Configuring / Deploying hekad
* Contributing to hekad's source code
* Writing Go / Sandbox Plugins

`go doc`
    Documentation strings in the Go source code for hekad. These doc
    strings are extracted and published to document the inner workings
    of hekad for contributers and Go plugin developers.

`sphinx`
    The sphinx project docs found in the hekad repository are intended
    for those running the hekad daemon, configuring it, and writing
    sandbox-based plugins. Man pages and HTML are generated from the
    Sphinx docs.

heka-docs
---------

The documentation you are currently reading, `located in a separate
github repository <https://github.com/mozilla-services/heka-docs>`_.
Content here includes:

* Documentation that applies to the project as a whole
* How heka components fit together
* How to get started with hekad and a client library
* Available client libraries
* Links to the various sub-components

heka-clients
------------

Documentation on how to use a specific client to talk to hekad. These
vary from language to language and should include content on:

* Client Installation
* Configuration of the client
* Examples of using the client with hekad
