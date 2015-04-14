..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
An app resource for managing applications
===========================================

https://blueprints.launchpad.net/solum/+spec/app-resource

This specification lines out a new first-class resource, the app, to be
presented by the API as a simplified collection of assemblies, plan
registrations, logs of actions, and infrastructure.


Problem description
===================

Interacting with Solum presently involves managing languagepacks, plans,
assemblies, and sometimes components, heat stacks, pipelines, and services.
The CLI has been augmented with a virtual "app" resource to simplify some of
these interactions to reduce the resources to languagepacks and these new apps.
Unfortunately, this simplification isn't available in the Solum API, and users
of the API outside the CLI are missing out on this interaction.

In addition, due to the direct reference to plan in assembly, plans may not be
deleted if there are any existing assemblies referencing it, meaning at present
that soft deletions aren't possible for assemblies with our current models.

As a result of these hard deletes, a lot of arguably important data about Solum
apps is destroyed when an assembly or plan is deleted. It's very difficult to
reliably track the progress of an assembly as a third party, but trivial to do
it from within the engine.


Proposed change
===============

Some elements of an application should persist between builds and deployments,
like the networking information, or the overall status of an application.
Other elements are complicated enough to warrant being separate resources,
though not top-level due to their direct reliance on other resources, and to
their nature of changing more frequently than the app as a whole.

In addition to standard Openstack resource metadata, an app will own configuration
information including where and how to get a users's code, what languagepack to use
to build apps, and what steps to follow to produce the app.

The app will also bear some living data that is not directly mutable by users, including
a heat stack that contains a load balancer and a possibly a container running the user's
app.

An app resource will own a series of workflows, and the app's status will be
an aggregate of the states of those workflows. An app can be simultaneously deployed
and building, for example. These workflows will glean information about the steps
to execute, and how to execute those steps, from the app's configuration. Should
some of that data require evaluation, as in the case of source revisions pointing
to the head of a branch or deploying the most recent successful build of an app,
this information will be stored with the workflow to avoid ambiguity.

As an app's workflows progress through the stages of their execution, they may
produce logs or container images in addition to the result of a stage. This
data will be stored along with timestamp and relevant association data so the
complete history of an app can be discerned with simple queries.


Alternatives
------------

One of the major pain points in manipulating Solum resources at present is
understanding the relationship between plan and assembly, and what part each
resource represents in an application. At minimum, decoupling assembly's direct
reference to plan would be a good start, and could open the way to adding
versioning to plans without necessarily needing to rebuild every assembly.
Having an assembly keep a current record of which plan, or more specifically
which code source and test/run commands were run would make for a much clearer
interface.

Discerning from the API how many assemblies were created with a given plan is a
tedious task, requiring fetching all a user's assemblies and filtering them by
inspecting their plan_uri, and parsing that to fetch the uuid. At minimum, some
search tools ought to be built into the API for example to filter, paginate,
and order these assemblies by their fields, including created and updated
dates. In any case, the conveniences the official CLI provides ought to be
pulled back into the API to make Solum more accessible.


Data model impact
-----------------

- Plan table to be supplanted by new App table; relevant registration data like
  repository, branch, and test commands to be part of app entries.

- Workflow table to be added to track progress of an app. An app can have
  several workflows in various stages of completion at any given time.

- History table to track status changes, created logs, created artifacts,
  and even rows deleted from the app table.

- Assembly table to be removed. Containers will still be used to do the actual
  work of testing, building, and deploying, but they'll belong to workflows and
  their progress will be tracked accordingly.

App table

- Immutable fields:

  - (standard metadata)

  - :code:`languagepack_id`: reference to images.id

  - :code:`load_balancer`: entry point of deployed app, points to LB

  - :code:`heat_stack_id`: stack that contains persistent LB

  - :code:`entry_points`: JSON array of user entry points, like {"web": <url>, "db": <url>}

  - :code:`trigger_id`: type-4 uuid for building trigger URLs

  - :code:`language_pack_id`: reference to images.id, indirectly mutable

- Mutable fields:

  - :code:`name`: human-set and -readable title

  - :code:`description`: human-set

  - :code:`ports`: list of ports on the app container to expose

  - :code:`source`: JSON object containing repository, revision, and pertinent auth information

  - :code:`workflow_config`: JSON object containing test command, run command, and pertinent config information

  - :code:`trigger_actions`: JSON array of commands to be executed when the trigger is invoked.


Workflow table:

- Immutable fields:

  - (standard metadata)

  - :code:`app_id`: reference to app.id

  - :code:`wf_id`: incremental id, orders workflows that share app_id

  - :code:`source`: snapshot of app.source

  - :code:`config`: shapshot of app.workflow_config

  - :code:`actions`: JSON list of actions to be completed on app


