.. _server:

======
Server
======

If you are implementing an OpenADR Server ("Virtual Top Node") using OpenLEADR, read this page.

.. _server_registration:

Registration
============

If a client (VEN) wants to register for the first time, it will go through a Registration procedure.

.. admonition:: Implementation Checklist

    1. Create a handler that decides what to do with new registrations, based on their registration info.


The client will send a :ref:`oadrQueryRegistration` message. The server will respond with a :ref:`oadrCreatedPartyRegistration` message containing a list of its capabilities, notably the implemented OpenADR protocol versions and the available Transport Mechanisms (HTTP and/or XMPP).

The client will then usually send a :ref:`oadrCreatePartyRegistration` message, in which it registers to a specific OpenADR version and Transport Method. The server must then decide what it wants to do with this registration.

In the case that the registration is accepted, the VTN will generate a venID and a RegistrationID for this VEN and respond with a :ref:`oadrCreatedPartyRegistration` message.

In your application, when a VEN sends a :ref:`oadrCreatePartyRegistration` request, it will call your ``on_create_party_registration`` handler. This handler must somehow look up what to do with this request, and respond with a ``ven_id, registration_id`` tuple.

Example implementation:

.. code-block:: python3

    from openleadr.utils import generate_id

    async def on_create_party_registration(payload):
        ven_name = payload['ven_name']
        # Check whether or not this VEN is allowed to register
        result = await database.query("""SELECT COUNT(*)
                                           FROM vens
                                          WHERE ven_name = ?""",
                                      (payload['ven_name'],))
        if result == 1:
            # Generate an ID for this registration
            ven_id = generate_id()
            registration_id = generate_id()

            # Store the registration in a database (pseudo-code)
            await database.query("""UPDATE vens
                                       SET ven_id = ?
                                       registration_id = ?
                                     WHERE ven_name = ?""",
                                 (ven_id, registration_id, ven_name))

            # Return the registration ID.
            # This will be put into the correct form by the OpenADRServer.
            return ven_id, registration_id

.. _server_events:

Events
======

The server (VTN) is expected to know when it needs to inform the clients (VENs) of certain events that they must respond to. This could be a predicted shortage or overage of available power in a certain electricity grid area, for example.

The easiest way to supply events to a VEN is by using OpenLEADR's built-in message queing system. You simply add an event for a ven using the ``server.add_event`` method. You supply the ven_id for which the event is required, as well as the ``signal_name``, ``signal_type``, ``intervals`` and ``targets``. This will build an event object with a single signal for a VEN. If you need more flexibility, you can alternatively construct the event dictionary yourself and supply it directly to the ``add_raw_event`` method.

The VEN can decide whether to opt in or opt out of the event. To be notified of their opt status, you supply a callback handler which will be called when the VEN has responded to the event request.

.. code-block:: python3

    from openleadr import OpenADRServer
    from functools import partial
    from datetime import datetime, timezzone

    async def event_callback(ven_id, event_id, opt_status):
        print(f"VEN {ven_id} responded {opt_status} to event {event_id}")

    server = OpenADRServer(vtn_id='myvtn')
    server.add_event(ven_id='ven123',
                     event_id='event123',
                     signal_name='simple',
                     signal_type='level',
                     intervals=[{'dtstart': datetime(2020. 1, 1, 12, 0, 0, tzinfo=timezone.utc),
                                 'signal_payload': 1},
                                 {'dtstart': datetime(2020. 1, 1, 12, 15, 0, tzinfo=timezone.utc),
                                 'signal_payload': 0}],
                     target=[{'resource_id': 'Device001'}],
                     callback=partial(event_callback, ven_id='ven123', event_id='event123'))


Alternatively, you can use the handy constructors in ``openleadr.objects`` to format parts of the event:

.. code-block:: python3

    from openleadr import OpenADRServer
    from openleadr.objects import Target, Interval
    from datetime import datetime, timezone
    from functools import partial

    server = OpenADRServer(vtn_id='myvtn')
    server.add_event(ven_id='ven123',
                     event_id='event123',
                     signal_name='simple',
                     signal_type='level',
                     intervals=[Interval(dtstart=datetime(2020, 1, 1, 12, 15, 0, tzinfo=timezone.utc),
                                         signal_payload=0),
                                Interval(dtstart=datetime(2020, 1, 1, 12, 15, 0, tzinfo=timezone.utc),
                                         signal_payload=1)]
                     target=[Target(resource_id='Device001')],
                     callback=partial(event_callback, ven_id='ven123', event_id='event123'))


