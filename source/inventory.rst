.. _INVENTORY:

INVENTORY
=========

INVENTORY protocol can be used by Inventory, NetDiscovery and NetInventory agent tasks, and also by injector, to submit any inventory.

.. hint:: Supported since GLPI 10.0.0

A. Agent request to submit an inventory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The request **MUST** match `GLPI "inventory_format" specs <https://github.com/glpi-project/inventory_format>`_ and :ref:`COMMON <COMMON>`.

.. code::

    {
        "action": "inventory",
        "deviceid": "<device id>",
        "content": {
            "...inventory object following inventory_format..."
        },
        "itemtype": "Computer"
    }

Description:
 * "action": [optional] and could be set to "inventory", "netdiscovery" or "netinventory", defaults is "inventory" when missing
 * "deviceid": [**mandatory**] as defined in `inventory_format <https://github.com/glpi-project/inventory_format>`_
 * "content": [**mandatory**] as defined in `inventory_format <https://github.com/glpi-project/inventory_format>`_
 * "itemtype": [optional] as defined in `inventory_format <https://github.com/glpi-project/inventory_format>`_ but defaults to ``Computer``

B. Server answer to a submitted inventory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

See also :ref:`COMMON <COMMON>` protocol specs.

On successul submission, we expect a simple answer:

.. code::

    {
        "status": "ok",
        "expiration": 24
    }

The server can return an error like:

.. code::

    {
        "status": "error",
        "message": "bad-format",
        "expiration": 24
    }

or

.. code::

    {
        "status": "error",
        "message": "busy server",
        "expiration": 1
    }

On `busy server` error, the agent should keep the inventory and retry its submission at the specified expiration.
