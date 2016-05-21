----
authors: Alex Wilson <alex.wilson@joyent.com>
state: predraft
----

<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright 2016 Joyent, Inc
-->

# RFD 34 SDC A&A Overhaul (AUTHAPI)

# Background

## Authentication and Authorization

Two key components of any infrastructure that enforces security policy are
authentication and authorization.

The term "authentication" refers to the establishment and proof of identity of
a user or component of the system. This is normally by way of the agent 
presenting information that could only be known to them if they possessed the
identity being authenticated, such as a password or the output of a 
cryptographic primitive (such as a key signature or a step in a hash chain).

We will refer to the abstract identity as a "principal", which a "user", 
"agent" or "client" must prove that they possess in the authentication process.

The other component, "authorization", is responsible for ascertaining what 
actions the principal may take once they have been authenticated. It
encompasses access control lists, roles, user groups, and everything needed in
order to make these decisions within a given system.

Authentication is generally a fairly well-established part of any system, and 
SDC is no exception. It tends to require fairly similar processes and steps
across a wide variety of systems, and as a result is generally both very well
studied and well practiced by engineers who care to do some reading on the 
topic before proceeding to implementation.

Authorization, on the other hand, is where the fundamentals of identity have 
to be mapped to the possible actions and resources in each individual
application. This component exhibits high variability of design between
systems, though certain general patterns and schools of thought have been
established.

## Role-Based Access Control (RBAC)

One of these general patterns is known as Role-Based Access Control, or RBAC. 
The notion of RBAC was formalized in the "NIST model", by Sandhu et al (2000),
after being originally proposed and experimented with through the 1980s and
1990s.

The NIST model approaches authorization by assigning principals to be members 
of one or more *roles*. Roles, then, grant *permissions* to access certain
*resources* in the system. These permissions come in the form of access control
rules -- tuples of the form `(action, resource)` representing actions that will
be allowed.

If no permissions is granted via a role for a given principal to take an 
action, then that action is denied. This is a so-called *default-deny*
authorization model.

# A&A in SDC: a short history

Originally, SDC possessed an extremely simple authorization scheme: there were
SDC *accounts*, and resources were always associated with exactly one account.
Accounts had all possible access to their own resources and nothing else.

During the development of Manta, it became obvious that this simplistic model
was not going to meet the needs of a number of major customers, and so a new
system (loosely) based on NIST RBAC was devised.

## The Manta model

 * Manta *users* are billable principals that can own manta resources and are 
   their own segment of the manta directory tree.
 * *Ownership* of a resource implies full action rights over that resource.
 * *Sub-users* are principals that are subservient to their parent *user*.
   They cannot own resources themselves and must be granted explicit rights
   to do anything.
 * Principals can be members of a *role* in two ways: regular membership, where
   they must *take up* the role by supplying appropriate headers with each
   request; and default membership, where the role is always active.
 * Roles can be associated with *policies*, which are lists of actions.
 * Roles can be associated with *resources*, in the form of *role tags*.

Note that in order to grant any access to a principal, the full set of 
associated *role*, *policy* and *role tag* must be present.

The reason why *role tags* are used is essentially to bridge the gap between
the NIST model of resources (a flat space of unstructured specifiers) and
Manta's resource model (a UNIX-style filesystem tree). Having to specify
every possible file on the filesystem that should be targetted by a policy
is prohibitive for administrators, and prefix/suffix matching is insufficient
for many uses.

## The Manta model comes to SDC

With the success of the Manta model with the original customers for whom it was
devised, there was pressure to update the SDC authorization model in a similar
fashion. To avoid duplication, it was decided to simply copy the same model
used in Manta, and make the two share their data.

Unfortunately, SDC would actually have been a better fit for something closer
to the stock NIST model, as it is not organised as a hierarchical filesystem.
To resolve the mismatch, it was simply decided that the role tags would be
applied to CloudAPI URLs (which look like an hierarchical filesystem if you
squint a bit).

The action verbs used by the Manta model were also defined largely by the
filesystem semantics of the storage system. For SDC, it was decided to instead 
use the internal names given to each CloudAPI endpoint (also in the
documentation) as the action names. This creates some curious points of
overlap, where the endpoint in question has to be doubly stated by the
administrator: first in the action name, and then in the resource to be role-
tagged.

While this adoption of the Manta model in SDC was certainly expedient at the
time, it has not enjoyed very much success. This seems to be due it being 
considered difficult to use and poorly documented. Little material was ever
written or supplied to users to explain how to use the RBAC model with SDC, and
the APIs used to control it were not very discoverable. There is no way,
for example, to list all of the available verbs, or discover what the correct
resource URL to use for a particular SDC object should be, making the writing
of effective policy quite challenging indeed.

## SDC Authentication

As well as the authorization scheme, there has been some history in the
development of authentication in SDC and Manta.

Originally, all information about customers was stored in CAPI (the Customer
API). During the development of SDC 7, it was decided that an LDAP-based
database would be a better store of this information than the existing CAPI.
This, combined with a dissatisfaction with other existing LDAP databases, lead
to the development of UFDS.

UFDS is an LDAP server implemented atop Moray, the key-value document store
developed primarily for use in Manta. Manta is very well-served by the key-
value paradigm, as most of its stored data is very close to immutable, and
very rarely looked up by anything other than a single primary key (e.g. an
object's path in the filesystem hierarchy). LDAP's underlying X.509 directory
structure, however, is a strict tree, and LDAP servers generally expect to
serve a lot of workload on lookups that are not on primary keys (DNs or
distinguished names). LDAP objects also experience frequent updates to single
properties (such as to change a password or update a timestamp). There is thus
a very high impedance mismatch between the key-value document store paradigm that Moray provides, and the LDAP paradigm that UFDS must show to the outside
world. This is generally considered unfortunate.

LDAP could have had some interesting advantages for SDC, such as being able to
integrate with PAM and the operating system authentication and authorization
model. Again, unfortunately, the LDAP schema that was selected for SDC's use
with UFDS precluded this, and UFDS grew into an LDAP store that was really only
ever used as yet another internal database, a job at which it has never proved 
especially adept.
