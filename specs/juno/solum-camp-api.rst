..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Solum CAMP API
==========================================

https://blueprints.launchpad.net/solum/+spec/solum-camp-api

The Cloud Application Management for Platforms (CAMP) [CAMP-v1.1]_
specification is an open standard for managing applications in a PaaS
environment. CAMP was created with the goals of increasing
interoperability between independent application management tools and
PaaS clouds as well furthering the portability of applications across
different clouds. Due to its mindshare, momentum, etc. OpenStack is an
obvious candidate for a CAMP implementation. The Solum project, which
is partially based on CAMP, is the natural place for this
implementation.


Problem description
===================

Although the Solum API and resource model are similar (and in some
cases identical) to the API and resource model defined in the CAMP
specification, they are also different in a number of significant
ways. Tools and applications written to consume the CAMP API cannot
use the Solum API. What is required is to provide a CAMP facade on the
services provided by Solum.


Proposed change
===============

This specification proposes adding an alternate, CAMP compliant, API
to the Solum API service. This API (hereafter called the "Solum CAMP
API") will exist alongside the current REST API (hereafter referred to
as the "Solum API").

This proposal is scoped by the following constraints:

* The existence of the Solum CAMP API shall have no impact on Users
  and Deployers that interact exclusively with in the Solum API. The
  "solum" command-line client, the Horizon plugin, the python-solum
  library etc. shall be unaffected by the Solum CAMP API.

* The use cases supported by the Solum CAMP API shall be a subset of
  those supported by the Solum API; Users should not have to choose
  between alternate sets of functionality. Consumers of the CAMP API
  should do so because their application/tool requires that API.

* From a functional perspective, entities (applications, plans,
  components) created by interacting with one API shall be visible via
  the other API. For example, if a User creates an application via the
  Solum API, another User should (if they are authorized to do so) be
  able to see the "assembly" resource that represents that application
  via the Solum CAMP API.

* From an implementation perspective the Solum CAMP API should, to the
  greatest extent possible, re-use existing Solum classes and
  configuration settings.

* The architectural constraints and implementation conventions of
  Solum shall be strictly adhered to.

Alternatives
------------

None.

Data model impact
-----------------

The impact of the Solum CAMP API on the data model can be divided into
two separate areas: impacts on the data model due to the
implementation of CAMP-specific resource types and impacts on the data
model due to the difference between Solum resources and their CAMP
analogs (i.e. the difference between the Solum API's the CAMP API's
versions of the "assembly" resource).

CAMP-specific resources
^^^^^^^^^^^^^^^^^^^^^^^

The CAMP-specific resources (i.e. resource types that exist in the
CAMP API but not the Solum API) are either static (e.g. the
"platform_endpoints" resource or the "type_definitions" resource) or
act as collections of other resources (e.g. the "services"
resource). The information presented by these resources is either a
reflection of configuration information or information about other
resources. In neither case is it necessary to store data about
multiple instances of these resource types.

CAMP-analog resources
^^^^^^^^^^^^^^^^^^^^^

At the time of this writing it appears that the majority of the
CAMP-analog resources (for example, the CAMP version of the "assembly"
resource) can be implemented without changing the database
schema. This is due to the fact that most of the CAMP-required
attributes that are missing from their Solum API counterparts are
Links to other resources. The information necessary to synthesize these
Links is present in Solum, but is presented in a different fashion by
the Solum API.

*If* additional information is required to support a CAMP-analog
resource, the suggested solution would be to create an additional
table that contains the CAMP-unique information and cross-references
the Solum resource using its id as a key.


REST API impact
---------------

CAMP's REST API is described by the Cloud Application Management for
Platforms (CAMP) specification. The URLs for Solum CAMP API resources
will exist in a separate sub-tree (named "camp") from those of the
Solum API.

The URL of the "platform_endpoints" resource (the resource that is
used to advertise the existence of distinct CAMP implementations) will
be:

    *Solum_API_base_URL*/camp/platform_endpoints

The URL of the "platform_endpoint" resource for the CAMP v1.1
implementation will be:

    *Solum_API_base_URL*/camp/camp_v1_1_endpoint

The URL of the "platform" resource (the root of the CAMP v1.1 
resource tree) will be:

    *Solum_API_base_URL*/camp/v1_1/platform

Security impact
---------------

Since the Solum CAMP API functions as an alternate interface to the
core Solum functionality, the addition of this API should not create
any additional attack vectors beyond those that may already exist
within Solum.