Workflow history table:

- Immutable fields:

  - (standard metadata)

  - :code:`app_id`: weak reference to app.id, for easier indexing and searches

  - :code:`workflow_id`: weak reference to workflow.id

  - :code:`source`: snapshot of workflow.source

  - :code:`config`: snapshot of workflow.config

  - :code:`action`: current stage of workflow execution

  - :code:`status`: progress of mentioned :code:`action`; one of *QUEUED*, *IN PROGRESS*, *SUCCESS*, *FAILURE*, and *ERROR*

  - :code:`logs`: JSON array of URIs pointing to any created logs

  - :code:`artifacts`: JSON array of URIs pointing to any created artifacts


REST API impact
---------------

**App Commands**

The app is the primary resource Solum manages. In addition to its own metadata
and information, an app also owns a list of workflows. An app's status is an
aggregate of the status fields of its child workflow resources. As such, an
app can be deployed, building, and testing at once. It can also have a failed
build and remain deployed with no problem. Being a first-class REST resource,
an app has the standard Create, Read, Update, and Delete verbs:

List all Apps::

  GET /apps

  200 OK
  {
    "apps": [
      {
        "uuid": "039db61a-b79a-43b3-821f-1e84e49fbdf3",
        "status": {
          "test": "IN PROGRESS",
          "build": "QUEUED",
          "deploy": "SUCCESS",
        },
        "created": <datetime>,
        "updated": <datetime>,
        "app_url": "http://192.0.2.100",
        "language_pack": "python27",
        "trigger_url": "/triggers/f2970536-c225-4959-9634-ddaf162cc214",
        "name": "ghost",
        "description": "My ghost blog",
        "ports": [80],
        "source": {
          "repository": "http://github.com/fakeuser/ghost.git",
          "revision": "master",
          "oauth_token": "ghostblog-token"
        },
        "workflow_config": {
          "test_cmd": "tox -epep8 -epy27",
          "run_cmd": "/app/bin/run-blog.sh -f /app/config/ghost.conf",
        },
        "trigger_actions": [
          "test",
          "build",
          "deploy"
        ],
        "workflows": [
          ...
        ]
      }
    ]
  }


Create an App::

  POST /apps/
  {
    "name": "djangodemo",
    "description": "Simple todo-list app",
    "ports": [80],
    "source": {
      "repository": "http://github.com/fakeuser/fakedjangoapp.git",
      "revision": "master"
     },
    "language_pack": "python27"
    "workflow_config": {
      "test_cmd": "tox -epep8 -epy27",
      "run_cmd": "bin/start-app.py",
    },
    "trigger_actions": [
      "test",
      "build",
      "deploy
    ]
  }

  200 OK
  {
    "app": {
      "uuid": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
      "status": {
        "test": "IN PROGRESS",
        "build": "QUEUED",
        "deploy": "SUCCESS",
      },
      "created": <datetime>,
      "updated": <datetime>,
      "app_url": "http://192.0.2.101",
      "language_pack": "python27",
      "trigger_url": "/triggers/4ed0cd4b-da91-4552-a9c0-8c4a49fd2f56",
      "name": "djangodemo",
      "description": "Simple todo-list app",
      "ports": [80],
      "source": {
        "repository": "http://github.com/fakeuser/fakedjangoapp.git",
        "revision": "master",
      },
      "workflow_config": {
        "test_cmd": "tox -epep8 -epy27",
        "run_cmd": "bin/start-app.py",
      },
      "trigger_actions": [
        "test",
        "build",
        "deploy"
      ],
      "workflows": []
    }
  }

Show one App::

  GET /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e

  200 OK
  {
    "app": {
      "uuid": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
      "status": {
        "test": "IN PROGRESS",
        "build": "QUEUED",
        "deploy": "SUCCESS",
      },
      "created": <datetime>,
      "updated": <datetime>,
      "app_url": "http://192.0.2.101",
      "language_pack": "python27",
      "trigger_url": "/triggers/4ed0cd4b-da91-4552-a9c0-8c4a49fd2f56",
      "name": "djangodemo",
      "description": "Simple todo-list app",
      "ports": [80],
      "source": {
        "repository": "http://github.com/fakeuser/fakedjangoapp.git",
        "revision": "master",
      },
      "workflow_config": {
        "test_cmd": "tox -epep8 -epy27",
        "run_cmd": "bin/start-app.py",
      },
      "trigger_actions": [
        "test",
        "build",
        "deploy"
      ],
      "workflows: [
        ...
      ]
    }
  }

