..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Storage and retrieval of stage logs
===================================

https://blueprints.launchpad.net/solum/+spec/stage-logs

Solum needs to collect the logs of each stage of assembly creation, and
associate those logs with the assemblies for reference and debugging by users
of Solum.


Problem description
===================

At present, users of Solum cannot see the log output from tasks such as
build-app and unit-test. I identify three important steps to provide these
logs to users:

- Solum's worker agent must store the output of the tasks it performs.
- Solum must associate these logs with the appropriate Assembly.
- Solum must communicate this association to users in a simple way. A user must
  be able to find the logs of each of the stages the user's assembly has been
  through with no more information than the assembly id.

I also identify several constraints on this problem:

- Users must only have their own logs presented to them.
- Storage and organization of logs must be as simple as possible to facilitate
  consumption by a service such as Kibana to be run by deployers of Solum, and
  also simple manual consumption by deployers not wishing to deploy and
  maintain additional dependent services to run Solum.
- Multiple worker agents may be running one or more hosts concurrently, so
  individual agents must not inhibit the work of other agents.
- An assembly will likely be rebuilt over the course of an application's
  development; therefore a single assembly may bear more than one set of logs
  from a particular stage. These multiple runs must be distinguished from each
  other, and of special note is finding which of a set of logs is the most
  recent. Once Mistral is used for workflow execution, we can associate these
  logs by Execution, but until then, timestamps is the simplest solution.


Proposed change
===============

These features are best achieved in incremental steps:

- Store the logs of a stage on the host of the worker agent.

  - Develop a naming scheme for ouput files for storage. I propose ::

    /var/log/solum/worker/<assembly>/<stage>/<iso8601>.log

    - For logging the output of a stage, maintaining metadata is important,
      and a file path is not a strong way to associate it, especially with
      multiple hosts simultaneously recording logs. It is then important that
      the metadata remain in the body of the log. To that end, I propose
      recording logs using the following format: ::

        { "@timestamp": "<iso8601 timestamp>",
          "assembly_id": "<assembly id>",
          "solum_stage": "<'unittest', 'build-app', etc>",
           ...
          "message": "<captured output>"
        }

      This format is suggested as a simple way for this logging mechanism to
      interface with log-consuming tools like Logstash or ElasticSearch, or
      syslog.

  - In addition, attaching a field to Assembly to convey the location of its
    relevant logs would be helpful at this stage, using a text field to store
    a dictionary shaped nominally like the following: ::

      {
        unittest: {
          <date1>: <host>:<path/to/file>,
          <date2>: <host>:<path/to/file>
        },
        build-app: {
          <date1>: <host>:<path/to/file>
       }
     }

- Collect and organize the logs from the hosts to a central location.

  - This may be achieved most easily by storing the logs initially on network
    storage, or emitting them direclty to a service like syslog.

- Present the organized logs to users, ideally via additional API methods.

  - Develop a URI scheme for presenting the logs to users. I propose ::

      SOLUM/v2/assemblies/<assembly>/logs/<stage>/<iso8601>.log


Alternatives
------------

- Swift can likely be used for storage, but Solum does not already make use of
  the service. Initially, we will focus on local storage, and work in other
  services such as Swift in later stages.

Data model impact
-----------------

Minimal. Initially, an extra field on Assembly is proposed.

REST API impact
---------------

None

Security impact
---------------

Exposing the output of a docker container running user unit tests against
user code is no more dangerous than running those tests against the code is
originally.

One concern of this format is making sure only the owner of an assembly can
retrieve its logs.

The concern of a user filling up storage is not a large one. Ideally, system
monitor triggers ought to warn long before a user exhausts all available
storage with a single docker container.

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

None

Developer impact
----------------

This setup can be reused by future stages--
functional testing, for example.

Implementation
==============

Assignee(s)
-----------

Ed Cranford (ed--cranford) will implement this logging proposal.

Work Items
----------

- Add configurable path and ensure the directory is created as worker starts.
- Modify worker shell handler to capture and store the output of build and
  unittest tasks.


Dependencies
============

None


Testing
=======

This is a practical change, and difficult to test automatically. This is
further complicated by the differences between the development environment VM
and an actual deployed Solum environment, which may be distributed across
several distinct hosts and present accessibility problems we cannot foresee in
development.


Documentation Impact
====================

The documentation changes must cover minimally:
- For operators, how to configure the logging location.
- For users, how to retrieve the output of assembly stages.


References
==========

None