Notifications impact
--------------------

The Solum CAMP API will send the same notifications, for the same
events, as the existing Solum API.

Other end user impact
---------------------

Users employing CAMP compliant tools for managing their applications
will be able to use OpenStack/Solum without having to change these
tools.

Performance Impact
------------------

None.

Other deployer impact
---------------------

The Solum CAMP API will be enabled/disabled via a configuration option
(e.g."camp-support-enabled = [True | False]"). The Solum CAMP API will
be enabled by default. Solum's configuration documentation will be
updated to describe this option and the effect of enabling/disabling
it.

Developer impact
----------------

Adding additional code to the Solum project will have a maintenance
impact as features are added and bugs are fixed. For example, a change
to a handler class that is shared by the Solum API and CAMP API could
break the CAMP API code. The following steps will be taken to address
this impact:

* Ensure that interface between the Solum CAMP API and the core Solum
  code is as clean as possible. This decreases the probability that
  unrelated changes will break the CAMP API code or that changes to
  the CAMP API will break other Solum code.

* Assign resources to maintain the Solum CAMP API code. The
  implementation assignees identified below will be assigned this
  task.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gilbert.pilz

Other contributors:
  anish-karmarkar

Work Items
----------

static resources
^^^^^^^^^^^^^^^^

Some of the resources defined in CAMP are static for a given
deployment and configuration. This work item will implement those
resources.

:Resources to be implemented:
   platform

   platform_endpoints

   platform_endpoint

   formats

   format

   type_definitions

   type_definition

   attribute_definition

At the completion of this step it will be possible to perform a
successful HTTP GET on these resources. Some of the attributes in
these resource may be missing or contain dummy values.

top-level container resources
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This work item will implement the "top-level" container resources
defined by CAMP.

:Resources to be implemented:

   assemblies

   services

   plans

Upon completion of this item it will be possible to perform a
successful HTTP GET on these resources. The Link arrays in these
resources will reference the Solum API versions of the "assembly",
"service", and "plan" resources (respectively) even though the Solum
versions of some of these resources are not CAMP-compliant.

register a Plan via the Solum CAMP API
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This item will implement the code necessary to allow Users to POST a
Plan to the *Solum_API_base_URL*/camp/v1_1/plans resource using one of
the methods described in Section 6.12.2, "Registering a Plan by Value"
of the CAMP Specification.

Upon completion of this item it will be possible to POST a Plan to
*Solum_API_base_URL*/camp/v1_1/plans and have the contents of that
file appear as a "plan" resource in both the
*Solum_API_base_URL*/camp/v1_1/plans resource (as an element in the
plans_links array) and *Solum_API_base_URL*/v1/plans resource (as per
today). Note the "plan" resource in question will be the Solum version
of the resource; at this point there will be no CAMP-analog of the
"plan" resource.

Sending a DELETE request to the
*Solum_API_base_URL*/camp/v1_1/plan/*uuid* resource will remove the
"plan" resource from both the CAMP and Solum collections.

create an assembly from a reference to a plan resource
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This item will implement the code necessary to allow Users create a
running application by POST-ing a reference to a "plan" resource to
the *Solum_API_base_URL*/camp/v1_1/assemblies resource as described in
Section 6.11.1, "Deploying an Application by Reference" of the CAMP
Specification.

:Resources to be implemented:
   assembly

   component