Update one App::

  PATCH /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e
  {
    "description": "To-do list with new engine",
  }

  200 OK
  {
    "app": {
      "uuid": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
      "status": {
        "test": "IN PROGRESS",
        "build": "QUEUED",
        "deploy": "SUCCESS",
      },
      "created": <datetime>,
      "updated": <datetime>,
      "app_url": "http://192.0.2.101",
      "language_pack": "python27",
      "trigger_url": "/triggers/4ed0cd4b-da91-4552-a9c0-8c4a49fd2f56",
      "name": "djangodemo",
      "description": "To-do list with new engine",
      "ports": [80],
      "source": {
        "repository": "http://github.com/fakeuser/fakedjangoapp.git",
        "revision": "master",
      },
      "workflow_config": {
        "test_cmd": "tox -epep8 -epy27",
        "run_cmd": "bin/start-app.py",
      },
      "trigger_actions": [
        "test",
        "build",
        "deploy"
      ],
      "workflows": [
        ...
      ]
    }
  }


Delete one stopped App::

  DELETE /apps/039db61a-b79a-43b3-821f-1e84e49fbdf3

  204 NO CONTENT


**Workflow commands**

An app manages an application through each stage of its CI/CD lifecycle. The
workflow commands expose control verbs to the user for direct interaction.

Test an app's code::

  POST /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e/workflows
  {
    "actions": ["test"]
  }

  202 ACCEPTED
  {
    "workflow": {
      "app_id": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
      "wf_id": 34,
      "created": <datetime>,
      "updated": <datetime>,
      "status": {
        "test": "QUEUED"
      },
      "source": {
        "repository": "http://github.com/fakeuser/fakedjangoapp.git",
        "revision": "master"
      },
      "config": {
        "test_cmd": "tox -epep8 -epy27",
        "run_cmd": "bin/start-app.py",
        "build_id": 34,
      },
      "actions": [
        "test"
      ],
      "logs": [],
      "artifacts": []
    }
  }

Test and build app::

  POST /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e/workflows
  {
    "actions": ["test", "build"]
  }

  202 ACCEPTED
  {
    "workflow": {
      "app_id": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
      "wf_id": 35,
      "created": <datetime>,
      "updated": <datetime>,
      "status": {
        "test": "QUEUED",
        "build": "QUEUED"
      },
      "source": {
        "repository": "http://github.com/fakeuser/fakedjangoapp.git",
        "revision": "master"
      },
      "config": {
        "test_cmd": "tox -epep8 -epy27",
        "run_cmd": "bin/start-app.py",
        "build_id": 35,
      },
      "actions": [
        "test",
        "build"
      ],
      "logs": [],
      "artifacts": []
    }
  }

Skip tests, just build app::

  POST /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e/workflows
  {
    "actions": ["build"]
  }

  202 ACCEPTED
  {
    "workflow": {
      "app_id": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
      "wf_id": 36,
      "created": <datetime>,
      "updated": <datetime>,
      "status": {
        "build": "QUEUED"
      },
      "source": {
        "repository": "http://github.com/fakeuser/fakedjangoapp.git",
        "revision": "master"
      },
      "config": {
        "test_cmd": "tox -epep8 -epy27",
        "run_cmd": "bin/start-app.py",
        "build_id": 36,
      },
      "actions": [
        "build"
      ],
      "logs": [],
      "artifacts": []
    }
  }

Deploy app with last good build

    Since wf_id starts at 1, a build_id 0 is a sentinel to mean "the last good
    build at deploy time". This is not updated until the deploy action is started,
    at which point the workflow's build_id will be updated to the appropriate value,
    or else the workflow will be marked as ERROR, with the reason "No such build".::

      POST /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e/workflows
      {
        "actions": ["deploy"],
        "config": {
          "build_id": 0
        }
      }

      202 ACCEPTED
      {
        "workflow": {
          "app_id": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
          "wf_id": 37,
          "created": <datetime>,
          "updated": <datetime>,
          "status": {
            "deploy": "QUEUED"
          },
          "source": {
            "repository": "http://github.com/fakeuser/fakedjangoapp.git",
            "revision": "master"
          },
          "config": {
            "test_cmd": "tox -epep8 -epy27",
            "run_cmd": "bin/start-app.py",
            "build_id": 0,
          },
          "actions": [
            "deploy"
          ],
          "logs": [],
          "artifacts": []
        }
      }

      ...

      GET /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e/workflows/37

      200 OK
      {
        "workflow": {
          "app_id": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
          "wf_id": 37,
          "created": <datetime>,
          "updated": <datetime>,
          "status": {
            "deploy": "IN PROGRESS"
          },
          "source": {
            "repository": "http://github.com/fakeuser/fakedjangoapp.git",
            "revision": "master"
          },
          "config": {
            "test_cmd": "tox -epep8 -epy27",
            "run_cmd": "bin/start-app.py",
            "build_id": 34,
          },
          "actions": [
            "deploy"
          ],
          "logs": [],
          "artifacts": []
        }
      }

