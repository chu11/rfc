.. github display
   GitHub is NOT the preferred viewer for this file. Please visit
   https://flux-framework.rtfd.io/projects/flux-rfc/en/latest/spec_9.html

9/Distributed Communication and Synchronization Best Practices
==============================================================

.. list-table::
  :widths: 25 75

  * - **Name**
    - github.com/flux-framework/rfc/spec_9.rst
  * - **Editor**
    - Tom Scogland <scogland1@llnl.gov>
  * - **State**
    - raw

Language
--------

.. include:: common/language.rst

Goals
-----

To establish best practices, preferred patterns and anti-patterns for
distributed services in the flux framework. Several of the core services of
Flux, including messages, RPCs, and the KVS, provide services for distributed
notification and synchronization, but not all of them are suitable for all
cases. For now, this is a listing of some common patterns and anti-patterns,
but may shape into a more comprehensive guide as the most effective patterns
are identified.


Anti-Patterns
-------------

-  **Watch chaining:** The KVS provides the capability to watch keys for
   changes in value, receiving a callback or value on change. It is tempting
   to chain these events together, adding a new watch inside the callback from
   an event in order to catch the next action of a remote service. This is
   extremely likely to result in race-conditions unless additional external
   synchronization is employed. Flux services SHOULD NOT employ a pattern of
   chained watches.

-  **KVS replacement updates:** When keys are used as triggers that represent
   streams of events to be read, services SHOULD NOT use replacement on the
   value of each update. Keys for which only the most recent value is useful,
   such as monotonically increasing counters, MAY prefer replacement since their
   historical values can be inferred.


Best-Practices
--------------


General Guidelines
~~~~~~~~~~~~~~~~~~

-  Services SHOULD NOT block the reactor any longer than necessary. Prefer to
   place an asynchronous request or RPC and return to the reactor rather than
   block for a reply, co-routines in message handlers are one way to accomplish
   this.

-  For global notifications, prefer events.

-  For data which should be persisted and produce a notification, prefer to write to
   the KVS in a way that supports watches.


Preferred Patterns
~~~~~~~~~~~~~~~~~~

-  **Centralize update triggers:** Services may notify completion or transition
   events through watches on known points in the KVS. Services that use this
   form of notification SHOULD group the points to be watched and/or retain the
   updates made by appending to rather than replacing values.

-  **KVS request and response:** Communication that needs to be logged and then
   triggered may be written to the KVS, but the service SHOULD provide either a
   single touch-point that it will watch in the KVS for requests or an event or
   request name to trigger the read of the data. If out-of-band triggers are
   used, ``kvs_get_version()`` SHOULD be used to retrieve the KVS version in
   which the data is ready, and ``kvs_wait_version()`` SHOULD be used by the
   client to wait for consistency.
