CakePHP 2.0.6 released
======================

by lorenzo on February 06, 2012

This will hopefully be the last 2.0 release, the CakePHP will focus on
getting 2.1 done and all bugfixes and security patches will go into
this branch.

The CakePHP core team is proud to announce the immediate availability
of CakePHP 2.0.6 [1]. This will hopefully be the last minor version
released for the 2.0 branch if no important changes are needed to be
done before we get version 2.1 stable. After this one all bugfixes and
security patches will be done in the 2.1 branch, which is 100%
compatible with this one.

In total, there were over 90 commits and 45 issues have been resolved
since 2.0.5. A complete list of the changes can be viewed on the
changelogs page [2], but a quick summary of changes that made it into
2.0.6:

+ Updating minimum PHP version requirement to 5.2.8 due to PHP bugs
  #44251 and #45748
+ Model does not call calculate() and expression() method on the
  datasource if it does not implements it.
+ SqlServer views can be mapped with models as you can do with normal
  tables.
+ Fixed issue where session would always be regenerated regardeless of
  configs set.
+ Not requiring a database connection for Models having $useTable =
  false when combined with the FormHelper
+ Not throwing errors when instantiating model classes with no
  constructor using the ClassRegistry
+ Fixing memory leaks in some Database drivers
+ TextHelper::excerpt() now works as documented.
+ Defaulting back to using PHP_SELF to get current url, this solves
  several issues in shared hosts

We are almost done with the 2.1 release and you can expect a stable
release within a couple of weeks. 2.1 will be fully backwards
compatible with 2.0, and add a number of useful features. For a list
of what is complete, and what is planned for 2.1, see the roadmap [3]
and the in-progress 2.1 migration guide [4]

Thanks once again to the excellent CakePHP community, for all of the
outstanding work. Without your contributions and love there would be
no CakePHP.

+ Download 2.0.6 [1] View the changelog [2]



Links
~~~~~

+ [1] `http://github.com/cakephp/cakephp/zipball/2.0.6`_
+ [2] `http://cakephp.org/changelogs/2.0.6`_
+ [3] `http://cakephp.lighthouseapp.com/projects/42648/development-
  roadmap`_
+ [4] `http://book.cakephp.org/2.0/en/appendices/2-1-migration-
  guide.html`_




.. _http://cakephp.lighthouseapp.com/projects/42648/development-roadmap: http://cakephp.lighthouseapp.com/projects/42648/development-roadmap
.. _http://book.cakephp.org/2.0/en/appendices/2-1-migration-guide.html: http://book.cakephp.org/2.0/en/appendices/2-1-migration-guide.html
.. _http://cakephp.org/changelogs/2.0.6: http://cakephp.org/changelogs/2.0.6
.. _http://github.com/cakephp/cakephp/zipball/2.0.6: http://github.com/cakephp/cakephp/zipball/2.0.6
.. meta::
    :title: CakePHP 2.0.6 released
    :description: CakePHP Article related to release,2.0.6,News
    :keywords: release,2.0.6,News
    :copyright: Copyright 2012 lorenzo
    :category: news

