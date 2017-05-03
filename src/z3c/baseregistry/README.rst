===============
Base Components
===============

The purpose of this package is to define, populate and use multiple
``IComponents`` instances using filesystem-based development -- in other
words, Python code and ZCML.


Motivation
----------

The current state of the component architecture allows us to

1.  create a global components registry, populate it using ZCML, and use it
    via the ``zope.component`` API functions.

2.  define local sites (local components registries), populate them with local
    (persistent) components, and use them selectively based on location --
    commonly defined by the path of the URL.

Unfortunately, it is impossible to populate local sites with ZCML. The main
reason is the lack of addressability of local sites during the initial startup
process.

However, on the other hand we have a very advanced UI configuration system
that involves views, resources, layers and skins. So let's compare the two.

1.  Views/Resources in the UI are like registered components in the component
    architecture.

2.  Skin Layers in the UI behave very much like registries. The default skin
    is like the global base registry. Skins, like local sites, are activated
    during traversal, but can be populated using ZCML.

3.  Layers are really base layers to the skin layer. The equivalent in the
    component architecture is to specify bases for a components registry,
    which is possible since the Great Component Architecture refactoring for
    Zope 3.3 in 2006.

But layers can be defined and configured via ZCML. The purpose of this package
is to be able to create base components registries and then populate them
using ZCML. (As a side note: As skin layers and layers are practically the
same components, there is no difference between the concept of global, local
and base components registries.)

The second feature is specific to the Zope application server. It provides an
UI to set the bases on a local site manager. The user can select among all
registries that have been registered as ``IComponents`` utilities.

There are also a few options that could be considered in the future. For
example, it would be simple to integrate the ``zope:registerIn`` directive
(see below for documentation) into the ``zope:configure`` directive.

If the above text is too dry and theoretical for you, here is the
summary. This package

1.  implements Steve Alexander's long dream (at least 3 years) of defining
    local sites via ZCML.

2.  solves all of my (Stephan Richter) problems I am having with a complex
    Application Service Provider (ASP) setup.

3.  implements a missing feature that you and everyone else really wanted,
    even if you did not know it yet.

Thanks goes to Jim Fulton, whose outstanding design of the
``zope.configuration`` and ``zope.component`` packages made the implementation
of the feature such a breeze. I also want to thank Fred Drake for helping with
the initial design ideas.


"Base Components" Registries
----------------------------

Base registries are global component registries implementing the
``IComponents`` interface. In comparison to the base global registry (also
known as ``globalSiteManager``), these registries are not necessarily
available via module globals and *must* be registered with a parent registry,
most commonly the base global registry:

  >>> from z3c.baseregistry import baseregistry
  >>> import zope.component
  >>> myRegistry = baseregistry.BaseComponents(
  ...     zope.component.globalSiteManager, 'myRegistry')

  >>> myRegistry
  <BaseComponents myRegistry>

Another *VERY IMPORTANT* requirement is that ``zope.component`` hooks are in
place. Install the hooks now:

  >>> import zope.component.hooks
  >>> zope.component.hooks.setHooks()


Since this registry does not implement any of the ``IComponents`` API itself,
it is not necessary to demonstrate those features here. Please see the
corresponding documentation in the ``zope.component`` package.

One feature of global registries must be that they pickle efficiently, since
they can be referenced in persisted objects. As you can see, the base registry
pickles quite well:

  >>> import pickle
  >>> jar = pickle.dumps(myRegistry, 2)
  >>> len(jar) <= 100
  True

However, when reading the jar, we get an error:

  >>> pickle.loads(jar)
  Traceback (most recent call last):
  ...
  zope.interface.interfaces.ComponentLookupError: (<InterfaceClass zope.interface.interfaces.IComponents>, 'myRegistry')

This is because we have not registered the registry in its parent as an
``IComponents`` utility, yet:

  >>> from zope.component.interfaces import IComponents
  >>> zope.component.provideUtility(myRegistry, IComponents, 'myRegistry')

  >>> pickle.loads(jar)
  <BaseComponents myRegistry>

Thus it is very important that you *always* register your base registry with
its parent!

Like any other components registry, a base registry can also have bases:

  >>> myOtherRegistry = baseregistry.BaseComponents(
  ...     zope.component.globalSiteManager, 'myRegistry', (myRegistry,))
  >>> myOtherRegistry.__bases__
  (<BaseComponents myRegistry>,)

Let's now have a look at how base registries can be defined and used
via ZCML, which is the usual mode of operation.


Defining Base Registries
------------------------

