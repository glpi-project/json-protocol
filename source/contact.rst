.. _CONTACT:

CONTACT
=======

CONTACT protocol is needed to permit GLPI Agent to detect it communicates with a supported server (GLPI, FusionInventory plugin for GLPI).

.. hint:: Supported since GLPI 10.0.0

Features:
 * Keep compatibility with FusionInventory as fallback
 * Cover all the same features then FusionInventory protocol
 * Add more features:

   * agent tells server which tasks are installed
   * agent tells server which tasks are enabled
   * server tells which tasks has to be disabled
   * agent tells its configured tag so it can have a meaning for any task

A. Agent CONTACT request
~~~~~~~~~~~~~~~~~~~~~~~~

The agent **MUST** set new HTTP headers accordingly to the :ref:`COMMON <COMMON>` protocol specs.

*2 cases*: the agent knows or not server supported content types. A new agent option should permit to force this "knowledge".

A1. Agent does NOT know server supported content-types
""""""""""""""""""""""""""""""""""""""""""""""""""""""

.. _PROLOG:

Based on what is still doing FusionInventory agent:
 * the agent can send a HTTP request named **PROLOG** with such following content:

.. code::

    <REQUEST>
      <QUERY>PROLOG</QUERY>
      <TOKEN>12345678</TOKEN>
      <DEVICEID>foo-agent-deviceid</DEVICEID>
    </REQUEST>

About HTTP headers in :ref:`PROLOG <PROLOG>` request:
 * `Content-Type` is restricted to the following values, depending on supported compression scheme as they are the only ones supported by FusionInventory for GLPI plugin:

    * `application/xml`
    * `application/x-compress-zlib`
    * `application/x-compress-gzip`

A2. Agent think it knows server supported content-types
"""""""""""""""""""""""""""""""""""""""""""""""""""""""

This can indeed be an assumption based on previously exchanged messages.
 * the agent sends a HTTP request named **CONTACT** with the following JSON possible content:

.. code::

    {
      "action": "contact",
      "deviceid": "classic-agent-deviceid",
      "name": "GLPI-Agent",
      "version": "1.0",
      "installed-tasks": [
        "inventory",
        "register",
        "..."
      ],
      "enabled-tasks": [
        "collect",
        "deploy",
        "..."
      ],
      "httpd-port": "62354",
      "httpd-plugins": {
        "ssl": {
          "disabled": "no",
          "ports": [
            "0",
            "62356"
          ],
          "ssl_cert_file": "cert.pem",
          "ssl_key_file": "key.pem"
        },
        "proxy": "disabled",
        "toolbox": "disabled",
        "..."
      },
      "tag": "awesome-tag"
    }

Description:
 * ``action``: [**mandatory**] must be set to "contact"
 * ``deviceid``: [**mandatory**] just a friendly string to name an agent
 * ``name``: [**mandatory**] the product name of the agent
 * ``version``: [**mandatory**] the product version of the agent
 * ``installed-tasks``: [**mandatory**] a list of agent installed tasks
 * ``enabled-tasks``: [optional] a list of enabled task if different than "installed-tasks"
 * ``httpd-port``: [optional] the httpd port used by the agent if it listens on
 * ``httpd-plugins``: [optional] a list of agent httpd plugins with "disabled" status or its configuration if enabled
 * ``tag``: [optional] the "tag" setup in the agent configuration

B. Server CONTACT answer
~~~~~~~~~~~~~~~~~~~~~~~~

As the server detects ``GLPI-Agent-ID`` as HTTP header, it SHOULD always answer using the new protocol. The answer could have the following content:

.. code::

    {
      "status": "<token>",
      "message": "<optional string>",
      "tasks": {
        "inventory": {
          "no-category": "processes",
          "server": "glpi",
          "version": "10.0.0"
        },
        "networkinventory": {
          "server": "glpiinventory",
          "version": "1.0"
        },
        "deploy": {
          "server": "glpiinventory",
          "version": "1.0"
        }
      },
      "disabled": [
        "collect",
        "wakeonlan",
        "remoteinventory"
      ],
      "jobs": {
        "networkinventory": [
          {
            "task": "networkinventory",
            "jobid": "this job id",
            "credentials": [ "1", "2" ]
          }
        ],
        "deploy": [
          {
            "task": "deploy",
            "jobid": "this job id"
          }
        ]
      },
      "credentials": {
        "1": {
          "community": "public",
          "type": "snmp",
          "version": "v1"
        },
        "2": {
          "community": "public",
          "type": "snmp",
          "version": "v2c"
        }
      },
      "expiration": "1d"
    }

Description:
 * ``status``: [**mandatory**] the resulting status of the request in "error", "pending", "ok"
 * ``expiration``: [**mandatory**] the delay before asking for another CONTACT. It has the same purpose than PROLOG_FREQ but includes a unit, example "1d" for one day.
 * ``tasks``: [optional] a list of task configuration objects reduced as JSON structure

   * should be used to pass params to listed tasks
   * as example, "url" can be set to set another URL to request during the task processing
   * the list can be empty

 * ``disabled``: [optional] a list of tasks disabled on server side so it won't be run by the agent and the agent will log the task is disabled
 * ``jobs``: [optional] a dictionary with task names as key and list of jobs as values. Each jobs list is an ordered list of job objects in the task related format related as JSON structure.
 * ``credentials``: [optional] a dictionary of needed credentials. Keys are positive integer numbers and should be referenced in a job object. Values are credentials objects as JSON structure.
 * ``message``: [optional] message to be logged on agent side with error or at debug level

Error handling
~~~~~~~~~~~~~~

In case of error on server side, "status" should be set to "error" in the answer and a meaningful and short string should be set as "message". The returned HTTP code should be set to 4XX. See :ref:`COMMON <COMMON>` for error message specs.

Example:
.. code::

    {
      "status": "error",
      "message": "malformed json",
      "expiration": "1d"
    }

.. code::

    {
      "status": "error",
      "message": "forbidden",
      "expiration": "1d"
    }

.. code::

    {
      "status": "error",
      "message": "unsupported content-type",
      "expiration": 24
    }

.. code::

    {
      "status": "error",
      "message": "busy server",
      "expiration": "30m"
    }