Upon completion of this item it will be possible to POST a reference
to a plan resource to *Solum_API_base_URL/camp/v1_1/assemblies* and
have Solum build and create a running application. This application
will be represented by two, analogous resources, a CAMP version
(*Solum_API_base_URL*/camp/v1_1/assembly/*uuid*) and a Solum version
(Solum_API_base_URL/v1/assemblies/*uuid*). Each resource will be
referenced by its corresponding container. The CAMP version of the
"assembly" resource will reference a tree of CAMP-specific "component"
resources (also analogs of their Solum counterparts)

Sending a DELETE request to the
*Solum_API_base_URL*/camp/v1_1/assembly/*uuid* resource will halt the
application and remove it from the system, removing both the CAMP and
Solum versions of the "assembly" resource and any corresponding
"component" resources.

create an assembly directly from a Plan file
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This item will implement the code necessary to allow Users to create a
running application by POST-ing a Plan to the
*Solum_API_base_URL*/camp/v1_1/assemblies resource using one of the
methods described in Section 6.11.2, "Deploying an Application by
Value" of the CAMP Specification. This will be implemented by
combining the implementations of the two previous items.

Upon completion of this item it will be possible to POST a Plan file
to *Solum_API_base_URL*/camp/v1_1/assemblies and have Solum build and
create a running application.  As a side-effect, a "plan" resource
will be created. The "assembly" and "component" resources will be the
same as for the preceding item.

Sending a DELETE request to the
*Solum_API_base_URL*/camp/v1_1/assembly/uuid resource will halt the
application and remove it from the system, removing both the CAMP and
Solum versions of the "assembly" resource and any corresponding
"component" resources but not the "plan" resource.

select_attr support
^^^^^^^^^^^^^^^^^^^

The CAMP specification requires implementations to support the use of
the "select_attr" query parameter as defined in sections 6.5, "Request
Parameters", and 6.10.1.1, "Partial Updates with PUT", of the CAMP
specification.

The Solum API does not support the use of the “select_attr” query
parameter. This item will add support for “select_attr”, for both
GET and PUT, to all resources exposed by the Solum CAMP API.

Upon completion of this item it should be possible for Users to use
the “select_attr” query parameter in conjunction with the GET and PUT
methods to retrieve and update (where permitted) a subset of a
resource’s attributes.

HTTP PATCH / JSON Patch support
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The CAMP specification requires implementations to support the use of
the HTTP PATCH method in conjunction with the
"application/json-patch+json" [RFC6902]_ media type.

The Solum API does not support the HTTP PATCH method. This item
will add support for the use of JSON Patch in conjunction with the
HTTP PATCH method to all resources.

Upon completion of this item it should be possible for Users to use
HTTP PATCH with a JSON Patch payload to update (where permitted) all
or subset of a resource’s attributes.


Dependencies
============

No dependencies other than those that already exist in Solum.


Testing
=======

In addition to unit tests, this project will develop a number of
tempest tests which will exercise the main use cases of the Solum CAMP
API. These tests will have the following pattern:

1. Walk the Solum CAMP API resource tree to find the appropriate
   resource (e.g. "assemblies" resource)

2. Perform some action on that resource (e.g. POST a Plan file to that
   resource).

3. Verify that the action has produced the desired result (e.g. the
   creation of a new "assembly" resource.

The assertions defined in the "Cloud Application Management for
Platform (CAMP) Test Assertions" [CAMP-Test-Assertions-v1.1]_ document
will be used, where appropriate, to verify the proper behavior of the
API. Note, it is not our intention to cover every assertion defined in
this document, but simply to leverage the work that has been done in
this area.


Documentation Impact
====================

Information about the CAMP API (specifications, primers, etc.) is
provided by the OASIS CAMP Technical Committee.

Information about enabling/disabling the Solum CAMP API and any other
configuration information will be added to the Solum documentation.


References
==========

Specifications
--------------

.. [CAMP-v1.1] *Cloud Application Management for Platforms Version
   1.1.* Edited by Jaques Durand, Adrian Otto, Gilbert Pilz, and Tom
   Rutt. 12 February 2014. OASIS Committee Specification Draft 04 /
   Public Review Draft 02.
   http://docs.oasis-open.org/camp/camp-spec/v1.1/csprd02/camp-spec-v1.1-csprd02.html
   Latest version:
   http://docs.oasis-open.org/camp/camp-spec/v1.1/camp-spec-v1.1.html

.. [CAMP-Test-Assertions-v1.1] *Cloud Application Management for
   Platforms (CAMP) Test Assertions v1.1.* Edited by Jaques Durand, Adrian
   Otto, Gilbert Pilz, and Tom Rutt. 12 February 2014. OASIS Committee
   Specification Draft 01 / Public Review Draft 01.
   http://docs.oasis-open.org/camp/camp-ta/v1.1/csprd01/camp-ta-v1.1-csprd01.html
   Latest version:
   http://docs.oasis-open.org/camp/camp-ta/v1.1/camp-ta-v1.1.html

.. [RFC6902] Bryan, P., Ed., and M. Nottingham, Ed., "JavaScript
   Object Notation (JSON) Patch", RFC 6902,
   April 2013. http://www.ietf.org/rfc/rfc6902.txt

Implementations
---------------

* nCAMP, CAMP v1.1 Proof of Concept.
  http://ec2-107-20-16-71.compute-1.amazonaws.com/campSrv/
