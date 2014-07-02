High-level Description:
------------------------------------
This spec outlines approaches towards creating a Chef language pack.
A recommendation is made towards one particular approach.


Problem:
----------------
Within Solum we want to provide ability for application developers to test
their chef cookbooks. Testing chef cookbooks broadly involves two kinds of
tests. The first kind of tests are concerned with things such as
ensuring that the cookbook code is
syntactically correct, stylistically correct, and the recipes are
unit tested to ensure that the mutual
inter-dependencies are working fine. The tools available for these tasks are,
Knife for ruby syntax, foodcritic for linting
(https://github.com/acrmp/foodcritic) and
ChefSpec for unit testing (https://github.com/sethvargo/chefspec).
The second kind of tests are concerned with testing whether the configuration
which resulted from running a set of recipes has indeed converged to the
intended state. The tool available for this is test
kitchen (https://github.com/test-kitchen/test-kitchen).

In Solum we want to support application developers who are developing chef
recipes to use above tools for testing their recipes.


Proposed Solution:
---------------------
We propose to provide a Chef language pack (a image) in
Solum with chef, foodcritic,
ChefSpec, and test kitchen installed on it.
Solum operator would create this language pack and register it within
Solum's glance.
Developers would use this language pack id within their plan definition.


Implementation options:
------------------------
There are at least four different ways one can build such a Chef
language pack.

1) We could use disk-image-builder and write a 'Chef element' which
provides a script with the installation instructions.

2) We could customize Heroku's Ruby buildpack
(https://github.com/heroku/heroku-buildpack-ruby) to
install the required tools.

3) We could provide a Dockerfile to create a container image and use that
as the Chef language pack.


Option 1 has problems associated with it. For details
see:

https://review.openstack.org/#/c/103689/

I propose that we rule out option 2. The reasons are as follows.
We could customize Heroku's buildpack in two ways. One, we maintain a
fork of the buildpack repo.

This approach, even though possible, is lot of work.
Second approach would be to maintain just the customization bits and then
'apply' them
to the 'lp-cedarish's heroku-buildpack-ruby, which we clone and 'install'
within the lp-cedarish flow.
The customization bits would essentially be installation logic written
in a '.rb' file which
would need to be 'added' to the 'spec' directory of the heroku-buildpack-ruby.
The problem with this approach is that the creation of the language pack
gets tied with Heroku's Ruby buildpack.

So that leaves option 3. I propose we start there.
(This is the recommended option in https://review.openstack.org/#/c/103689/)


First cut of the Chef language pack is available here:

https://review.openstack.org/#/c/103671/



Design Issues:
-----------------------
1) How to pass custom style rules to foodcritic? Once this is possible
we would invoke foodcritic like so:

 foodcritic cookbooks --include foodcritic-rules.rb --tags ~FC001

2) How to pass location of cookbooks on the running VM to the various
commands. One option is that the developer will provide the exact command(s)
for invoking the test. These will be provided in the Plan file. For example,
lets say there is a hello_world cookbook
in my git repository. Then in the plan file I may specify commands
in the following manner:

{
   syntax_check: knife cookbook test /hello_world

   style_check: foodcritic /hello_world

   unit_test: rspec /hello_world --format RspecJunitFormatter

   integration_test: kitchen test /hello_world

}

References:
-----------

https://blueprints.launchpad.net/solum/+spec/chef-language-pack
