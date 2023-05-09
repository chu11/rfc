.. github display
   GitHub is NOT the preferred viewer for this file. Please visit
   https://flux-framework.rtfd.io/projects/flux-rfc/en/latest/spec_12.html

12/Flux Security Architecture
=============================

This document describes the mechanisms used to secure Flux instances
against unauthorized access and prevent privilege escalation and other
attacks, while ensuring programs run with appropriate user credentials
and are contained within their set of allocated resources.

-  Name: github.com/flux-framework/rfc/spec_12.rst

-  Editor: Jim Garlick <garlick@llnl.gov>

-  State: raw


Language
--------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to
be interpreted as described in `RFC 2119 <https://tools.ietf.org/html/rfc2119>`__.


Related Standards
-----------------

-  :doc:`3/Flux Message Protocol <spec_3>`


Goals
-----

-  Design for auditability.

-  Minimize code running with elevated privilege.

-  Security algorithms should be configurable according to site policy.

-  Run programs with the credentials of the submitting user.

-  Prevent unauthorized access.

-  Assume networks are NOT physically secure by default.

-  Programs are contained within allocated resources.

-  Integration with Linux distribution security services

-  Integration with site security services


Overview
--------

Flux is distributed software that runs parallel programs on behalf of
users in a multi-user Linux environment, including but not limited to
commodity HPC Linux clusters. Flux is unique among resource managers
in that a Flux instance may be launched as a parallel program by an
unprivileged user.

A Flux instance is built upon message brokers communicating via overlay
network(s). A Flux instance has an "instance owner" whose identity is
used to launch the broker processes, and whose credentials secure overlay
network connections, for privacy and message integrity.

The instance owner has full control over the instance, including
its resources (within bounds enforced by the enclosing instance),
its scheduling policies, its loadable extension modules, the access
rights of other users within the instance, even the non-privileged
executable components of the instance.

Neither the instance owner nor the broker executables require special
system privileges. The privilege necessary to launch work as other users
and establish containment to the allocated resources is concentrated in
a single privileged command. Only work cryptographically shown to have
been requested by a user AND authorized by the instance owner to use a
set of resources that it owns will be launched by the privileged command
in a container with those resources.

This concentration of privilege combined with simple rules makes Flux
security auditable, and *safely* gives the instance owner flexibility
to customize and augment Flux services.

A user interacts with a Flux instance (for example via a job submission
command) by connecting to a broker, then sending and receiving messages
as described in RFC 3. Connections are established using broker plugins
called "connectors". The connector is responsible for authenticating
the user, assigning the user one or more "roles", and accepting
or denying the connection.

If a connection is accepted, the userid and role set are saved with
connection state in the connector and subsequent messages originating
from the connection are stamped with this information. The instance
owner controls the configuration for assignment of roles, thus controls
what other users will be allowed do within the instance.

Services that have arranged to receive requests by users other than the
instance owner can gate access to operations using the userid and role
stamped on the request message by the connector, according to the service’s
security policy. For example, a scheduler could allow a user to dequeue
their own jobs, or if they have the "admin" role, to dequeue jobs belonging
to others.

A service’s security policy resides within the service, and is initialized
to a default state by the service. The default security policy for some
services may be altered by instance owner. For example, the instance owner
could extend job cancellation to anyone with the "user" role.


Implementation
--------------

Flux Credentials
~~~~~~~~~~~~~~~~

Flux credentials SHALL consist of a 32-bit *userid* and a 32-bit *rolemask*.
A users's Flux user ID SHALL be the same as the user's POSIX UID.

FLUX_USERID_UNKNOWN (4294967295) SHALL be a reserved userid to indicate
"invalid user".

The Flux rolemask MAY be assigned the following roles:

FLUX_ROLE_NONE (0) SHALL indicate "invalid rolemask".

FLUX_ROLE_OWNER (1) SHALL confer the maximum privilege upon the user,
and is REQUIRED to be assigned to the instance owner.

FLUX_ROLE_USER (2) SHALL confer access, but no administrative privilege
upon the user.

FLUX_ROLE_LOCAL (4) SHALL be set on messages that were sent from the local
broker rank.  This role SHALL be cleared when the message is received by
another broker.

Other role bit definitions are TBD.


Configuration
~~~~~~~~~~~~~

The security policy of a Flux instance SHALL be configurable.

An instance SHALL restrict access to the instance owner, unless explicitly
configured to allow guest users.

Additional configuration MAY include:

- Allowed types of user authentication

