.. _COMMON:

COMMON
======

Common specs to all task protocol specs with some specific explanations on some goal.

.. hint:: Supported since GLPI 10.0.0

Transport protocol
~~~~~~~~~~~~~~~~~~

Messages are passed between client and server via HTTP or HTTPS protocol.

HTTP headers
~~~~~~~~~~~~

``GLPI-Agent-ID``
-----------------

An agent identity **MUST** be set in HTTP headers:
 * This is the **agentid**
 * **agentid** HTTP header: ``GLPI-Agent-ID``
 * **agentid** format: plain text UUID which can be reduced in a 128 bits raw id by code
 * example: ``3a609a2e-947f-4e6a-9af9-32c024ac3944``
 * it is generated by the agent the first time it starts
 * server must also set it in answer to specify agent destination, this can be used by a proxy agent to route server answer

``GLPI-Request-ID``
-------------------

A request id **CAN** be set in HTTP headers:
 * HTTP header: ``GLPI-Request-ID``
 * format: a 8 digit hexadecimal string in higher case like ``42E6A9AF1``
 * When present, it **MUST** be set back in answer
 * When present, it **SHOULD** be logged
 * It **MUST** be set by a proxy in an answer when the answer is still not definitively known
 * It **MUST** be set when using a GET request to query the final request status from a proxy. It **MUST** then be identical the the one provided by the previous proxy answer.

``Content-Type``
----------------

The content type **MUST** be set in HTTP headers and follow the official public specs:
 * HTTP header: ``Content-Type``
 * The following content type are to be supported:

   * ``application/json``: **MANDATORY SUPPORT** This format must be supported as this will be the default format
   * ``application/x-compress-zlib``: Compressed JSON with zlib compression
   * ``application/x-compress-gzip``: Compressed JSON with gzip compression
   * ``application/x-compress-br``: Compressed JSON with brotli compression
   * ``application/xml``: Only for [CONTACT](specs/CONTACT) protocol

``Accept``
----------

A party **CAN** announce which content types it supports setting ``Accept`` HTTP header
 * HTTP header: ``Accept``
 * format: must follow `RFC7231 <https://tools.ietf.org/html/rfc7231#page-38>`_
 * can be used to try another content type on "unsupported content-type" error
 * Example:

   * ``Accept: application/json``
   * ``Accept: application/xml; q=0.1, application/json; q=0.5, application/x-compress-zlib; q=0.7, application/x-compress-gzip; q=0.8, application/x-compress-br``

``Pragma``
----------

The agent requests **MUST** includes the pragma header to avoid any caching done by the server:
 * HTTP header and value: ``Pragma: no-cache``

``GLPI-CryptoKey-ID``
---------------------

If an encryption key has been shared and encryption is enabled, a side party must announce the key owner used to encrypt data: this is generally **agentid**
 * HTTP header: ``GLPI-CryptoKey-ID``
 * format: same as agentid
 * the same encryption should be used when answering unless encryption is disabled on a side
 * in a proxy context, this id can be different than ``GLPI-Agent-ID``
 * when this header is missing, data are not expected to be encrypted

``GLPI-Proxy-ID``
-----------------

When a GLPI-Agent is used as a proxy, it **MUST** set/update this HTTP header with its own agentid.
 * HTTP header: ``GLPI-Proxy-ID``
 * format: list of agentid separated by commas
 * a proxy agent must check its agentid is still not in the list, otherwise it **MUST** answer a HTTP 404 error with a JSON error message (see below): "proxy-loop-detected"
 * a proxy agent parameter should define the maximum number of proxy to be set in the list. In the case, the header reaches the maximum on a proxy, it **MUST** answer a HTTP 404 error with a JSON error message (see below): "too-many-proxy". By default, the maximum number of proxy should be set to 5 as that's still large and most usercase will only use 1 proxy.
 * GLPI server should show the proxy list in the agent management panel.

JSON template messages
~~~~~~~~~~~~~~~~~~~~~~

Requests
--------

.. code::

    {
      "action": "<token>"
    }

Description:
 * "action": [optional] but **should** be set to a supported token otherwise action defaults to `inventory` and then **MUST** respect the :ref:`INVENTORY <INVENTORY>` protocol specs.

Answers
-------

.. code::

    {
      "status": "<token>",
      "message": "<string>",
      "expiration": "<delay definition>"
    }

Description:
 * "status": [**mandatory**] the resulting status of the request related to the action. Must be a supported token, see related protocol.
  * Common tokens: "pending", "error", "ok"
  * The "pending" token with a short "expiration" delay could be used by proxy agents to tell agents to retry the same request soon as the proxy has the time to obtain the server expected answer.

 * "message": [optional] a message to keep in log as an error reason or for debugging purpose
 * "expiration": [optional] the delay before the agent can send another message for the current action. The delay may have different meanings regarding the related action. The delay definition must be a positive integer number immediately followed by an optional letter defining the number unit. The letter should be one of "s", "m", "h", "d" which are respectively representing seconds, minutes, hours and days. The default is "h" when no unit follows the number.

  * if not set, no delay is required and next request can be handled immediately
  * in case of error or no answer, the latest delay or configured one should be used by the client
  * example: "1d", "24", "1m", "30s", "6h"

Error answers
-------------

.. code::

    {
      "status": "error",
      "message": "unsupported content-type"
    }

Description:
 * On error, the HTTP answer error code **MUST** be set with the 4XX or 5XX HTTP related error
 * For 4XX errors, status code in free, but the answer **SHOULD** be a short JSON message with the "error" value set as "status" property and eventually a "message" property to help the other party to analyze the error