The above tasks are more commonly done in ZCML. Base components registries --
or any ``IComponents`` implementation for that matter -- can be seen as
utilities providing the aforementioned interface and are distinguishable by
name. So let's define a "custom" registry:

  >>> custom = baseregistry.BaseComponents(
  ...     zope.component.globalSiteManager, 'custom')

Let's make sure that the parent of the custom registry is the base registry:

  >>> custom.__parent__
  <BaseGlobalComponents base>

The registry is then registered using the standard utility directive. After
loading the meta directives for this package,

  >>> from zope.configuration import xmlconfig
  >>> from zope.configuration.config import ConfigurationConflictError
  >>> context = xmlconfig.string('''
  ... <configure i18n_domain="zope">
  ...   <include package="z3c.baseregistry" file="meta.zcml" />
  ...   <include package="zope.component" file="meta.zcml" />
  ... </configure>
  ... ''')

we can register the registry:

  >>> context = xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <utility
  ...       component="README.custom"
  ...       provides="zope.component.interfaces.IComponents"
  ...       name="custom" />
  ...
  ... </configure>
  ... ''', context=context)

The new registry can now be accessed as follows:

  >>> custom = zope.component.getUtility(IComponents, name='custom')
  >>> custom
  <BaseComponents custom>


Populating Different Registries
-------------------------------

Now to the interesting part. Let's register components for both the global
base and the "custom" registry. Let's first create some utilities we can
register:

  >>> import zope.interface

  >>> class IExample(zope.interface.Interface):
  ...     name = zope.interface.Attribute('Name of Example')

  >>> @zope.interface.implementer(IExample)
  ... class Example(object):
  ...     def __init__(self, name):
  ...         self.name = name
  ...     def __repr__(self):
  ...         return '<%s %r>' %(self.__class__.__name__, self.name)

  >>> example1 = Example('example1')
  >>> example2 = Example('example2')

Create some adapters we can register:

  >>> class IToAdapt1(zope.interface.Interface):
  ...     pass

  >>> class IToAdapt2(zope.interface.Interface):
  ...     pass

  >>> class IAdapted(zope.interface.Interface):
  ...     pass

  >>> @zope.component.adapter(IToAdapt1)
  ... @zope.interface.implementer(IAdapted)
  ... def adapter1(context):
  ...     return "adapted1"

  >>> @zope.component.adapter(IToAdapt2)
  ... @zope.interface.implementer(IAdapted)
  ... def adapter2(context):
  ...     return "adapted2"

  >>> @zope.interface.implementer(IToAdapt1)
  ... class ToAdapt1(object):
  ...     def __init__(self, name):
  ...         self.name = name
  ...     def __repr__(self):
  ...         return '<%s %r>' %(self.__class__.__name__, self.name)
  >>> toAdapt1 = ToAdapt1('toAdapt1')

  >>> @zope.interface.implementer(IToAdapt2)
  ... class ToAdapt2(object):
  ...     def __init__(self, name):
  ...         self.name = name
  ...     def __repr__(self):
  ...         return '<%s %r>' %(self.__class__.__name__, self.name)
  >>> toAdapt2 = ToAdapt2('toAdapt2')

Let' now register "example1", adapter1 in the global registry
and "example2", "adapter2" in our custom registry:

  >>> context = xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <utility component="README.example1"
  ...            name="example1" />
  ...   <adapter
  ...         factory="README.adapter1"
  ...         name="adapter1"/>
  ...
  ...   <registerIn registry="README.custom">
  ...     <utility component="README.example2"
  ...              name="example2" />
  ...     <adapter
  ...         factory="README.adapter2"
  ...         name="adapter2"/>
  ...   </registerIn>
  ...
  ... </configure>
  ... ''', context=context)

Let's now make sure that the utilities have been registered in the right
registry:

  >>> zope.component.getUtility(IExample, name="example1")
  <Example 'example1'>

  >>> zope.component.getUtility(IExample, name="example2")
  Traceback (most recent call last):
  ...
  zope.interface.interfaces.ComponentLookupError: (<InterfaceClass README.IExample>, 'example2')

Let's now make sure that the adapters have been registered in the right
registry:

  >>> zope.component.getAdapter(toAdapt1, IAdapted, name="adapter1")
  'adapted1'

  >>> zope.component.getAdapter(toAdapt2, IAdapted, name="adapter2")
  Traceback (most recent call last):
  ...
  zope.interface.interfaces.ComponentLookupError: (<ToAdapt2 'toAdapt2'>, <InterfaceClass README.IAdapted>, 'adapter2')


  >>> custom = zope.component.getUtility(IComponents, name='custom')

  >>> custom.getUtility(IExample, name="example1")
  Traceback (most recent call last):
  ...
  zope.interface.interfaces.ComponentLookupError: (<InterfaceClass README.IExample>, 'example1')

  >>> custom.getUtility(IExample, name="example2")
  <Example 'example2'>


  >>> custom.getAdapter(toAdapt1, IAdapted, name="adapter1")
  Traceback (most recent call last):
  ...
  zope.interface.interfaces.ComponentLookupError: (<ToAdapt1 'toAdapt1'>, <InterfaceClass README.IAdapted>, 'adapter1')

  >>> custom.getAdapter(toAdapt2, IAdapted, name="adapter2")
  'adapted2'


