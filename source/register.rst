.. _REGISTER:

REGISTER
========

REGISTER protocol should be used by the GLPI Agent Register task to fully register the agent with a GLPI server.
It could also be used to register the agent on another agent acting as a proxy agent thanks to its Proxy plugin.

.. attention:: This specification is still considered as a **DRAFT** as not implemented in GLPI 10 and GLPI-Agent

Features:
 * agent is identified by its **agentid** in UUID format
 * new configuration parameter on agent side: `token`

    * without it or if it is invalid, only simple registration is possible and in that case the server should be configured to accept that
    * with a valid token, all exchanges with the server after registration will be secured and encrypted, even if not done throught a SSL connection
    * the token format is UUID string as:

        * this format is indeed representing 16 bytes in hexdecimal
        * this can be used as 128 bits bloc like 128 bits key for AES cipher

 * a 128 bits encryption key can be provided during the registration

    * the key will have to be renew after an expiration

 * this protocol could be supported by the agent Proxy plugin:

    * the proxy agent could manage its own tokens, unknown from the server
    * the proxy agent could not know the final token and then just pass messages being unable to know what secrets are exchanged
    * the registration process could also permit server to contact agents behind proxy agents

 * on server side:

    * we need a way to manage tokens

        * token creation
        * token revocation involving all agent provided keys revocation

    * each token can be conditioned on the use of a tag
    * we must be able to revoke any key provided to an agent or to ask for a registration renew
    * agent keys management: cleanup expired keys
    * new agent could first have to be manually validated
    * encryption could be optional: we need an option to disable encryption

Protocol
~~~~~~~~

Few messages could be exchange between agent and server during agent registration.
The base format of messages is JSON and should respect the :ref:`COMMON <COMMON>` protocol specs.

1. First message from the agent
"""""""""""""""""""""""""""""""

Example:

.. code::

    {
        "action": "register",
        "deviceid": "classic-agent-deviceid",
        "port": 62354,
        "name": "GLPI-Agent",
        "version": "1.0",
        "tag": "awesome-tag"
    }

Description:
 * ``action``: [**mandatory**] must be set to "register"
 * ``deviceid``: [**mandatory**] just a friendly string to name an agent
 * ``port``: [**mandatory**] the TCP port on which the agent is joinable or 0 if not joinable
 * ``name``: [**mandatory**] the product name of the agent
 * ``version``: [**mandatory**] the product version of the agent
 * ``tag``: [optional] the setup "tag" in the agent configuration

2. Server answer
""""""""""""""""

Examples:

.. code::

    {
        "status": "registered",
        "expiration": "30d"
    }

.. code::

    {
        "status": "error",
        "message": "forbidden",
        "expiration": "4h"
    }

.. code::

    {
        "status": "pending",
        "needs": "token-validation",
        "expiration": "1m",
        "challenge": "a3540c0e-ac3c-46cf-892f-692ca02209f8"
    }

Description:
 * ``status``: [**mandatory**] the resulting status of the request in ``registered``, ``error`` or ``pending``
 * ``message``: [optional] a message to keep in log as an error reason or for debugging purpose
 * ``expiration``: [**mandatory**] the expiration of the status
 * ``needs``: [optional] a string in ``token-validation``, ``manual-validation``, ``server-validation`` but should be set if ``status`` is ``pending``
 * ``challenge``: [optional] a string in UUID format which is a 128 bits cryptographic challenge *(details in Cryptographic exchanges chapter)*. It must be set when ``status`` is ``pending`` and ``needs`` is ``token-validation``.

About ``expiration``, it has different meanings:
 * if ``status`` is ``registered``, the agent will have to register again before the expiration:

    * by default, it tries to register again after the half of the expiration
    * if it has no answer, it will wait at the middle of the remaining delay
    * the delay should not be lower than the :ref:`CONTACT <CONTACT>` protocol delay

 * if ``status`` is ``error``, the agent should not try to register (and even to communicate) before the given expiration
 * if ``status`` is ``pending``:

    * if ``needs`` is ``token-validation``, this is the expiration of the challenge as the agent should answer the challenge asap
    * if ``needs`` is ``server-validation`` or ``manual-validation``, this is the delay for the next contact with the same request. ``server-validation`` can be returned by a proxy and should not be used by GLPI server.

The agent is not expected to request another message unless ``status`` is ``pending`` and ``needs`` is ``token-validation``. Unless that case, next agent register message is like a new registration, the big difference is the message is encrypted if it is registered and the expiration has not been reached.

3. Agent token validation message
"""""""""""""""""""""""""""""""""

Example:

.. code::

    {
        "action": "register",
        "challenge": "07f2cc8b-194c-45b9-a4e8-68a78129b8e6"
    }

.. code::

    {
        "action": "register",
        "challenge": "failure"
    }

Description:
 * ``action``: [**mandatory**] must be set to ``register``
 * ``challenge``: [**mandatory**] in principle, a string in UUID format which is a 128 bits cryptographic challenge *(details in Cryptographic exchanges chapter)*

    * It must be the answer to the challenge defined by the server
    * It case of error on agent side, can be set to a message like simply ``failure``

4. Server challenge answer
""""""""""""""""""""""""""

Examples:

.. code::

    {
        "status": "registered",
        "expiration": "30d",
        "challenge": "393c263e-1168-44a7-bbdc-6d2ce8514db0",
        "crypto": "680ca885-e017-44a4-81c9-729f759ee3c6"
    }

.. code::

    {
        "status": "pending",
        "expiration": "1m"
    }

.. code::

    {
        "status": "error",
        "message": "challenge failed",
        "expiration": "1h"
    }

