Introduction
============

GLPI 10+ and GLPI-Agent can communicate using a JSON protocol described here.

GLPI agent is a tool with few important constraints to know when thinking about protocol:

 * by design, it schedules by itself to run its installed `tasks` at an expected time

This is why we will talking about protocol by task: **one task => one protocol spec**

By the way, the first protocol spec is not really an agent task but is inherited from FusionInventory agent history.
It's related to a message named :ref:`PROLOG <PROLOG>`, but for GLPI Agent we will name it :ref:`CONTACT <CONTACT>` and it specifies mostly the first contact a GLPI Agent will have to do with its configured server, even when not knowing what kind of server it is.

GLPI Agent will be better designed to communicate directly with a GLPI server. With that goal in mind, we need to first specify few features :ref:`COMMON <COMMON>` to all task protocol specs.

Future evolution
================

Few evolutions are expected to this JSON protocol as we wanted to enhance communication between GLPI and agents:

 * Agent will be able to use :ref:`REGISTER <REGISTER>` to secure protocol exchanges
 * Agent will be able to use :ref:`CONFIGURATION <CONFIGURATION>` to get its configuration from server
 * As agent supported tasks will be supported in GLPI 10 Core, they will start using:

    * :ref:`NETDISCOVERY <NETDISCOVERY>` protocol specs
    * :ref:`NETINVENTORY <NETINVENTORY>` protocol specs
    * :ref:`COLLECT <COLLECT>` protocol specs
    * :ref:`DEPLOY <DEPLOY>` protocol specs
    * :ref:`ESX <ESX>` protocol specs
    * :ref:`WAKEONLAN <WAKEONLAN>` protocol specs

 * It should also be possible to setup remoteinventory tasks with :ref:`REMOTEINVENTORY <REMOTEINVENTORY>`


