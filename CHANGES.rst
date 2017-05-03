=======
CHANGES
=======

2.1.0 (2017-05-03)
------------------

- Add support for Python 3.4, 3.5, 3.6 and PyPy.

- Remove test dependency on ``zope.app.testing`` and
  ``zope.app.zcmlfiles``, among others.


2.0.0 (2012-11-17)
------------------

- zope.configuration changed action tuples to action dicts. This version works
  with the new action dict given from zope.configuration since version 3.8.0.
  This version is not compatible with zope.configuration version less then
  3.8.0


1.3.0 (2010-10-28)
------------------

- Fundamental change in the way how baseregistry hooks into ZCA.
  Now it uses hooks.setSite, which requires that zope.component hooks
  are in place. Usually they are installed by zope.app.appsetup.
  Unless you use zope.app.appsetup, install the hooks with
  zope.component.hooks.setHooks().
  This applies to zope.component versions >= 3.9.4.


1.2.0 (2009-12-27)
------------------

- Moved browser dependencies to zmi extras


1.1.0 (2009-03-19)
------------------

- Fix base registry management form failure in case, when a site has its
  parent's local site manager (that isn't registered as utility) in its
  __bases__.

- Use zope.site instead of zope.app.component.

- Drop unused dependencies on zope.app.i18n and zope.app.pagetemplate.


1.0.0 (2008-01-24)
------------------

- Initial Release