Deploy app with specific build

  If there's no artifact from that build_id (the build failed), this request
  will fail. If it is a build, and isn't QUEUED, IN PROGRESS, or SUCCESS, this
  request will fail. ::

    POST /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e/workflows
    {
      "actions": ["deploy"],
      "config": {
        "build_id": 33
      }
    }

    202 ACCEPTED
    {
      "workflow": {
        "app_id": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
        "wf_id": 38,
        "created": <datetime>,
        "updated": <datetime>,
        "status": {
          "deploy": "QUEUED"
        },
        "source": {
          "repository": "http://github.com/fakeuser/fakedjangoapp.git",
          "revision": "master"
        },
        "config": {
          "test_cmd": "tox -epep8 -epy27",
          "run_cmd": "bin/start-app.py",
          "build_id": 33,
        },
        "actions": [
          "deploy"
        ],
        "logs": [],
        "artifacts": []
      }
    }

Stop running app

  Halt and destroy a container running user code. Starting another app is a
  matter of deploying the same build again, which will create a new assembly
  in the process. ::


    POST /apps/94cb7b89-0de8-492b-bf54-05ae96c9bd0e/workflows
    {
      "actions": ["stop"]
    }


    202 ACCEPTED
    {
      "workflow": {
        "app_id": "94cb7b89-0de8-492b-bf54-05ae96c9bd0e",
        "wf_id": 39,
        "created": <datetime>,
        "updated": <datetime>,
        "status": {
          "stop": "QUEUED",
        },
        "source": {
          "repository": "http://github.com/fakeuser/fakedjangoapp.git",
          "revision": "master"
        },
        "config": {
          "test_cmd": "tox -epep8 -epy27",
          "run_cmd": "bin/start-app.py",
          "build_id": 39,
        },
        "actions": [
          "stop"
        ],
        "logs": [],
        "artifacts": []
      }
    }

**History, Log, and Artifact commands**

As workflows progress through their actions, a record of the changes made to
an app are recorded in the history table. In addition, any logs or artifacts
produced during the course of these actions will be stored with this record.
These three resources are managed by the engine with the same table, but several
resources are exposed via the API to facilitate their retrieval.

These commands in particular benefit from filtering and pagination.

Show change history of one app::

  GET /apps/2797a1f4-fc03-4c21-9dde-099cf7636ceb/history

Show recent history of one app::

  GET /apps/2797a1f4-fc03-4c21-9dde-099cf7636ceb/history?limit=5

List all logs for one app::

  GET /apps/2797a1f4-fc03-4c21-9dde-099cf7636ceb/logs

Fetch logs for last failed test action of one app::

  GET /apps/2797a1f4-fc03-4c21-9dde-099cf7636ceb/logs?action=test&status=FAILED&limit=1

List all artifacts for one app::

  GET /apps/2797a1f4-fc03-4c21-9dde-099cf7636ceb/artifacts


Security impact
---------------

(to be determined)


Notifications impact
--------------------

(to be determined)


Other end user impact
---------------------

At minimum, python-solumclient will be drastically simplified. At present, it
already presents app commands that manipulate primarily plan and assembly
resources. By implementing these features in the API, the playing field is much
more level should someone want to interact with Solum without using the
official CLI.

The planfile format will also change dramatically. I seek to remove the
confusing multiple-artifact section, and its free-form content section.
Instead, to reflect the new fields in the planfile, it might be a simple as: ::

  name: fooweb
  description: my fooweb app
  languagepack: python27
  ports: [80]
  source:
  - repository: https://github.com/fakeuser/fooweb.git
  workflow_config:
  - test_cmd: tox -epep8 -epy27
    run_cmd: /app/run.sh /app/app.ini


Performance Impact
------------------

None


Other deployer impact
---------------------

None


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <ed--cranford>


Work Items
----------

- Create app resource
- Replace use of plan resources in code with app
- Create workflow resource
- Replace use of assembly resource in code with workflow
- Create history resource
- Add code to add rows to history table with every workflow update
- Add CLI commands for (new) app, workflows, and history resources
- Remove assembly, plan, component, pipeline commands from CLI
- Remove plan and assembly resources from API


Dependencies
============

None


Testing
=======

A drastic modification of the models and resources in Solum will of course
require extensive changes to both unit and tempest tests. Ideally, the tests
will be made simpler, as a lot of metadata should be handled by the API and not
by waitloops and client-side aggregation and filtering of API responses.


Documentation Impact
====================

Significantly less effort will be spent on explaining assemblies and plans and
their relationship to an application--arguably one of the most confusing ideas
in Solum at present is that it is a tool for managing application lifecycles
and yet has no application resource to speak of.


References
==========

None

.. # vim: set sw=2 ts=2 sts=2 tw=79:
