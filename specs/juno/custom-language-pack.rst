Problem description
===================

This spec considers requirements, implementation, and changes to Solum
CLI and API to support custom language packs.


Problem Details:
------------------
A language pack in Solum is essentially a base image (VM) with appropriate
libraries, compilers, runtimes installed on it.

Custom language pack is a base image that has libraries and packages
specified by the user (cloud operator and/or an application developer)
installed on it.

Towards building such custom language packs Solum needs:
(a) the ability to specify the required base image type and version
(b) the ability to specify the required libraries and packages to be installed
on the base image
(c) the ability to register the language pack with Glance

One way to implement this feature is to require that a user provide two things
to Solum:
(a) base image name and type
(b) link to github repository with a script with a predefined name
(such as 'prepare') in it, which contains installation instructions to install
the required libraries and packages.

Solum would provide a 'language pack builder' API to build the language pack
and register it in glance.

At a high-level the builder API will do following actions in a secure
environment:
(a) mount the specified base image
(b) clone the repository
(c) run the 'prepare' script
(d) snapshot the new image
(e) upload the image to glance


Proposed implementation:
-------------------------

User provides a Dockerfile in their language
pack repository. Solum builds the language pack via 'docker build'.

Note that the language pack created using this approach is going to be a
Docker container and not a VM image.

One of the main advantage of this approach is that it uses Dockerfiles as
standard format which became mainstream in several systems requiring
containers.

Towards using such a language pack in creating a DU (deployment unit), we have
to consider the following.

If the operator has configured Solum to use 'docker' as the SOLUM_IMAGE_FORMAT
then the Docker based LP will work without any issues.

However, if SOLUM_IMAGE_FORMAT is set to 'vm' then we will have to provide
a VM image that has Docker installed. CoreOS is a possible approach
(see this WIP https://review.openstack.org/#/c/102646/).
Another approach would be to use the heat docker plugin.


Additional considerations:
----------------------------
The question of why to separate out language pack creation step and not have
the language pack Dockerfile listed in users' application code repository
can be asked. While this is certainly possible there are several advantages of
separating language pack creation from application building. First, language
pack creation is a one time process. Clubbing it with application
building will lead to build performance overhead. Second, by requiring
application developers to define Dockerfiles in their application
repositories Solum will be binding them
in a contract that their code would work only when the operator has configured
Solum to use Docker as the image format.


Proposed change
===============
1) CLI changes:
- Create a new command 'languagepack build' which will be used as follows:

solum languagepack build <github url of custom lp repository>

This will use the 'builder' API (which runs on port 9778) which Solum already provides.


2) API changes:
(a) Start with our builder API. Modify it if required.
We can start with POST /v1/solum/builder/
The data sent in will be
github repository URL containing the Dockerfile.
This will lead to following steps:

- clone the git repo
- do 'docker build'
- do 'docker push' to Solum's docker registry to upload the LP and make it
  available in Glance


Alternatives
------------

Option 1:
---------
Use disk-image-builder (dib) as the mechanism to build the language pack
The issues with this option are:

(1) Figuring out how do the contents of 'prepare' translate into dib element
(pre-install.d, install.d, post-install.d, finalize.d, etc.)

(2) Figuring out how to run 'prepare' in a sandboxed environment.
We need this as the contents of 'prepare' can be anything.

Advantages:
-----------
The images created are compatible with glance.

Disadvantages:
----------------
One of the disadvantages is that Solum would need to
build a translator to translate contents of 'prepare' in the dib's dsl.

Option 2:
---------
Don't use disk-image-builder, but do similar to what we are currently
doing as part of 'build-app' (i.e. through Solum code perform the steps
of mounting fs, executing installation steps, creating image snapshot).

Advantages:
-----------
One of the advantages of Option 2 is that no translation is required of
the contents of the 'prepare' script.

Disadvantages:
--------------
The disadvantage is that Solum would need to build mechanisms to create a
Glance compatible image.

One approach to achieve advantages of both options without incurring their
disadvantages is to use dibs but avoid converting 'prepare' into various
elements. Just convert it into install.d element.
Overtime Solum can support 'hints' which can be used to indicate how should
'prepare' be broken up to be used within different elements. It is possible
that such hints are provided as separate scripts that are supported by Solum.
For example, Solum may start supporting 'pre_install, prepare, post_install'
scripts.


Data model impact
-----------------

Would need to be changed to support LP creation actions. This includes
changes to the data model to persist build and test status.



REST API impact
---------------

Discussed above


Security impact
---------------
Language pack builds would need to be done in isolated environments so
as to not affect other builds. Need to investigate in detail if Docker-based
approach would provide good enough contained environment.


Notifications impact
--------------------



Other end user impact
---------------------



Performance Impact
------------------
Building a languagepack is expensive operation as it involves building an image
by downloading and installing the specified packages in the Dockerfile.
Depending on the available CPU, network, and memory resources the performance
of this operation will get affected.


Other deployer impact
---------------------


Developer impact
----------------


Implementation
==============

Assignee(s)
-----------

Devdatta Kulkarni (devdatta-kulkarni) will implement this custom language
pack proposal:

Work Items
----------
- Create an example custom language pack
  (https://review.openstack.org/#/c/103671/)
- Make the required REST API changes (identified in proposed changes section)
- Make the required CLI change (identified in proposed changes section)
  (Arati Mahimane has started on this)
- Enchance the data model to store data about image build actions and the
  test results.


Dependencies
============
We will start with the solum-builder api (and code). Once the build-farm
stuff is merged we can revisit this to see if that would be something
that can be used for this.


Testing
=======
Test that the created language pack was installed with all the packages and
libraries listed in the Dockerfile.
One way to achieve this would be to delegate the testing responsibility
itself to the language pack author. For instance, lp author could use
RUN statements within their Dockerfile that does the necessary checks.
We would require that after checks are completed a test result file be
created (say, /solum.lp.test) which contains the language pack creation
status. We can then look for that to determine if the testing passed or not,
without worrying about figuring out what package management style was used.

If language pack creation fails then Solum will take following actions:
- Save the details of the build and test results in Solum's internal database
- Log the build failure status in log stream for the user's actions
- If a user's email address is available, send a notification of build failure
to that address. Solum would first need to be enhanced with a
mail server functionality to support this.


Documentation Impact
====================
Documentation would need to be updated to reflect the addition of new CLI
commands.


References
==========

https://blueprints.launchpad.net/solum/+spec/custom-language-packs

https://etherpad.openstack.org/p/custom-language-packs