Description:
 * ``status``: [**mandatory**] the resulting status of the request in ``registered``, ``error`` or ``pending``

    * ``pending`` is to be used by proxy agents. The agent will have to send again the same challenge at expiration.

 * ``message``: [optional] a message to keep in log as an error reason or for debugging purpose
 * ``expiration``: [**mandatory**] the expiration of the status
 * ``challenge``: [optional] a string in UUID format which is the final 128 bits encrypted server answer challenge *(details in Cryptographic exchanges chapter)*
 * ``crypto``: [optional] a string in UUID format which is a 128 bits encrypted key *(details in Cryptographic exchanges chapter)*. It is optional as encryption may be not required by the server.

Cryptographic exchanges
~~~~~~~~~~~~~~~~~~~~~~~

All cryptographic exchanges are based on AES with 128 bits keys.

1. First challenge from the server
""""""""""""""""""""""""""""""""""

When the server has to create a 128 bits challenge:
 * it uses 8 random bytes (64 bits) as first part, this is the **server secret**
 * it select an agentid: the one from the HTTP header or one from the ``GLPI-Proxy-ID`` HTTP header list. This is to support the case where we are sure we didn't share a token with the agent but we trust a proxy. The tag could be used to trust a proxy.
 * it concatenates the first 8 random bytes with the last 8 bytes of the selected agentid taken as raw 16 bytes
 * it encrypts this 128 bits secret with AES cipher using the token as 128 bits key. The token is the one the server expects the target agent knows.
 * it transforms the secret as UUID string to be included in the JSON answer as ``challenge`` parameter

2. Challenge handling in the agent
""""""""""""""""""""""""""""""""""

When an agent receive a first server answer with a ``challenge``, it has to:
 * transform the UUID challenge into a 128 bits bloc
 * decrypt the bloc with AES cypher using its configured token as 128 bits key to obtain the secret
 * compare the last 64 bits of the secret to its own agentid last 64 bits:

    * if the bits doesn't match:

        * if the agent is not a proxy, this is an error, the agent can send a message with ``failure`` as ``challenge`` parameter. The agent expect a ``status`` set to ``error`` and an ``expiration`` set to a delay before retrying a registration
        * if the agent is a proxy and does the registration on the behalf of another agent, it keeps the challenge to be include in the answer for the next contact of the related agent

            * security notes: if agent and proxy shares the same token, the proxy could see the 64 bits matched the target agentid and then it knows the secret in the first 64 bits. It is then advised to not use the same tokens for agent and proxy. To be safe, each proxy should even have its own and personal token. In that way, the proxy won't be able to know anything about all exchange between the agent and the server

    * if the bits matches:

        * the agent is the target of the challenge
        * the challenge secret is the first 64 bits

 * the agent uses the challenge secret as first 64 bits for the answer challenge
 * it uses 8 random bytes (64 bits) as **agent secret** for the last 64 bits and obtain a 128 bits answer challenge
 * it encrypts this 128 bits secret with AES cipher using the token as 128 bits key. Of course, the token is the one the agent expects the server knows.
 * it transforms the encrypted bloc as UUID string to be included in the JSON answer as ``challenge`` parameter

3. Answer challenge handling in the server
""""""""""""""""""""""""""""""""""""""""""

As an agent proxy knowing the answer is expected by a server:
 * returns a message with ``status`` set to ``pending``
 * transmit the challenge to the server

Otherwise as the final server:
 * transform the UUID answer challenge into a 128 bits bloc
 * decrypt the bloc with AES cypher using the expected token as 128 bits key to obtain the secret
 * compare the first 64 bits of the secret to the expected **server secret** defined in step 1

    * if the bits doesn't match:

        * return an ``error`` message and abort the registration

 * as the bits matches, the last 64 bits will be used as **agent secret**
 * **agent secret** and **server secret** are concatenated in that order into a 128 bits bloc
 * the bloc is encrypted with AES cipher using the token as 128 bits key
 * the encrypted bloc is transformed as UUID string to be included in the final JSON answer as ``challenge`` parameter
 * a private 128 bits keys is generated as 16 random bytes an associated to the agent
 * the private key as 128 bits blocs is encrypted with AES cipher using the token as 128 bits key
 * that encrypted bloc is transformed as UUID string to be included in the final JSON answer as ``crypto`` parameter

4. Final answer challenge handling in the agent
"""""""""""""""""""""""""""""""""""""""""""""""

As an agent proxy knowing the answer is not for itself:
 * the message is saved
 * the saved message is transmitted to the following agent at next contact

Otherwise as the target agent:
 * transform the UUID answer challenge into a 128 bits bloc
 * decrypt the bloc with AES cypher using the token as 128 bits key to obtain the secret
 * compare the first 64 bits of the secret to the expected **agent secret** defined in step 2

    * if the bits doesn't match:

        * send an ``register`` message with ``failure`` as ``challenge``

 * compare the last 64 bits of the secret to the expected **server secret** defined in step 1

    * if the bits doesn't match:

        * send an ``register`` message with ``failure`` as ``challenge``

 * if present, transform the UUID in ``crypto`` into a 128 bits bloc
 * decrypt the bloc with AES cypher using the token as 128 bits key to obtain the communication 128 bits key. This key can now be used for all future communications.

Remarks
~~~~~~~

About port & proxy
""""""""""""""""""

The port should be set to the proxy one on the first proxy transmitted message toward
next server unless the agent or a proxy has its HTTP listener disabled. So if an option
is enabled on proxy, it will also be able to join the agent on the behalf of the server.
This should even work with more than one proxy between server and target agent.
Only asynchronous messages should be handled that way, so each protocol specs should
support asynchronous messaging.
