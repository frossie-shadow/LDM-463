:tocdepth: 1

.. sectnum::

.. _intro:

Introduction
============

.. _change-record:

There are two primary methods of data access. The first, which appears
primarily as a library to the user, is the Butler. The second is a set of HTTP
interfaces collectively known as the DAX APIs or loosely referred to as
webserv. There may be some overlap between the two of these; in some cases,
the Butler may call out to the DAX APIs or the DAX APIs may use the Butler
underneath, and we plan to have a DAX API which operates as a Butler bridge
service. This last service will serve primarily as a proxy for remote
applications which may be coded in any language.

Butler
======

Overview - What is the Data Butler?
-----------------------------------

The Butler is a framework for generic I/O and data management. It isolates
application code from the underlying data-access implementation in terms of
storage formats, physical locations, data staging, database mapping, etc.

Butler stores data in and retrieves data from Repositories. Each repository has
a mapper that is used to find data in the repository. The mapper is configured
by a Policy that provides configuration details.

This general concept makes the Data Butler potentially able to serve as a data
*router* (or hub or switch). Data can be published and sent to multiple
locations based on the Butler configuration. Those locations may include
"persistent storages" that are actually dynamic (like displays or subscription
streams) rather than truly persistent storage.

'Old' vs. 'New'  Butler
-----------------------

'Old' Butler mechanisms and their 'new' Butler counterparts are discussed
briefly here.

A few things to know about 'new' Butler mechanisms
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

'Old' butler managed a single i/o repository. It could have parents via a
``parent`` symlink. The ``CameraMapper`` allowed a separate ``calib``
repository, and a separate output repository specified by ``outputRoot``. 'New'
butler manages multiple repositories that can be specified as ``inputs``
(read-only) and ``outputs`` which may be write-only or read-write. The behavior
of 'new' Butler repositories is discussed throughout this document.

Repository configuration (cfg) parameters are stored in a yaml file in the
repository called ``repositoryCfg.yaml``. This cfg replaces the ``_parent``
symlink and ``_mapper`` file, and includes other parameters.

'New' butler repositories can have multiple parent repositories. References to
parents are stored in the repository cfg.

'Old' Butler mechanisms and their 'new' Butler counterparts
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``_mapper``
    Instead of naming a mapper in a single-purpose text file, the mapper is
    embedded in the repository cfg file.

``_parent``
    Instead of a symlink to another repository on the local disc, the repository
    cfg has a list of URIs. Each points to the parent repository's cfg file.

``CameraMapper``
    The current CameraMapper makes many calls to the local file system directly,
    and uses the ``_parent`` mechanism to refer to parent repositories. This is
    currently the case, but very soon will be refactored; each repository (parents
    and children) that use the CameraMapper will be managed by a separate Repository
    class in Butler. Searching parents will be executed by Butler.

CameraMapper no longer Accesses Parents Directly
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In New Butler the CameraMapper does not access parents directly (via _parent
symlinks). In the cases where child repositories need to access elements of
parent repositories, those elements are passed to the children:

* Parent Registry: when a Repository does not have its own sqlite Registry, it
  may use the sqlite Registry of its parent to perform lookups. To provide this,
  the Butler searches the Repository's parent Repositories (depth first) until a
  Repository with an sqlite Registry is found, and that Registry is passed to
  the Repository and to its mapper as the "parent Registry". Right now the rule
  is very simplistic (simply search for and return the first-found sqlite
  registry). We could restrict the search so that some registries are passed
  over if they don't match the child Repository. I think it would be worth
  waiting and designing this with the output registry feature we've discussed.
* Parent mapper: new output repositories used to discover their mapper by using
  the same mapper as the input Repository (via the `outputRoot` mechanism). Now,
  a "default mapper" is discovered by inspecting the input repositories. If all
  the input repositories use the same mapper then a default mapper is inferred
  and new output repositories will use that same mapper. If input repositories
  use different mappers then a default mapper can not be established and new
  Repositories must specify a mapper with their RepositoryArgs. We have discussed
  that some mappers are nonsensical as candidates for default mapper, no fitlering
  of mappers is implemented yet.
* Mapper init: To allow Butler to set parameters that are used in
  ``CameraMapper.__init__`` (e.g. the mapper's registry, which may be a parent's
  registry), mappers should not be initialized when passed into Butler. The
  mapper (passed in via a dict or a RepositoryArgs instance) should be a Class
  object or an importable string.

ButlerLocation Requires Storage Interface
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A mapper may manage more than one Storage location (e.g. ``CameraMapper``'s
``calibRoot`` can point at a second Storage location). Because of this,
ButlerLocation now requires a Storage class instance that is the storage to
which the location refers.

A ``ButlerLocation`` contains a location (path) within a Storage Interface. The
Storage Interface keeps track of the root, so the location needs only the path
from root. As a result, ``ButlerLocaiton.getLocations()`` returns the location
without the root.

To get the location prepeneded with root (root+location), use
``ButlerLocation.getLocationsWithRoot()``.

When initializing a ``ButlerLocation`` (during a map), a Storage Interface
instance must be passed to ``ButlerLocation.__init__(...)``

Mappers Should Provide Access to their Registry
"""""""""""""""""""""""""""""""""""""""""""""""
Mappers should provide access to their registry via the method
``getRegistry(self)``. This allows mappers in child repositories to use the
registry from a parent.

Dataset
-------

A dataset is the persisted form of an in-memory object; it could be a single
item, a composite, or a collection. Examples: `int`/`long`, `PropertySet`,
`ExposureF`, WCS, PSF, `set`/`list`/`dict`.

Repository
----------

.. warning::

    This section describes New Butler Repositories, much of which should be
    considered internal-only, non-public API for the time being. See
    documentation in daf_persistence and daf_butlerUtils for current butler use
    and API.

A Repository refers to data storage. In persisted form (i.e. in on disk, in a
database, or in other storage) a repository contains any/all of:

- a collection of datasets
- the name of the mapper to use
- the policy that is used to configure the mapper
- a registry to expedite data lookup
- references to parent repositories.

When a Repository class is instantiated, it uses input configuration arguments
to locate or create a repository in storage. It then instantiates the Mapper and
Access classes as well as its parent & peer Repositories.

If the configuration arguments point to a repository that already exists then
the arguments must be consistent with the stored repository configuration. This
includes which mapper to use and what the parents of that repository are.
Butler checks for this, and if a mismatch is detected it will raise a
RuntimeError in ``Butler.__init__``.

Root
^^^^

Root is the 'top-level' location of a Repository and provides access to
within-repository items such as RepositoryCfg_, a Policy, and the Registry.
These provide further information about where actual Datasets can be found
(currently with in-filesystem repositories, the files are always stored under
the folder indicated by Root).

Root is passed to the Butler by URI. Currently the only supported storage back
end is the local filesystem. URI for this may be passed as a relative path
(``foo/bar``), an absolute path (``/foo/bar``), or explicitly with the ``file``
schema name (``file:///foo/bar``); only absolute paths are supported when naming
the file schema explicitly. In the future other schemas will be supported.


CfgRoot
^^^^^^^
If a RepositoryCfg_ is not located within the repository it can exist at a
"configuration root". It's ``root`` parameter must indicate where the repository
is.

Parents
^^^^^^^

Repositories may have zero or more parents that are other repositories. These
repositories are used by the butler as inputs. Repositories' parent
configuration are referenced by URI in the repository configuration.

When a repository is used as a Butler output, all of the butler's input and
writable output repositories are listed in that output repository's list of
parent repositories. (But not parents-of-parents etc; these are found again the
next time the parent is loaded).

Repository Version
^^^^^^^^^^^^^^^^^^
This feature is still being designed and developed under
`DM-4168 - Data repository selection based on version
<https://jira.lsstcorp.org/browse/DM-4168>`_.

The stated requirement is: Must support a system so that data may be referred to
by version at repository load time at the latest. (Won't be selectable by dataId
when calling ``Butler.get(...)``) .

Butler
------
The ``Butler`` class is the  overall interface and manager for repositories.
The Butler init function takes a list of input and output repositories (see
below for a description of inputs and outputs) that are used as locations for
i/o.

The new butler initializer API is ``Butler(inputs=None, outputs=None)``. Values
for both ``inputs`` and ``outputs`` can be an instance of the ``RepositoryArgs``
class, a ``dict`` that will be used to initialize a ``RepositoryArgs`` instance,
or can be a string that is treated as the root (a URI to a repository). In
inputs, root must refer to a location that contains a persisted
``RepositoryCfg`` and in outputs it can refer to an existing ``RepositoryCfg``
or can be a location to create a new repository. The value can be either a
single item or a sequence of one or more items.

Note that with the old Butler init API
(``Butler.__init__(root, mapper, **mapperArgs)``), the location indicated by
``root`` does not have to contain a ``RepositoryCfg``, and Butler will not
write one at that location. Internally Butler will create an in-memory only
``RepositoryCfg`` to use that Repository.

Inputs and Outputs
^^^^^^^^^^^^^^^^^^

Butler will only perform read actions on input repositories and will perform
read and write actions on output repositories. Repositories may also have a
mode that can be one of:

* read
* write
* read-write

When the mode of an output repository is read-write it will also be used as an
input. Attempting to pass a read-only repo as a butler output or a write-only
repo as a butler input will raise an exception.


Already existing Repositories as Outputs
""""""""""""""""""""""""""""""""""""""""

When initializing a ``Butler`` with one or more output repositories that already
exist it is important to not add any input repositories that are not already
parents of the output repository. This is because all butler inputs (including
readable outputs) become parents of output repositories, and the repository
configuration is not allowed to change after it has been written.

Output arguments derived from inputs
""""""""""""""""""""""""""""""""""""

Some settings for output repositories can be derived from input repository
configurations. For example, if an output configuration does not specify a
mapper, the input mapper may possibly be assumed (this will work as long as all
the input repositories use the same type of mapper; if the inputs use different
types of mapper then a single type mapper can not be inferred to use for the
output repositories). When possible the butler will use settings from input
configurations to complete RepositoryArgs parameters for output repositories.

Search Order
""""""""""""
The order of repositories passed to inputs and outputs is meaningful; search is
depth-first and in order (left to right). Readable outputs will be searched
before inputs. Parents of readable outputs/inputs will be searched before the
next passed-in output/input.

Tagging
^^^^^^^

Readable repositories can be “tagged” with an identifier that gets used when
reading from a repository. A tag can be specified in ``RepositoryArgs`` when
initializing a Butler. A repository can be tagged with more than one tag by
passing in a container of tags. The tag is not persisted with the repository.

When performing read operations on the butler, if the DataId contains one or
more tags, the repository will only be used for lookups if it is also tagged
with one of the tags in the DataId. If the DataId has no tags, then all input
repositories will be used. More information about DataId and its tag are
available in the DataId section.

RepositoryArgs
^^^^^^^^^^^^^^

``RepositoryArgs`` instances are used to instantiate repositories in Butler. Its
parameters are:

* ``mode``
    * Optional.
    * string - This can be one of 'r', 'w', or 'rw' (read, write, read-write).
    * It is used to indicate the read/write state of the repositories. Input
      repositories are always read-only and an exception will be raised if the
      mode of an input repository is 'w'. It may be 'rw' but for inputs the
      behavior will be the same as 'r'. Output repositories must be 'w' or 'rw'.
      If it is 'rw' the repository will also be used as an input repository. If
      mode is not specified, outputs will default to 'w' and inputs will default
      to 'w'.