- Configuration for each allowed authentication method

- Allow/deny list gating access to specific users

- Assignment of special privileges to lists of users


Connector Security
~~~~~~~~~~~~~~~~~~

Flux connectors SHALL authenticate each connection, mapping it to a valid
Flux userid and rolemask, or rejecting it.

As indicated in RFC 3, Flux messages have a userid and rolemask field.
In messages received en route to the broker, the connector SHALL rewrite
these fields from the expected values of FLUX_USERID_UNKNOWN and FLUX_ROLE_NONE
to the authenticated userid and rolemask.

If the user is not authenticated with FLUX_ROLE_OWNER, and a message is
received en route to the broker with the userid and rolemask NOT set to
the expected values, the message SHALL be rejected: if it is a request,
a POSIX EPERM (1) error response SHALL be returned to the sender; otherwise
the message SHALL be dropped.

If the user is authenticated with FLUX_ROLE_OWNER, valid userid and rolemask
fields SHALL NOT be rewritten. This facilitates testing, and allows
connectors implemented as processes or threads running as the instance owner
to authenticate messages, while themselves connecting to the broker via
authenticated connector.

Connectors that support connections spanning physical networks SHALL protect
against eavesdropping, man-in-the-middle, and other attacks on public
networks.


Service Security Policy
~~~~~~~~~~~~~~~~~~~~~~~

Flux services that implement message handlers SHALL implement security
policy based on the userid and/or rolemask fields in inbound messages.

A policy mechanism SHALL be provided by the Flux reactor for each message
handler that compares the rolemask of inbound messages against an "allow"
rolemask. If a logical and of the two rolemasks produces zero, the message
is rejected: requests SHALL receive a POSIX EPERM (1) error response; other
message types SHALL be dropped. By default the handler rolemask contains
only FLUX_ROLE_OWNER.

A message handler MAY disable the built-in policy by setting its rolemask
to FLUX_ROLE_ALL (4294967295). It MAY then use message functions to
access userid and rolemask to implement its own algorithm for accepting
or rejecting messages.

FLUX_ROLE_OWNER MUST NOT be excluded from the "allow" rolemask of a message
handler.


Instance Owner
~~~~~~~~~~~~~~

The Flux broker processes comprising a Flux instance SHALL run
as a common userid termed the "instance owner". The instance owner
SHALL have control over the instance and its resources; however,
the instance owner SHALL NOT have the capability to launch work as
other users without their consent.

A system instance MAY run as a dedicated user, such as "flux".
The system instance owner SHALL NOT be the root user.

Other users MAY start their own instances as parallel programs according
to the policy of the enclosing instance.


Overlay Networks
~~~~~~~~~~~~~~~~

The overlay networks are for direct broker to broker communication
within an instance.

Users other than the instance owner SHALL NOT be permitted to connect
to an instance’s overlay networks. Since overlay networks are implemented
using the ZeroMQ messaging library, these connections SHALL be secured
using a configurable ZeroMQ security plugin that implements message privacy,
authenticity, and integrity such as "CURVE" or "GSSAPI".

ZeroMQ security is documented in:

-  `ZeroMQ RFC 23 ZMTP ZeroMQ Message Transport Protocol <https://rfc.zeromq.org/spec:23>`__

-  `ZeroMQ RFC 24 ZMTP PLAIN <https://rfc.zeromq.org/spec:24>`__

-  `ZeroMQ RFC 25 ZMTP CURVE <https://rfc.zeromq.org/spec:25>`__

-  `ZeroMQ RFC 26 CurveZMQ <https://rfc.zeromq.org/spec:26>`__

-  `ZeroMQ RFC 27 ZAP ZeroMQ Authentication Protocol <https://rfc.zeromq.org/spec:27>`__

-  `ZeroMQ RFC 38 ZMTP GSSAPI <https://rfc.zeromq.org/spec:38>`__

The default ZeroMQ security plugin SHALL be "CURVE", which requires
minimal security infrastructure to operate.

When a CURVE public, secret key pair is stored on a file system,
the key pair SHALL be encoded using the ZeroMQ certificate format
documented in:

-  `Securing ZeroMQ: Soul of a New Certificate <http://hintjens.com/blog:53>`__, P. Hintjens, October 2013.

-  `ZeroMQ Certificates, Design Iteration 1 <http://hintjens.com/blog:62>`__, P. Hintjens, October 2013.