Let's now register other instances of the ``Example`` class without a
name. This should *not* cause a conflict error:

  >>> example3 = Example('example3')
  >>> example4 = Example('example4')

  >>> context = xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <utility component="README.example3" />
  ...
  ...   <registerIn registry="README.custom">
  ...     <utility component="README.example4" />
  ...   </registerIn>
  ...
  ... </configure>
  ... ''', context=context)

  >>> zope.component.getUtility(IExample)
  <Example 'example3'>

  >>> custom.getUtility(IExample)
  <Example 'example4'>


Using Base Registries
---------------------

Most commonly base registries will be used in local site managers. So let's
create a local site:

  >>> from zope.site.folder import Folder
  >>> site = Folder()

  >>> from zope.site.site import LocalSiteManager
  >>> site.setSiteManager(LocalSiteManager(site))
  >>> sm = site.getSiteManager()

Initially only the base global registry is a base of the local site manager:

  >>> sm.__bases__
  (<BaseGlobalComponents base>,)

Now only registrations from the base site are available:

  >>> sm.getUtility(IExample)
  <Example 'example3'>

  >>> sm.getUtility(IExample, name="example1")
  <Example 'example1'>

  >>> sm.getUtility(IExample, name="example2")
  Traceback (most recent call last):
  ...
  zope.interface.interfaces.ComponentLookupError: (<InterfaceClass README.IExample>, 'example2')

  >>> sm.getAdapter(toAdapt1, IAdapted, name="adapter1")
  'adapted1'

  >>> sm.getAdapter(toAdapt2, IAdapted, name="adapter2")
  Traceback (most recent call last):
  ...
  zope.interface.interfaces.ComponentLookupError: (<ToAdapt2 'toAdapt2'>, <InterfaceClass README.IAdapted>, 'adapter2')

But if we add the "custom" registry, then things look more interesting:

  >>> sm.__bases__ += (custom,)
  >>> sm.__bases__
  (<BaseGlobalComponents base>, <BaseComponents custom>)

  >>> sm.getUtility(IExample)
  <Example 'example3'>

  >>> sm.getUtility(IExample, name="example1")
  <Example 'example1'>

  >>> sm.getUtility(IExample, name="example2")
  <Example 'example2'>

  >>> sm.getAdapter(toAdapt1, IAdapted, name="adapter1")
  'adapted1'

  >>> sm.getAdapter(toAdapt2, IAdapted, name="adapter2")
  'adapted2'

But where is the registration for example 4? Well, the order of the bases
matters, like the order of base classes in Python matters. The bases run from
must specific to most generic. Thus, if we reverse the order,

  >>> bases = list(sm.__bases__)
  >>> bases.reverse()
  >>> sm.__bases__ = bases
  >>> sm.__bases__
  (<BaseComponents custom>, <BaseGlobalComponents base>)

then our "custom" registry effectively overrides the global one:

  >>> sm.getUtility(IExample)
  <Example 'example4'>

  >>> sm.getUtility(IExample, name="example1")
  <Example 'example1'>

  >>> sm.getUtility(IExample, name="example2")
  <Example 'example2'>


Edge Cases and Food for Thought
-------------------------------

Duplicate Registrations
~~~~~~~~~~~~~~~~~~~~~~~

Like before, duplicate registrations are detected and reported:

  >>> try:
  ...    xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <registerIn registry="README.custom">
  ...     <utility component="README.example3" name="default" />
  ...     <utility component="README.example4" name="default" />
  ...   </registerIn>
  ...
  ... </configure>
  ... ''', context=context)
  ... except ConfigurationConflictError as e:
  ...    print(e)
  Conflicting configuration actions
    For: (<BaseComponents custom>, ('utility', <InterfaceClass README.IExample>, ...'default'))
  ...