* ``mapper``
    * Optional if the repository already exists - for inputs it's better to
      leave this parameter empty. For outputs it's optional if the mapper can be
      inferred from the input repositories and is otherwise required.
    * Can be an importable & instantiatable string (e.g.
      ``lsst.daf.persistence.CameraMapper``), an class object, or a class
      instance.
    * This specifies the mapper to be used by the repository.
* ``mapperArgs``
    * Optional
    * dict
    * These arguments are passed to the mapper when it is being instantiated (if
      it needs to be instantiated). If the mapper requires Root_ it does not
      need to be included in mapperArgs. When creating the mapper if Root_ is
      needed the butler will get Root_ from storage and use that.
* ``root`` and ``cfgRoot``
    * at least one is required.
    * string URI
    * ``root`` is a URI to where the repository or repositoryCfg is (if it
      exists already) or to where the repository should be (if it does not exist
      yet). If the RepositoryCfg is or should be stored separately from the
      repository then ``cfgRoot`` should be a URI to the persisted RepositoryCfg.
* ``policy``
    * Optional
    * dictionary (nested)
    * Represents policy to be used and saved with the repository configuration.
* ``tags``
    * Optional
    * Any tag type
    * Indicates the tags that a repository should be labeled with in the
      butler. (There is also a member function of ``RepositoryCfg`` to set
      tags on an instantiated cfg.)

If the repository already exists it is best to only to populate:

 * ``root`` (required, to find the repository cfg)
 * ``tags`` - if any are to be used.
 * ``mode`` - for output repositories that should be readable.

If ``mapper`` and/or ``mapperArgs`` are populated and the value in args does not
match the value of the persisted RepositoryCfg an exception will be raised.

Details about the repository configuration are persisted in the
``RepositoryCfg`` object when it is serialized. ``RepositoryArgs`` parameters
that do not appear in the ``RepositoryCfg`` are not persisted (``mode``,
``tags``).

RepositoryCfg
^^^^^^^^^^^^^

When a ``Repository`` is initialized by Butler its ``RepositoryCfg`` is persisted.
The ``RepositoryCfg`` is written only once and can not change. The ``RepositoryCfg``
parameters are:

* ``root``
    * Required (but may not appear in persisted RepositoryCfg). This field is
      populated in the persisted cfg in the case where the cfg is not stored in
      the repository. If the persisted cfg is at the root of the repository then
      the field is left blank.
    * string URI
    * This is a URI to the root location of the repository.
* ``mapper``
    * Required
    * Can be an importable & instantiatable string (e.g.
      ``lsst.daf.persistence.CameraMapper``), an class object, or a class
      instance.
    * This indicates the mapper to use with this repository
* ``mapperArgs``
    * Required
    * dict or None
    * These arguments are passed to the mapper when it is being instantiated (if
      it needs to be instantiated and the mapper parameter does not have the
      args packed into that value). If the mapper requires root it does not need
      to be included in mapperArgs. When creating the mapper if Root_ is needed
      the butler will get root from storage and use that.
* ``parents``
    * required
    * list or None
    * This is a list of URI to the ``RepositoryCfg`` of each parent.
* ``policy``
    * optional
    * dict (nested)
    * This is policy to be added to the rest of the policy loaded by the butler
      for this repository.