.. _server_reports:

Reports
=======

Please see the :ref:`reporting` section.


.. _server_implement:

Things you should implement
===========================

You should implement the following handlers:

- ``on_create_party_registration(registration_info)``
- ``on_register_report(ven_id, resource_id, measurement, unit, scale, min_sampling_interval, max_sampling_interval)``

Optionally:

- ``on_poll(ven_id)``; only if you don't want to use the internal message queue.

.. _server_signing_messages:

Signing Messages
================

The OpenLEADR can sign your messages and validate incoming messages. For some background, see the :ref:`message_signing`.

Example implementation:

.. code-block:: python3

    from openleadr import OpenADRServr

    def fingerprint_lookup(ven_id):
        # Look up the certificate fingerprint that is associated with this VEN.
        fingerprint = database.lookup('certificate_fingerprint').where(ven_id=ven_id) # Pseudo code
        return fingerprint

    server = OpenADRServer(vtn_id='MyVTN',
                           cert='/path/to/cert.pem',
                           key='/path/to/private/key.pem',
                           passphrase='mypassphrase',
                           fingerprint_lookup=fingerprint_lookup)

The VEN's fingerprint should be obtained from the VEN outside of OpenADR.


.. _server_message_handlers:

Message Handlers
================

Your server has to deal with the different OpenADR messages. The way this works is that OpenLEADR will expose certain modules at the appropriate endpoints (like /oadrPoll and /EiRegister), and figure out what type of message is being sent. It will then call your handler with the contents of the message that are relevant for you to handle. This section provides an overview with examples for the different kinds of messages that you can expect and what should be returned.

.. _server_on_register_report:

on_register_report
------------------

The VEN informs you which reports it has available. If you want to periodically receive any of these reports, you should return a list of the r_ids that you want to receive.

Signature:

.. code-block:: python3

    async def on_register_report(ven_id, resource_id, measurement, unit, scale,
                                 min_sampling_interval, max_sampling_interval):
        # If we want this report:
        return (callback, requested_sampling_interval)
        # or
        return None

.. _server_on_query_registration:

on_query_registration
---------------------

A prospective VEN is requesting information about your VTN, like the versions and transports you support. You should not implement this handler and let OpenLEADR handle this response.

.. _server_on_create_party_registration:

on_create_party_registration
----------------------------

The VEN tries to register with you. You will receive a registration_info dict that contains, among other things, a field `ven_name` which is how the VEN identifies itself. If the VEN is accepted, you return a ``ven_id, registration_id`` tuple. If not, return ``False``:

.. code-block:: python3

    async def on_create_party_registration(registration_info):
        ven_name = registration_info['ven_name']
        ...
        if ven_is_known:
            return ven_id, registration_id
        else
            return None

During this step, the VEN probably does not have a ``venID`` yet. If they connected using a secure TLS connection, the ``registration_info`` dict will contain the fingerprint of the public key that was used for this connection (``registration_info['fingerprint']``). Your ``on_create_party_registration`` handler should check this fingerprint value against a value that you received offline, to be sure that the ven with this venName is the correct VEN.

.. _server_on_cancel_party_registration:

on_cancel_party_registration
----------------------------

The VEN informs you that they are cancelling their registration and no longer wish to be contacted by you.

You should deregister the VEN internally, and return `None`.

Return: ``None``


.. _server_on_poll:

on_poll
-------

You only need to implement this if you don't want to use the automatic internal message queue. If you add this handler to the server, the internal message queue will be automatically disabled.

The VEN is requesting the next message that you have for it. You should return a tuple of message_type and message_payload as a dict. If there is no message for the VEN, you should return `None`.

Signature:

.. code-block:: python3

    async def on_poll(ven_id):
        ...
        return message_type, message_payload

If you implement your own on_poll handler, you should also include your own ``on_created_event`` handler that retrieves the opt status for a distributed event.

.. _server_on_created_event:

on_created_event
----------------

You only need to implement this if you don't want to use the automatic internal message queue. Otherwise, you supply a per-event callback function when you add the event to the internal queue.

Signature:

.. code-block:: python3

    async def on_created_event(ven_id, event_id, opt_status):
        print("Ven {ven_id} returned {opt_status} for event {event_id}")
        # return None