But as we have seen before, no duplication error is raised, if the same
registration is made for different sites:

  >>> context = xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <utility component="README.example3" name="default" />
  ...
  ...   <registerIn registry="README.custom">
  ...     <utility component="README.example4" name="default" />
  ...   </registerIn>
  ...
  ... </configure>
  ... ''', context=context)


Overriding ZCML
~~~~~~~~~~~~~~~

Overriding should behave as usual. If I define something within a particular
site, then it should be only overridable in that site.

In the following example, ``base-overrides.zcml`` overrides only the global
registration of the following snippet to "example3":

  >>> context.includepath = ('base.zcml', 'original.zcml')
  >>> context = xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <utility component="README.example1" />
  ...
  ...   <registerIn registry="README.custom">
  ...     <utility component="README.example2" />
  ...   </registerIn>
  ...
  ... </configure>
  ... ''', context=context, execute=False)

  >>> context.includepath = ('base.zcml',)
  >>> context = xmlconfig.string('''
  ...   <includeOverrides package="z3c.baseregistry.tests"
  ...                     file="base-overrides.zcml" />
  ... ''', context=context)

  >>> zope.component.getUtility(IExample)
  <Example 'example3'>

  >>> custom.getUtility(IExample)
  <Example 'example2'>

In the next example, ``custom-overrides.zcml`` overrides only the custom
registration of the following snippet to "example3":

  >>> context.includepath = ('base.zcml', 'original.zcml')
  >>> context = xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <utility component="README.example1" />
  ...
  ...   <registerIn registry="README.custom">
  ...     <utility component="README.example4" />
  ...   </registerIn>
  ...
  ... </configure>
  ... ''', context=context, execute=False)

  >>> context.includepath = ('base.zcml',)
  >>> context = xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <includeOverrides package="z3c.baseregistry.tests"
  ...                     file="custom-overrides.zcml" />
  ...
  ... </configure>
  ... ''', context=context)

  >>> zope.component.getUtility(IExample)
  <Example 'example1'>

  >>> custom.getUtility(IExample)
  <Example 'example3'>

Note: Sorry for the convoluted test sequence; this is just how it works. :-(


Nested Registry Usage
~~~~~~~~~~~~~~~~~~~~~

I thought about this one for a long time, but I think it is better not
allowing to nest ``zope:registerIn`` directives, because the logic of
manipulating the discriminator would be very complex for very little added
benefit.

  >>> try:
  ...    xmlconfig.string('''
  ... <configure xmlns="http://namespaces.zope.org/zope" i18n_domain="zope">
  ...
  ...   <registerIn registry="README.custom">
  ...     <registerIn registry="zope.component.globalregistry.base">
  ...       <utility component="README.example4" />
  ...     </registerIn>
  ...   </registerIn>
  ...
  ... </configure>
  ... ''', context=context)
  ... except Exception as e:
  ...     print(e)
  File...
      ConfigurationError: Nested ``registerIn`` directives are not permitted.

Cleanup
~~~~~~~

Just unregister the ``zope.component`` hooks:

  >>> zope.component.hooks.resetHooks()


Global Non-Component-Registration Actions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ZCML is not only responsible for populating the components registries, but also
to do other global configuration, such as defining security and assigning
interfaces to classes. On the other hand, the ``registerIn`` directive works
by manipulating the discriminator by prefixing it with the current
registry. While I assert that this is the right approach for component
registrations, it does not work for those other global configurations.

In order to address the issue, I need somehow more information. A balance must
be struck between the need to change existing directives and making the
solution non-monolithic. Here are some design ideas:

1. A Special Discriminator Prefix

   All directives that globally manipulate the state of the system and do not
   register a component have as their first discriminator entry a special
   string, like "StateChange". The directive can then look for those entries and
   not change the discriminator at this point.

   Advantages include the ability to use those directives inside the
   ``registerIn`` directive and allow gradual upgrading. In the other hand, util
   directives are adjusted, conflict resolution will not be available for those
   scenarios.

2. A Registry of Global Action Callables

   Here this package provides a registry of callables that change the state of
   the system. Directive authors can then subscribe their callables to this
   registry.

   The big advantage of this approach is that you can make it work now for all
   built-in directives without changing any implementation. The disadvantage is
   that the solution hides the problem to directive authors, so that detailed
   documentation must be provided to ensure integrity and avoid
   surprises. Another disadvantage is the complexity of yet another registry.

3. Autodetection with False-Positives

   As far as I can tell, all actions that manipulate the components registries
   use the ``zope.component.zcml.handler`` function. Okay, so that allows me to
   detect those. Unfortunately, there might be directives that do *not*
   manipulate the state, for example ensuring the existence of something. There
   are a bunch of those directives in the core.

   The advantage here is that for the core it should just work. However, 3rd
   party directive developers might be tripped by this feature. Also, we could
   only issue warnings with this solution and probably need to be able to turn
   them off.

I have not implemented any of those suggestions, waiting for input from the
community.