Persisted Parent Path is Relative When Possible
"""""""""""""""""""""""""""""""""""""""""""""""

When the Storage class can establish a relative path between the RepositoryCfg
root and a parent URI in the parents list, the URI in the parents list is stored
as a relative path. This makes it easier to move Repositories from one location
to another.

In the ``RepositoryCfg.parents`` property getter the relative paths are
converted to absolute paths. Everywhere else in the Butler framework absolute
paths are used so that repository identification is unambiguous.

``RepositoryCfg`` uses ``Storage.absolutePath(...)`` and
``Storage.relativePath(...)`` to try to get absolute and relative paths between
two URIs.


Moving Repositories and RepositoryCfgs
""""""""""""""""""""""""""""""""""""""

When copying a repository from one Storage type to another (e.g. from a
developer to a Swift location) it's possible the parent URIs will have to be
adjusted. When we add Storage locations this should be considered, and it's
possible we should write a helper script to support this.


Mapper
------

A Mapper is used by a Repository to find datasets (when reading) or
locations for datasets (when writing). the ``Mapper`` class must be subclassed
to implement meaningful behavior. The most commonly used Mapper subclass in LSST
is ``CameraMapper``.

Typically a Mapper instance is configured by the Policy.

Storage Layer
-------------

Storage is the abstraction layer that separates Repositories in persistant
storage from the parts of the framework that use the data in the Repository.

There is a ``Storage`` class that is a factory and convenience layer, and
storage interface classes implement access to different types of storage types
such as the local filesystem or remote object stores like Swift. There is an
abstract base class called ``StorageInterface`` that storage interface classes
should inherit from.

Storage Interface subclasses are responsible for implementing concurrency
control that cooperates with their actual storage.

Storage Interface classes are interfaces and may contain datasets (e.g.
in-memory storage), but they do not necessarily contain datasets.

Storage Class
^^^^^^^^^^^^^

Storage is a  factory class for Storage Interface instances. Storage interface
classes register themselves with the Storage class by calling
``Storage.registerStorageClass(scheme, cls)`` where ``scheme`` matches the
scheme of the URI that describes a Repository root, and ``cls`` is the class
object that implements the ``StorageInterface`` protocol.

Storage also is a helper for accessing storage interface functions. Without it,
users would have to call e.g. ``Storage.makeFromUri(uri).putRepositoryCfg(uri,
cfg)``, whereas with the helper in Storage, the user can call
``Storage.putRepositoryCfg(uri, cfg)`` and Storage handles making a temporary
Storage Interface instance inside the body of the function.

When getting a ``RepositoryCfg`` by calling ``Storage.getRepositoryCfg`` the
cfg is cached (key is the URI), so that multiple lookups are not performed on
storages (that might be remote and therefore expensive to fetch).
RepositoryCfgs are not supposed to change once they are created so this should
not lead to stale data.

StorageInterface
^^^^^^^^^^^^^^^^

Butler uses Storage Interface classes and a Storage factory class to allow
it to work with files that may be located anywhere (not just on the local
posix filesystem, e.g. in a Swift storage).

Storage-type specific classes should be used instead of calling filesystem
directly to access a repository. For example, an instance of ``PosixStorage``
is used to access the filesystem. A ``SwiftStorage`` instance will be used to
access a Repository in a Swift storage container.

Some ``StorageInterface`` subclasses are provided with Butler in daf_persistence,
they are discussed below. Others may be added by registering them with Storage
using its ``registerStorageClass`` function.

During ``Butler.__init__``, for each repository a ``StorageInterface`` subclass
is selected according to the scheme of the root URI passed into
``Butler.__init__`` (via the ``inputs`` and ``outputs`` arguments). For example
``root='swift://<swift parameters>'`` will select ``SwiftStorage`` interface
that is registered with Storage like
``Storage.registerStorageClass(scheme='swift', cls=SwiftStorage)``

The storage interface is still settling. The ``StorageInterface`` abstract base
class should be a superclass of all storage interfaces to ensure completeness
and help discoverability if/when the interface evolves.

The StorageInterface API is described in the class documentation.

PosixStorage
""""""""""""

``PosixStorage`` is the interface for file-based repositories on the local
filesystem. This interface will be used for root that are relative paths,
absolute paths that start with a single slash, and URIs that start with
``file:///`` (these will be treated as absolute paths).

``PosixStorage`` contains special functions that are used with 'old' butler
repositories. Butler framework classes use this API. These special functions
are not part of the StorageInterface API.

SwiftStorage
""""""""""""

``SwiftStorage`` is the interface for object-based repositories in Swift object
stores. This interface will be used for URIs that start with ``swift:///``.

To authorize the connection, the following environment variables must be set:

* ``SWIFT_USERNAME`` The username to use when authorizing the connection.
* ``SWIFT_PASSWORD`` The password to use when authorizing the connection.

The URI form is as follows:

.. code-block:: none

    swift://[URL without 'http://']/[Object API Version]/[tenant name (account)]/[container]

For example:

.. code-block:: none

    swift://nebula.ncsa.illinois.edu:5000/v2.0/lsst/my_container

Typically a single container holds a single repository, with an object called
``repositoryCfg.yaml`` in the container. There can be exceptions, for example
``CALIB`` is a second storage location used by ``CameraMapper`` and may not
contain a repository cfg, instead all the needed details are specified by the
primary storage location used by ``CameraMapper``.

SwiftStorage works by downloading objects to temporary files (python
``tempfile.NamedTemporaryFile``), and using PosixStorage to read the file
to instantiate the python object.

With NamedTemporaryFile handles, the file is deleted when the handle is deleted
(or explicitly closed). In some cases the file object should not be deleted
while the object exists (for example with an sqlite3 database file). To prevent
this from happening, the handle is monkey patched onto the object before it
is returned from ``SwiftStorage.read``.

Swift storage also caches the handle to the temporary file in case the object
is read again, thus saving time by not having to re-download the object. This
may not serve in place of monkey patching the file: this caching feature may
need to be optional (if the temporary file storage is getting too big), and it
is possible that objects could still exist after the SwiftStorage is destroyed.

Formatter
---------

A Formatter is an object that has the ability to serialize and deserialize
objects to and from a type of persistent storage. ``StorageInterface``
subclasses should use formatters to serialize & deserialize objects.

A Formatter is registered for a class type with the ``StorageInterface``
subclass that should use the formatter to serialize and deserialize instances of
that class.

The API of the Formatter is only dependent on the ``StorageInterface`` subclass
that will be calling it. The signature of functions called by the
``StorageInterface`` subclass should depend on what parameters are required to
read from and write to the persistent storage type.

Registering Formatters
^^^^^^^^^^^^^^^^^^^^^^

Formatters should be registered by calling

.. code-block:: python

    StorageInterface.registerFormatter(formatable, readFormatter, writeFormatter)

Typically ``formatable`` is the class type to serialize & deserialize.
``readFormatter`` is the function to call when reading and ``writeFormatter`` is
the function to call when writing.

``StorageInterface`` subclasses may choose to support something other than the
formatable python class type for the ``formatable`` argument. For example,
``PosixStorage`` accepts strings as they are specified in the ``storage``
parameter of the ``datasetType``.

An example of a formatter registration in ``daf_persistence``, in
``fmtPosixRepositoryCfg.py``:

.. code-block:: python

    PosixStorage.registerFormatter(RepositoryCfg, RepositoryCfgPosixFormatter)

This registers the ``RepositoryCfgPosixFormatter`` as a Formatter for
``RepositoryCfg`` classes to be written and read via a ``PosixStorage``.

Getting a Formatter
^^^^^^^^^^^^^^^^^^^

A ``StorageInterface`` subclass can get a Formatter by calling
``self._getFormatter`` and passing in the class type that will be read or
written.

In Butler the class type would typically come from a ``ButlerLocation`` instance
that had been created by the ``Mapper``. This call would look like this:

.. code-block:: python

    class PosixStorage(StorageInterface):
        def read(self, butlerLocation)
            formatter = self._getFormatter(butlerLocation.getPythonType())
            ...

ButlerLocation
--------------

A ``ButlerLocation`` class instance contains the results of a ``map`` action
including information from the policy such as what kind of object to instantiate
from the dataset as well as information for the ``Butler`` such as what Storage
instance to read the dataset from.

Compressed Datasets
^^^^^^^^^^^^^^^^^^^

The standard implementation of the ``Mapping`` object (used by ``CameraMapper``)
allows files to be compressed, with a ``.gz`` or ``.fz`` file extension. Support
for these extensions is hard coded into ``Mapping`` but could be refactored to
be specified by policy if necessary.

Mapper Configuration
--------------------

Policy
^^^^^^

The policy provides configuration details for the butler framework that will
access a dataset. The policy may be defined in any/all of:

1. The repository (as a parameter of the repository configuration)
2. The package that subclasses ``CameraMapper``. Policy files should be in a folder
   called 'policy' at the top level of the package directory (e.g.
   ``obs_test/policy/...``).
3. The butler framework provides basic policy details at
   ``daf_persistence/policy`` and ``obs_base/policy``.

If policy keys conflict, settings will override in that order, where the
in-repository settings will have highest priority.

In-repository policy is defined in new output repositories in the repository
args object passed to butler initialization, or in the repository configuration
of existing input and output repositories.

In-repository policy does not get inherited by child repositories; a parent
repository's policy is unique to that repository (unless explicitly duplicated
in the child repository's configuration).

Dataset Type
^^^^^^^^^^^^

A label given to a one or more datasets reflecting their meaning or usage
(not their persisted representation). Each dataset type corresponds to
exactly one Python type. Dataset types are used by convention by Tasks for
their inputs and outputs. Examples: `calexp`, `src`, `icSrc`.

Dataset Prototype
^^^^^^^^^^^^^^^^^

.. warning::

    Dataset Prototype is currently concept-ware and does not exist at all in
    code. See details below.

This concept is work-in-progress, and is related to making it possible to define
dataset types at runtime.
`DM-4180 - Butler: provide API so that a task can define the output dataset type
<https://jira.lsstcorp.org/browse/DM-4180>`_.

A labeled set of basic access characteristics serving as the basis for a
group of dataset types, used to define new dataset types. The characteristics
may include code, template strings, and other configuration data. Dataset
genres are often (but not necessarily) common to all dataset types with the
same Python type, making it easy for an application to select which genre is
applicable to a new dataset type that it is creating.

CameraMapper
------------

The `CameraMapper` is used by most packages that use the `Butler`.

(There is much to be said about the `CameraMapper` and this section is
work-in-progress.

Metadata Getter
^^^^^^^^^^^^^^^

The metadata of an object can be retrieved by adding ``_md`` to the end of the
dataset type name when calling ``Butler.get``. For example,
``calexpMetadata = butler.get(calexp_md, ...)``

Exposure component Getters
^^^^^^^^^^^^^^^^^^^^^^^^^^

For exposure datasets, the ``wcs``, ``calib``, ``visit``, and ``filter``
information can be retrieved by adding ``_`` plus the extension to the dataset
type when calling ``butler.get``, like so:

* ``<datasetType>_wcs``, for example ``wcs = butler.get(calexp_wcs, ...)``
* ``<datasetType>_calib``, for example ``calib = butler.get(calexp_calib, ...)``
* ``<datasetType>_visitInfo`` for example
  ``wcs = butler.get(calexp_visitInfo, ...)``
* ``<datasetType>_filter``, for example
  ``filt = butler.get(calexp_filter, ...)``

Bypass Functions
^^^^^^^^^^^^^^^^

If a ``CameraMapper`` subclass wants to use a different deserializer than the
standard Butler deserialization schemes, it can implement a function that starts
with ``bypass_`` followed by the datasetType name. The function signature must
be ``(self, datasetType, pythonType, location, dataId)`` where ``datasetType``
is the datasetType name (matches the policy), ``pythonType`` is a class instance
that the policy specifies for the datasetType's python type, ``location`` is a
``ButlerLocation``, and ``dataId`` is the the ``DataId`` instance that was used
to map the object. For example to implement an alternate reader for the
``calexp`` dataset, create a function
``def bypass_calexp(self, datasetType, pythonType, location, dataId)`` that
returns the read object.

Bypass functions do not participate in the Butler's deferred-read mechanism.
This is because with multiple repositories the Butler may successfully map the
ButlerLocation, but the object needed may actually exist in a parent repository.
Normally the Butler can accommodate this by looking to see if the located object
exists in the repository. But with a bypass function the Butler can not know
what object is actually needed, for example some bypass functions return
information derived from the `location.dataId` and do not actually need the
file itself to exist. Because the bypass function can not participate in the
deferred-read mechansim, Butler executes the bypass function while looking for
a valid location and if the bypass function succeeds Butler stops performing
lookups and returns the bypass results in the bypass attribute of the location.

The bypass function may raise any exception for any reason to indicate that
this bypass is not valid. All exceptions thrown by the bypass function are
silently caught by Butler. The most common case for an exception is that a file
does not exist. If the bypass function raises an exception the existence of the
bypass function is ignored and Butler proceeds performing lookups.

Standardize Functions
^^^^^^^^^^^^^^^^^^^^^

Sometimes objects require further processing after the Butler has instantiated
the object. To do this the mapper may declare a function with the object's
dataset type name, preceded by `std_`, that is: `std_<datasetType>`. The
signature must be `std_<datasetType>(self, item, dataId)`, where `item` is the
object to be standardized and `dataId` is the `dataId` that was passed in to
`Butler.get`, including any additional keyword arguments passed to `get` as
`**rest`. The function must return the standardized `item`. The passed-in
`item` may be modified and returned if that is what is desired.

Butler does not handle any exceptions thrown by the `std_` function, the user
code that called `Butler.get` must handle any possible exceptions raised by the
`std_` function.

Calibs
^^^^^^

The ``CameraMapper`` provides support for calibrations (e.g., bias, dark, flat,
fringe frames). An important requirement is that no calibration shall ever be
used unexpectedly. In order to achieve this requirement, we read and write the
calibrations from separate directories: calibrations shall be read from the
calibration root directory, and written to the same root directory as other
butler datasets.

Registry
--------

The mapper may use a Registry to look up data about an object when performing a
query. Currently this can be an sqlite3 database (the class that uses this is
``SqliteRegistry``) or a PostgreSQL database (using the ``PgsqlRegistry`` class).
Or if no database is found in the repository and
the repository is on the local filesystem (using a ``PosixStorage`` interface),
Butler will create a ``PosixRegistry`` class to perform data lookups on the
repository.

If a repository does not have a database registry then the Butler will look in
parent repositories for a parent with a suitable database registry and if/when
one is
found will stop looking, and pass that registry to the Mapper that is being
initialized as the ``parentRegistry`` init arg.

PostgreSQL registry
^^^^^^^^^^^^^^^^^^^

PostgreSQL registries can be used by supplying in the repository a YAML file
whose name is suffixed with ``.pgsql`` (typically ``registry.pgsql`` for the
raw data), containing the following:

* ``host``: Database hostname.
* ``port``: Database port.
* ``user``: Database username.
* ``database``: Database name to use.

In addition, an entry, ``password``, may be used with the database password.


DataId
------
A class that extends dict. As a dict it contains scientifically meaningful
key-value pairs the mapper to find a location of one or more datasets that
should be read or written.

It also contains a member variable called ``tag``:

* ``tag`` may be a string or other type, including container types. When
  searching repositories, if the tag argument is not None, then repositories will
  only be searched if their tag equals the value of tag (or if a match is found in
  either container of tags).
* When searching, if an input repository is tagged, all of its parents will be
  searched (even if they do not have a tag).
* The Butler API allows a dict to be passed instead of a DataId; as needed it
  will convert the dict into a DataId object (with no tags) internally.

Butler with Legacy Repositories
-------------------------------

_parent
^^^^^^^

Until March 2016 Butler did not have a class abstraction for repositories, and
a Butler was instantiated with a single repository. That single repository could
have "parent" repositories. This allowed the repository to access datasets from
other repositories. This was implemented putting a symlink at the top level of
the repository on disk (at  the location specified by "root") named ``_parent``
whose target was the root of the parent repository.

There is still support for ``_parent`` symlinks in the locations it was used as
of March 2016 (there is minimal support in the Butler framework classes and it
is mostly used by ``CameraMapper``). To the extent possible this will be
maintained but new code and features may not make any attempt to support it.

When searching multiple repositories (current implementation; parents and peers
set by the cfg) an 'old style' repo with _parent symlinks will be treated as a
single repository. IE the _parent symlinks get followed before the next repo in
``repository._parents`` is searched.

Subset
------

ButlerSubset is a container for ButlerDataRefs.  It represents a collection of
data ids that can be used to obtain datasets of the type used when creating the
collection or a compatible dataset type.  It can be thought of as the result of
a query for datasets matching a partial data id.

The ButlerDataRefs are generated at a specified level of the data id hierarchy.
If that is not the level at which datasets are specified, the
ButlerDataRef.subItems() method may be used to dive further into the
ButlerDataRefs.

DataRef
^^^^^^^
A ButlerDataRef is a reference to a potential dataset or group of datasets that
is portable between compatible dataset types.  As such, it can be used to create
or retrieve datasets.

ButlerDataRefs are (conceptually) created as elements of a ButlerSubset by
Butler.subset().  They are initially specific to the dataset type passed to that
call, but they may be used with any other compatible dataset type. Dataset type
compatibility must be determined externally (or by trial and error).

ButlerDataRefs may be created at any level of a data identifier hierarchy. If
the level is not one at which datasets exist, a ButlerSubset with lower-level
ButlerDataRefs can be created using ButlerDataRef.subItems().

DataRefSet
^^^^^^^^^^

Logically, a set of 'DataRef's. This may be implemented as an iterator/generator
in some contexts where materializing the set would be expensive. The
'DataRefSet' is usually generated by listing existing datasets of a particular
dataset type, but its component 'DataRef's can be used with other dataset types.

Composite Datasets
------------------

There need not be a one-to-one relationship between an instance of serialized
data, such as in a single FITS file, and datasets (where the definition of a
dataset is “the persisted form of an in-memory object; it could be a single
item, a composite, or a collection”). The relationship can occur as many
datasets in one file and can also occur as one dataset spread across many files.

A single file containing many (potentially overlapping) datasets is referred to as a "collection".
Typically the entire collection file can also be treated as a dataset, and the collection is usually written that way.
Each dataset in a collection file is retrievable independently, provided an appropriate plugin exists to read it.
Since there is nothing else special about them, they are not discussed further here.

A dataset that is composed of multiple files is referred to as a "composite dataset".
The composite dataset is composed of multiple "component" files, each
of which is its own independent dataset. The Butler Composite Dataset feature documented in this section supports getting python objects from these composite datasets.

Components
^^^^^^^^^^

A composite dataset is made up of component datasets, each of which may be any
kind of dataset, including another composite.

Components are specified in the policy, as named members under the ``composite``
section within the composite object datasetType definition:

.. code-block:: none

    <datasetType name>: {
        composite: {
            <component name>: {
                datasetType: <datasetType of component>
                ...
            }
            <component name>: {
                datasetType: <datasetType of component>
                ...
            }
            ...
        }
    }

Each component has a
named section that names the component and provides its dataset type and other
details, outlined below.

The datasetType of the component refers to another datasetType within the
policy. The details in the other datasetType will be used for getting and
putting this component.


Composeable and Decomposable Objects
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Python objects that should be created from or, may contain, individual component
objects must be able to be created from those separate components or have those
components assigned at a later time. Objects that should be persisted into
individual components must provide a means of accessing those component objects
for serialization (i.e. the class must provide getters or the member that is the
component must be accessible).

Assembler & Disassembler
^^^^^^^^^^^^^^^^^^^^^^^^

For every composite datasetType, an "assembler" function is required by Butler.
This function is used to assemble a composite object from its component parts.
Similarly, a "disassembler" is required, that can deconstruct a composite object
into component objects to be serialized individually.

There is a generic assembler and disassembler that can be used in some cases. If
it won't work, assembler and disassembler plugins can be specified.

Assembler & Disassembler Plugins
""""""""""""""""""""""""""""""""

Assembler and Disassembler functions are specified in the policy as members of the
composite object datasetType definition.

See :ref:`Composite Policy <composite-policy>` to see how to include the
assembler in the policy.

The assembler function signature is:

.. code-block:: none

    def <assembler function name>(dataId, componentInfo, cls):
        """Function for assembling <object> in Butler

        Parameters
        ----------
        dataId : dict
            The dataId that was used to find the objects in componentInfo.
        componentInfo : dict of componentDatasetName to ComponentInfo
            Keys are the component names as defined by the datasetType in the
            policy. Values are ComponentInfo objects.
        cls : class object
            A class object of the type specified for this datasetType by the policy.

        Returns
        -------
        object : instance of cls
            Object that has been assembled or constructed with component inputs.
        """

The disassembler function signature is:

.. code-block:: none

    def <disassembler function name>(obj, dataId, componentInfo):
        """Function for disassembling <object> in Butler

        Parameters
        ----------
        obj : object instance
            The object that will be disassembled into component parts.
        dataId : dict
            The dataId that is being used to persist this object.
        componentInfo : dict of componentDatasetName to ComponentInfo
            A dict of ComponentInfo instances to populate with the components
            that should be persisted for this composite. Keys are the component
            names as defined by the datasetType in the policy. Values are
            ComponentInfo objects.

        Returns
        -------
        None
        """

ComponentInfo
"""""""""""""

ComponentInfo is a class used when assembling and disassembling a composite
object in butler. It contains information about a component of a composite
object and has a slot for passing the component object into the assembler and for
the disssembler to pass the component object out.

ComponentInfo is used as an input to assemblers and disassemblers (which are
part of the butler public API).

Some ComponentInfo parameters are populated with information from the policy and
some are filled in by the butler, and when disassembling a composite object the
obj slot is populated by the disassembler. More details are available in the
class docstrings.

Generic Assembler & Disassembler
""""""""""""""""""""""""""""""""

Butler has a generic assembler and disassembler that can be used in some cases,
listed below. The list is in the order butler will try to apply the generic
assembler.

The generic assembler will be used when:

1. The policy for the composite does not name an assembler.
2. If a setter is not named for any component of the composite, and the python
   object's ``__init__`` function has input arguments for all the components
   that match the componentNames in the policy. In this case the object will be
   initialized with the components.
3. For each component:

 a. The policy names the setter for the component.
 b. The policy's component name matches the setter's name so that the setter
    name can be inferred (more on inference below).

The generic disassembler will be used when:

1. The policy for the composite does not name an disassembler.
2. For each component:

 a. The policy names the getter for the component.
 b. The policy's component name matches the getter's name so that the getter
    name can be inferred (more on inference below).

Setter & getter name inference:

For each component, if the policy does not specify a setter name and the python
object has setter names that match the component name then the setter name can
be inferred. It will first try ``'set' + <componentName>``, and if that does not
exist it will try ``'set' + <componentName>.capitalize`` (e.g. for component
name 'foo', it will try ``setfoo`` and then ``setFoo``.) If no setter is
specified and no setter can be found for a component object, it will raise a
runtime error.

Getter name inference works the same way (of course, replacing 'set' with 'get').

.. _composite-policy:

Component Subset
^^^^^^^^^^^^^^^^

If, instead of a single component object, a list of component objects is desired
for a given component, the keyword 'subset' can be added to the component
dataset definition, with the value ``True``.

.. code-block:: none

    <datasetType name>: {
        composite: {
            <component name>: {
                subset: True
                ...
            }
            ...
        }
    }

When Butler is getting components for a composite and sees this keyword, it will
use ``Butler.subset`` to find the component (instead of ``Butler.get``, which is
the normal case). When butler passes the dict of ``ComponentInfo`` to the
assembler, the ``ComponentInfo.obj`` parameter will be a list of component
objects.

Note that the generic assembler does not interpret the list of component
objects; the list will be passed as-is to the object constructor.

Input-Only Components
^^^^^^^^^^^^^^^^^^^^^

The policy may mark components as input-only. This allows the dataset to be
persisted to output repository (or repositories) without writing certain
components that should not be written. To do so, use the keyword
``inputOnly`` and make its value ``True``.

.. code-block:: none

    <datasetType name>: {
        composite: {
            <component name>: {
                inputOnly: True
                ...
            }
            ...
        }
    }


Composite Policy
^^^^^^^^^^^^^^^^

To indicate that a dataset should be serialized/deserialized from components,
the policy's dataset definition has a keyword ``composite``. The structure is:

.. code-block:: none

    <dataset type name>: {
        composite: {
            assembler: <importable function to assemble the components>
            disassembler: <importable function to assemble the components>
            <component name>: {
                datasetType: <dataset type>
                setter: <method name of setter>
                getter: <method name of getter>
                subset: bool
                inputOnly: bool
            }
            ...
        }
    }

Where:

<dataset type name>
    The name of the dataset

composite
    A new section. It is optional. (it should be omitted if the dataset is not a
    composite.)

<component name>
    A name that is used to refer to the component within the composite. Some
    default values can be inferred from the name (see setter & getter).

datasetType
    Names the dataset type that should be used for this component.

setter
    Names the method that is used to set the component in the composite class.
    Defaults to ``set<component name>``

getter
    Similar to 'setter': names the method that is used to get the component from
    the composite class. Defaults to ``get<component name>``

assembler
    Name of a function that can be used to instantiate the custom object.
    Optional, omit this parameter if the generic assembler can & should be used.

disassembler
    Similar to assembler but for custom deserialization.

subset
    Optional. If true, indicates that the object returned for this dataset type
    should be a list of objects, found by calling
    ``butler.subset(<dataset type>, dataId)``, where the dataset type is the component
    datasetType, and the dataId is what was passed into ``butler.get(..., dataId)``.

inputOnly
    Optional. If true, indicates that the object should not be put when putting
    the components of the composite dataset.

Component Dataset Location
^^^^^^^^^^^^^^^^^^^^^^^^^^

Component datasets may exist within the same or different repositories. Butler uses
``self.get(...)`` to find each component, so for each component the search will start
at the top of the repository search list.

Loading Components Individually
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It is possible to load a component object of a composite dataset without loading
the entire composite object. To do so, when calling :code:`butler.get()`, use
:code:`<datasetType>.<componentName>`. For example, when using a composite
declared like this:

.. code-block:: none

    calexp_psf: {
        ...type 1 dataset details...
    }

    calexp: {
        composite: {
            psf: {
                datasetType: 'calexp_psf'
            }
            ...
        }
        ...
    }

You can retrieve just the psf of the calexp by calling:

.. code-block:: python

    psf = butler.get('calexp.psf', dataId={...})

Similarly, you can put just the psf of the calexp by calling:

.. code-block:: python

    butler.put(psf, 'calexp.psf', dataId={...})

Note that componentName and its datasetType are in different namespaces. Because
of this, a component name can be the same string as its datasetType, but it does
not have to be.

DAX
===

Overview - What is DAX?
-----------------------

DAX is a collection of Data Access Interfaces implemented as REST APIs.
Currently, there are three core APIs: dbserv, metaserv, and imgserv. They are
all implemented in python using the `Flask framework
<http://flask.pocoo.org/>`_. The collection of these APIs referred to
as webserv.


dbserv
------
dbserv is the REST API for databases. The primary target for dbserv will be as
a frontend for QServ, but dbserv is generic enough to be used in front of any
database, providing the user a uniform interface and JSON data structure back.
dbserv's interface borrows heavily from the `IVOA Table Access Protocol
<http://www.ivoa.net/documents/TAP/20100327/REC-TAP-1.0.html>`_. While we aim
to provide a complete, TAP-compliant implementation, dbserv is currently a
small subset TAP. We implement only the `/sync` endpoint, and we also return
a JSON format

.. Link to dberv API here once we get sphinx autodoc works for dax_dbserv

Design Requirements
^^^^^^^^^^^^^^^^^^^

dbserv has two primary requirements. The first requirement is to behave as an
abstraction for database access, providing a portable way of performing both
synchronous and asynchronous queries, providing raw data access to any LSST
database through an HTTP API.  The second requirement of is, effectively, to
implement the features in TAP. This can be broken down into four parts:

The second requirement of is, effectively, to implement the features in TAP.
This can be broken down into four parts:

1. Serve and package data to the proper file format
   * (JSON, FITS, VOTable, HDF5, SQLite)
2. Implement ADQL (in some cases, a subset of ADQL)
3. Semantically-useful metadata about results (e.g. UCDs)
4. Handle table uploads from users

The current implementation of dbserv meets a small subset of these requirements.


Future work
^^^^^^^^^^^

In order to best meet these and future requirements moving forward, dbserv
will likely split into two components.

For the first requirement, we will have a component will behave as a lower
level API to the databases, optimized for the datacenter and simplicity of use.
While TAP could conceivably meet most of the needs, TAP's abstractions aren't
efficient for server to server communications, namely streaming, which is an
issue for results larger than a gigabyte.

The second component will act as an intermediary between the user and the lower
level component. In the common case of an ADQL query, this component will parse
the query, validate tables and columns, retrieve UCDs from metaserv (where
appropriate), and possibly rewrite the query to plain SQL for the lower level
API. If the downstream database does not directly implement asynchronous
queries, like the L2 database for example, this component will directly
implement them, otherwise it will defer to the implementation. Finally, this
component will be in charge of stitching together the raw data into an
appropriate file format and attaching semantic metadata about the results to
the file, like UCDs, whenever possible

In order to serve up UCDs and other semantic metadata about a query and/or it's
results, this second component will likely provide an API which a user might
also use in conjunction with the lower level API to mimic the functionality of
the full TAP implementation with the performance benefits of the lower level
API. This will likely be most useful for efficient server-to-server
communications, the likely customer being SUIT. It's also possible this API may
actually be implemented in metaserv.

imgserv
-------

imgserv is a lightweight API to serve up images. The current implementation of
imgserv uses the Butler to acquire relevant images, and imgserv transforms
those images in-process, providing cutouts of images. imgserv may also grow
into a collection of services including
`IVOA's Simple Image Access <http://www.ivoa.net/documents/SIA>`_  and
`SODA <http://www.ivoa.net/documents/SODA>`_.

.. Link to imgserv API here once we get sphinx autodoc works for dax_imgserv

metaserv
--------

metaserv is both a metadata database for dataset repositories and the API to
query that database. Currently, the only repositories that are supported are
databases repositories, and metaserv stores information about tables and their
columns, like UCDs. metaserv will likely grow to include metadata about images
and more generally, Butler repositories. It is not necessarily a goal of
metaserv to facilitate data discovery.

.. Link to metaserv API here once we get sphinx autodoc works for dax_metaserv


Future DAX Services
-------------------

We recognize the need for a Butler service which can act serve as a proxy for
remote Butler instantiations and also serve as a generic gateway for languages
where no Butler implementation exists, like Java. This use case is especially
desired by the SUIT team.


Change Record
=============

+-------------+------------+----------------------------------+-----------------+
| **Version** | **Date**   | **Description**                  | **Owner**       |
+=============+============+==================================+=================+
| 0.1         | 2/15/2016  | Initial version.                 | Jacek Becla     |
+-------------+------------+----------------------------------+-----------------+
