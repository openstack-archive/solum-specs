..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Git As a Service
==========================================

https://blueprints.launchpad.net/solum/+spec/solum-git-push

Enable users to push their code to Solum (via git server(s) hosted by Solum).

Following is the minimum viable use-case:

A user who is 'registered' with Solum will be able to do git push to their
code's remote in Solum.

Once the code is pushed, it will trigger some set of steps (a workflow)
which will generate/update running application.

Problem description
===================

Currently, we can only use external hosted repositories

Proposed change
===============

Add git hosted repositories in Solum, the same way we will do with the build
farm

To do this, I propose to use `gitolite <https://github.com/sitaramc/gitolite>`_
which is a simple Open Source tool to manage ssh repositories and access rights
Gitolite doesn't come with a UI, and is focus on access control.

We will provide QCOW2 and Docker images for Gitolite. In a first attempt, repo
would be store on ephemeral storage. but we should consider using cinder
volumes in a next iteration (For Docker, we would depend on Cinder support in
Docker)

Alternatives
------------

None

Data model impact
-----------------

We will reuse the infrastructure object and the git_url would be store in the
plan

REST API impact
---------------

This will reuse the infra endpoint proposed in solum-build-farm

Security impact
---------------

We will need a ssh keypair for each user who wants access to a git repository.
We will also need to be able to add/remove user access to a git repo. This can
be described in another blueprint

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

In a first implementation, Git VMs/Containers would be created "per tenant" but
are solum's responsibility to maintain and backup. In another iteration, this
will be driver by policy from the operator.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  vey-julien

Other contributors:
  None

Work Items
----------

* Create VM and Docker images for Gitolite

* Store git repos info in our DB

* Add git hooks to trigger Solum Pipelines when a repo is created

* Use the same mechanisms to configure the instances as described in build farm

Dependencies
============

Cinder support in Docker (Alternative is to run Docker containers on CoreOS
VMs, work on that was start by PaulCzar)

Testing
=======

Functional testing if it doesn't have too much impact on our gate performance

Documentation Impact
====================

Changes to the development and deployment process will need to be documented.

References
==========

Whiteboard on https://blueprints.launchpad.net/solum/+spec/solum-git-push with
previous discussions
