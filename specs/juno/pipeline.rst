..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
Solum pipeline
==============

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/solum/+spec/pipeline

The design of Solum's API in release 2014.1.1 focuses primarily on
resources to enable Application Lifecycle Management. It is suitable
for expressing how to deploy an application, but it's not as useful
for modeling a custom build/test/deploy pipeline for a given
application taking into account everything needed in order to produce
a CI/CD environment. This proposed design addresses that concern by
detailing how the default behavior of Solum can be customized to
accommodate a variety of different CI workflows.

Problem description
===================

Solum currently allows integration with simple development process
using Git, and a pre-defined workflow. We plan to add components that
will allow customization of events that happen before the
application's deployment, such as testing, image building, and
advancing between various Environments.

Proposed change
===============

Solum will use Mistral for it's workflow execution and definition.
This needs to be flexible so the user can select different types of
tasks on a range of infrastructural elements.

Program flow
------------
execute pipeline: https://drive.google.com/file/d/0B3SsMUWSuQAlbkdkNmV1bUtsZXc/edit?usp=sharing

Pluggable infrastructure
------------------------

* https://review.openstack.org/100539
* https://review.openstack.org/101212

Alternatives
------------

None

Data model impact
-----------------

* add new pipeline db objects
* add default workbooks


REST API impact
---------------

/v1/pipelines/ (POST/GET)
/v1/pipelines/(pipeline_id)/ (PUT/GET/DELETE)


Security impact
---------------
Solum will use the normal keystone auth, except for the trigger in
which case a trust will be used to perform actions on behalf of the user.


Notifications impact
--------------------

None

Other end user impact
---------------------

* This has support in the client (merged):
  https://review.openstack.org/#/c/100124/

* UI: https://review.openstack.org/101253

Performance Impact
------------------

None - speed of image building and deployment should not change.


Other deployer impact
---------------------


Developer impact
----------------
None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  asalkeld

Other contributors:
  related blueprints will be done by who ever picks up the work.

Work Items
----------

mistral
^^^^^^^
- mistral trust [asalkeld] (auth_token_info - to prevent re-authentication)
- mistral webhook trigger [optional] (https://blueprints.launchpad.net/mistral/+spec/mistral-ceilometer-integration)
- https://blueprints.launchpad.net/mistral/+spec/mistral-multitenancy
  [asalkeld - started, but others can help]

solum
^^^^^
- API and db objects (done)

- Create an empty stack on pipeline create.
  this is to work around the chained trusts issue

- If the workbook does not exist (and we have the definition), create it for the user.
  This is a work around until mistral has workbook sharing.

- Add mistral-plugins for:
  - heat update and status [asalkeld - started]
  - image build [asalkeld - started]
  - unit testing
  - functional testing (hopefully the same as unit test one)

- Add calls to mistral to kick off the execution (in review)

solumclient
^^^^^^^^^^^
- https://review.openstack.org/100124 (merged)
- once we are at least as functional as the assembly, change the cli


Dependencies
============

solum-dashboard
---------------
- investigate mistral dashboard
  can we use the mistral task history?
- add support for pipelines/
- link to mistral tasks if it looks good, else make
  a more build-job like tasks ui (pass/fail)
- since catalog is not a "thing" yet how do we discover our
  capabilities (supported workflows, vm/docker, buildfarm, etc)

see: https://review.openstack.org/101253

Testing
=======

* There will be unit tests
* functional tests can be achieved with fake mistral plugins (to be
  fast and not to require boot images)


Documentation Impact
====================

* The getting started guide will need to be modified.
* The rest api will need to be referenced in the auto generated docs.

References
==========

* https://wiki.openstack.org/wiki/Solum/Environments

* https://wiki.openstack.org/wiki/Solum/Pipeline

* https://blueprints.launchpad.net/solum/+spec/solum-build-farm

* https://blueprints.launchpad.net/solum/+spec/environments