A long-term CURVE certificate SHALL NOT be used if it is damaged, or if
file system permissions allow the private key to be disclosed to
users other than the Flux instance owner.  If certificates are stored in
a network file system, it is RECOMMENDED that network file system traffic
be protected from eavesdropping.

A Flux system instance using CURVE security is configured with access to
a single, shared CURVE certificate for the system.

A Flux instance that is launched with PMI self-generates a unique CURVE
key pair within the memory of each broker.  Public keys are shared via the
PMI KVS.  After PMI synchronization, each broker reads the public keys of
its immediate peers, and authorizes them to communicate.

Process Management Interface (PMI)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Programs launched by a Flux instance MAY use PMI services,
a quasi-standard set of APIs and wire protocols, to obtain program
attributes, exchange endpoint information, and to spawn additional tasks.
Programs use PMI in one of three methods:

1. Programs link against a shared library provided by the resource
   manager, which provides well known PMI API calls.

2. Programs are given a connection to the resource manager by passing
   an inherited file descriptor, whose number is communicated with an
   environment variable. Programs then use a well known PMI wire protocol
   to communicate with the resource manager.

3. programs and resource managers link against a shared library provided
   by a standalone PMI implementation, which implements both a well known PMI
   API and a resource manager API. The PMI implementation manages connections
   between programs and resource managers.

In a multi-user instance, PMI service as in (1) SHALL be provided by
a shared library that implements PMI API in terms of its wire protocol,
and proceeds as in (2).

In a single-user instance, PMI service as in (1) MAY be provided by
a shared library that implements PMI API directly in terms of Flux
services, as a stop-gap measure while multi-user PMI is under development.
Security is as described for direct broker connections.

PMI service as in (2) SHALL be provided by a purpose-built Flux service
that implements a handler for PMI wire protocol and pre-connects programs
using file descriptor passing. No security is required in this context.
This PMI service SHALL NOT expose Flux services directly to programs;
for example, the PMI KVS calls SHALL NOT be allowed full access to the
Flux KVS namespace.

PMI service as in (3) requires auditing of the standalone PMI implementation
to ensure that connections are properly secured, and access to Flux services
is limited as in (2). (This is the "preferred" PMIx model - viability TBD).


Other Program Services
~~~~~~~~~~~~~~~~~~~~~~

TBD: Tool interfaces, grow/shrink.


Resource Containment
~~~~~~~~~~~~~~~~~~~~

Programs launched by an instance SHALL be contained within their resource
allotment.

TBD: Unprivileged instance needs to call unshare(2), which requires
CAP_SYS_ADMIN, etc.

TBD: Containment should be implemented as a stack of plugins that execute
at different points in the life cycle of a program.


Integration with OS Security Software
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As a general rule Flux, and the packages it depends on, SHOULD link against
packaged, shared library versions of security significant software provided
by the OS distribution. This allows Flux to receive timely fixes for
security vulnerabilities, without requiring Flux to be rebuilt.
These include:

-  libzmq.so, libczmq.so

-  libsodium.so (libzmq should avoid configuring built in "tweetnacl" alternative)

-  libgssapi_krb5.so, libkrb5.so, libk5crypto.so, etc..

TBD: integration MAY be required with:

-  SELinux

-  Linux pluggable authentication modules (PAM)

-  Linux cgroups

-  Linux private namespaces (unshare(2))

-  systemd

-  SSH


Integration with site services
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

TBD: integration MAY be required with:

-  Kerberos V

-  LDAP

-  file systems


See also
--------

-  `MUNGE Uid 'N' Gid Emporium <https://github.com/dun/munge/wiki>`__, C. Dunlap

-  `Securing ZeroMQ: the Sodium Library <http://hintjens.com/blog:35>`__, P. Hintjens, March 2013.

-  `Securing ZeroMQ: CurveZMQ protocol and implementation <http://hintjens.com/blog:36>`__, P. Hintjens, March 2013.

-  `Securing ZeroMQ: draft ZMTP v3.0 Protocol <http://hintjens.com/blog:39>`__, P. Hintjens, April 2013.

-  `Securing ZeroMQ: Circus Time <http://hintjens.com/blog:45>`__, P. Hintjens, July 2013.

-  `Using ZeroMQ Security (part 1) <http://hintjens.com/blog:48>`__, P. Hintjens, September 2013.

-  `Using ZeroMQ Security (part 2) <http://hintjens.com/blog:49>`__, P. Hintjens, September 2013.

-  `Gist: ZeroMQ with GSSAPI <https://gist.github.com/cbusbey/11265987>`__, C. Busbey, April 2014.
