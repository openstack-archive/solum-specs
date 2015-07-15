..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
Deployer Plugins
================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/solum/+spec/deployer_plugins

Currently, any customization or user-selectable variety in application
deployment architecture is exclusively governed by available Heat templates
that the single deployer has access to. While sufficient for a few simple
architectural topologies sharing the same parameter requirements and update
workflows, more numerous and complex options make this single deployer
design untenable.

Problem description
===================

The current deployer design makes implementing many types of architectural
topology options difficult if not impossible. Relying solely on different
templates is insufficient since customization is limited by having to either
restrict the needs of these architectures to a strictly limited set of
parameters or requiring every template to include every possible parameter
of every other template.

Additionally, this mechanism doesn't allow for various different update
strategies. For example, an HA architectural topology may require multiple
calls to stack update and modifications to the template that simply
cannot be accounted for in a "one-size-fits-all" deployer design.

Finally, the current design requires operators to patch the deployer
should they wish to rely on different deployment mechanisms and/or
workflows.

Proposed change
===============

The proposed change would use Stevedore plugins to allow the deployer to be
easily extended to support various different architectural topologies. These
plugins would also expose any additional parameters that are relevant to and/or
allowed for in the particular architectural topology it implements.
Functionality such as generating pre-authenticated urls and general preamble
would still be handled by the current deployer class. However, in this proposed
design, the deployer would pass parameters and other user selections on to the
desired plugin which would then handle the actual deployment (template
selection, manipulation, and interaction with Heat or other provisioning
strategies). The plugin will also be responsible for monitoring provisioning
status and reporting same to the deployer. These plugins would still run in the
same process as the deployer and would not require any additional
synchronization or communication mechanisms.

Each plugin would optionally define properties similar to the way Heat
resource plugins do. These allow the plugin author to define the
user-configurable options available for that architectural topology. Property
definition will be similar to that found in Heat resource plugins. When an
application is created, the user will be able to pass property values which
are then validated and used in provisioning by the chosen plugin. The user
can query the API for available architectural topologies and their properties.
The API will get this information by requesting it from the deployer which in
turn will simply examine its loaded plugins.

Alternatives
------------

As mentioned in the Problem Description, various templates and ever-branching
logic in the deployer code could be relied on for a time, but this is
untenable in the long term.

Data model impact
-----------------

The ``app`` table will need to be modified to include a ``topology``
column to identify the plugin used to deploy the application. Since the
deployer plugin is now responsible for provisioning, the ``stack_id`` column
can be replaced by a generic json blob column called ``deployer_info``. This
column can contain arbitrary data that the specific plugin needs (stack ids,
persistent property values, etc).

REST API impact
---------------

The api would be extended to include the following calls:

*GET /topologies*
  Return a list of supported architectural topologies:

  - Response codes: 200
  - Returns a list of topologies and their short descriptions

*GET /topologies/<topology>*
  Return details of the specified topology:

  - Response codes: 200
  - Returns details of a topology similar to the response from
    ``heat resource-type-show``

Additionally, application creation and update would be extended to accept
parameters. These parameters and their values will be validated by the plugin
implementing the specified topology.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

Python-solum client will need to add the following corresponding methods:

* ``solum topology list``
  Return a list of supported architectural topologies
* ``solum topology show``
  Return details of the specified topology

Performance Impact
------------------

None

Other deployer impact
---------------------

Deployers will need to be aware of plugin packaging and deployment should
they wish to use this mechanism for extension via custom plugins that are not
distributed by default.

Developer impact
----------------

This is a new method of adding and maintaining deployer code. Developers will
need to be aware of and familiar with Stevedore plugins and how they are
defined, registered, and loaded.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  randallburt

Work Items
----------

* include Stevedore and create deployer plugin base class
* refactor current deployer to load plugins for topologies
* refactor existing "basic" flavor to be a plugin
* refactor tests and add coverage for manager and basic plugin
* add topology listing and detail to the api
* add functional tests (Tempest) for topology listing and detail
* add topology listing and detail to the cli


Dependencies
============

* Stevedore <http://docs.openstack.org/developer/stevedore/> will be an
  additional dependency in ``requirements.txt``.

Testing
=======

Tempest tests for the new api endpoints will be added to cover basic
functionality. Application deployment and other functions should not
impact current tests for them; in fact, current tests are required to pass as
is to prove no regressions


Documentation Impact
====================

Documentation for python-solumclient will need to be updated with the new
operations.


References
==========

None.
