.. _wiki2_adding_authorization:

====================
Adding Authorization
====================

:app:`Pyramid` provides facilities for :term:`authentication` and
:term:`authorization`.  We'll make use of both features to provide security
to our application.  Our application currently allows anyone with access to
the server to view, edit, and add pages to our wiki.  We'll change that
to allow only people who possess a specific username (`editor`)
to add and edit wiki pages but we'll continue allowing anyone with access to
the server to view pages.

We will do the following steps:

* Add a :term:`root factory` with an :term:`ACL` (``models.py``,
  ``__init__.py``).
* Add an :term:`authentication policy` and an :term:`authorization policy`
  (``__init__.py``).
* Add an authentication policy callback (new ``security.py`` module).
* Add routes for /login and /logout (``__init__.py``).
* Add ``login`` and ``logout`` views (``views.py``).
* Add :term:`permission` declarations to the ``edit_page`` and ``add_page``
  views (``views.py``).
* Make the existing views return a ``logged_in`` flag to the renderer (``views.py``).
* Add a login template (new ``login.pt``).
* Add a "Logout" link to be shown when logged in and viewing or editing a page
  (``view.pt``, ``edit.pt``).

The source code for this tutorial stage can be browsed at
`http://github.com/Pylons/pyramid/tree/1.3-branch/docs/tutorials/wiki2/src/authorization/
<http://github.com/Pylons/pyramid/tree/1.3-branch/docs/tutorials/wiki2/src/authorization/>`_.

Adding A Root Factory
~~~~~~~~~~~~~~~~~~~~~

Open ``tutorial/tutorial/models.py`` and add the following import
statement at the head:

.. literalinclude:: src/authorization/tutorial/models.py
   :lines: 1-4
   :linenos:
   :language: python

Add the following class definition:

.. literalinclude:: src/authorization/tutorial/models.py
   :lines: 35-39
   :linenos:
   :language: python

The ``RootFactory`` class is a :term:`root factory` that will be used by
:app:`Pyramid` to construct the :term:`context` of each request to
our application.  The context is attached to the request
object passed to our view callables as the ``context`` attribute,
and will be decorated with security declarations.   By using a custom
root factory to generate our contexts, we can use the
declarative security features of :app:`Pyramid`.

Open ``tutorial/tutorial/__init__.py`` and add a ``root_factory``
parameter to our :term:`Configurator` constructor, that points to
the class we created above:

.. literalinclude:: src/authorization/tutorial/__init__.py
   :lines: 19-20
   :linenos:
   :emphasize-lines: 2
   :language: python

(Only the highlighted line needs to be added.)

The context object generated by our root factory will possess an ``__acl__``
attribute that allows :data:`pyramid.security.Everyone` (a special principal)
to view all pages, while allowing only a :term:`principal` named
``group:editors`` to edit and add pages.  The ``__acl__`` attribute attached
to a context is interpreted specially by :app:`Pyramid` as an access control
list during view callable execution.  See :ref:`assigning_acls` for more
information about what an :term:`ACL` represents.

.. note::

    Although we don't use the functionality here, the ``factory`` used
    to create route contexts may differ per-route as opposed to globally.  See
    the ``factory`` argument to
    :meth:`pyramid.config.Configurator.add_route` for more info.

Add an Authorization Policy and an Authentication Policy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For any :app:`Pyramid` application to perform authorization, we need to add a
``security.py`` module (we'll do that shortly) and we'll need to change our
``__init__.py`` file to add an :term:`authentication policy` and an
:term:`authorization policy` which uses the ``security.py`` file for a
*callback*.

We'll enable an ``AuthTktAuthenticationPolicy`` and an ``ACLAuthorizationPolicy``
to implement declarative security checking. Open ``tutorial/__init__.py`` and
add these import statements:

.. literalinclude:: src/authorization/tutorial/__init__.py
   :lines: 2-3,7
   :linenos:
   :language: python

Now add those policies to the configuration:

.. literalinclude:: src/authorization/tutorial/__init__.py
   :lines: 16-22
   :linenos:
   :emphasize-lines: 1-3,6-7
   :language: python

(Only the highlighted lines need to be added.)

Note that the
:class:`pyramid.authentication.AuthTktAuthenticationPolicy` constructor
accepts two arguments: ``secret`` and ``callback``.  ``secret`` is a string
representing an encryption key used by the "authentication ticket" machinery
represented by this policy: it is required.  The ``callback`` is a
``groupfinder`` function in the current directory's ``security.py`` file.  We
haven't added that module yet, but we're about to.

Adding an authentication policy callback
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a new ``tutorial/tutorial/security.py`` module with the
following content:

.. literalinclude:: src/authorization/tutorial/security.py
   :linenos:
   :language: python

``groupfinder()`` is an :term:`authentication policy`
"callback"; it is a function that accepts a userid and a request and
returns one of these values:

- If the userid exists in the system, the callback will return a
  sequence of group identifiers (or an empty sequence if the user
  isn't a member of any groups).
- If the userid *does not* exist in the system, the callback will
  return ``None``.

We've given the ``editor`` user membership to the ``group:editors`` by
mapping him to this group in the ``GROUPS`` data structure above.
Since the ``groupfinder`` function
consults the ``GROUPS`` data structure, this will mean that, as a
result of the ACL attached to the :term:`context` object returned by
the root factory, and the permission associated with the ``add_page``
and ``edit_page`` views, the ``editor`` user should be able to add and
edit pages.

In a production system, user and group
data will most often come from a database, but here we use "dummy"
data to represent user and groups sources.

Add routes for /login and /logout
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Go back to ``tutorial/tutorial/__init__.py`` and add these two
routes:

.. literalinclude:: src/authorization/tutorial/__init__.py
   :lines: 25-26
   :linenos:
   :language: python

Adding Login and Logout Views
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To our ``views.py`` we'll add a ``login`` view callable which renders a login
form and processes the post from the login form, checking credentials.

We'll also add a ``logout`` view callable to our application and
provide a link to it.  This view will clear the credentials of the
logged in user and redirect back to the front page.

The ``login`` view callable will look something like this:

.. literalinclude:: src/authorization/tutorial/views.py
   :lines: 89-115
   :linenos:
   :language: python

The ``logout`` view callable will look something like this:

.. literalinclude:: src/authorization/tutorial/views.py
   :lines: 117-121
   :linenos:
   :language: python

The ``login`` view callable is decorated with two decorators, a
``@view_config`` decorator, which associates it with the ``login``
route, and a ``@forbidden_view_config`` decorator which turns it in to
an :term:`exception view`.  The one which associates it with the
``login`` route makes it visible when we visit ``/login``.  The other
one makes it a :term:`forbidden view`.  The forbidden view is
displayed whenever Pyramid or your application raises an
:class:`pyramid.httpexceptions.HTTPForbidden` exception.  In this
case, we'll be relying on the forbidden view to show the login form
whenever someone attempts to execute an action which they're not yet
authorized to perform.

The ``logout`` view callable is decorated with a ``@view_config`` decorator
which associates it with the ``logout`` route.  This makes it visible when we
visit ``/logout``.

We'll need to import some stuff to service the needs of these two functions:
the ``pyramid.view.forbidden_view_config`` class, a number of values from the
``pyramid.security`` module, and a value from our newly added
``tutorial.security`` package.  Add the following import statements to the
head of ``tutorial/tutorial/views.py``:

.. literalinclude:: src/authorization/tutorial/views.py
   :lines: 9-18,24-25
   :linenos:
   :emphasize-lines: 3,7-8,12
   :language: python

(Only the highlighted lines need to be added.)

Add permission declarations
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add a ``permission='edit'`` parameter to the ``@view_config``
decorator for ``add_page()`` and ``edit_page()``, for example:

.. code-block:: python
   :linenos:
   :emphasize-lines: 2

   @view_config(route_name='add_page', renderer='templates/edit.pt',
                permission='edit')

(Only the highlighted line needs to be added.)

The result is that only users who possess the ``edit``
permission at the time of the request may invoke those two views.

We've granted the ``group:editors`` :term:`principal` the ``edit``
permission in the :term:`root factory` via its ACL, so only a user who
is a member of the group named ``group:editors`` will be able to
invoke the views associated with the ``add_page`` or ``edit_page``
routes.

Return a logged_in flag to the renderer
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add the following import statement to the head of
``tutorial/tutorial/views.py``:

.. code-block:: python
   :linenos:

   from pyramid.security import (
        authenticated_userid,
        )


Add a  ``logged_in`` parameter to the return value of
``view_page()``, ``edit_page()`` and  ``add_page()``,
like this:

.. code-block:: python
   :linenos:
   :emphasize-lines: 3

   return dict(page = page,
               content = content,
               logged_in = authenticated_userid(request),
               edit_url = edit_url)

(Only the highlighted line needs to be added.)

:meth:`~pyramid.security.authenticated_userid()` will return None
if the user is not authenticated, or some user id it the user
is authenticated.

Adding the ``login.pt`` Template
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create ``tutorial/tutorial/templates/login.pt`` with the following
content:

.. literalinclude:: src/authorization/tutorial/templates/login.pt
   :language: xml

The above template is referred to within the login view we just
added to ``views.py``.

Add a "Logout" link when logged in
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Open ``tutorial/tutorial/templates/edit.pt`` and
``tutorial/tutorial/templates/view.pt``  and add this within the
``<div id="right" class="app-welcome align-right">`` div:

.. code-block:: xml

   <span tal:condition="logged_in">
      <a href="${request.application_url}/logout">Logout</a>
   </span>

The attribute ``tal:condition="logged_in"`` will make the element be
included when ``logged_in`` is any user id. The link will invoke
the logout view.  The above element will not be included if ``logged_in``
is ``None``, such as when a user is not authenticated.

Seeing Our Changes
~~~~~~~~~~~~~~~~~~

Our ``tutorial/tutorial/__init__.py`` will look something like this
when we're done:

.. literalinclude:: src/authorization/tutorial/__init__.py
   :linenos:
   :emphasize-lines: 2-3,7,16-18,20-22,25-26
   :language: python

(Only the highlighted lines need to be added.)

Our ``tutorial/tutorial/views.py`` will look something like this
when we're done:

.. literalinclude:: src/authorization/tutorial/views.py
   :linenos:
   :emphasize-lines: 11,14-18,56,59,71,74,86,89-115,117-121
   :language: python

(Only the highlighted lines need to be added.)

Our ``tutorial/tutorial/templates/edit.pt`` template will look
something like this when we're done:

.. literalinclude:: src/authorization/tutorial/templates/edit.pt
   :emphasize-lines: 41-43
   :language: xml

(Only the highlighted lines need to be added.)

Our ``tutorial/tutorial/templates/view.pt`` template will look
something like this when we're done:

.. literalinclude:: src/authorization/tutorial/templates/view.pt
   :emphasize-lines: 41-43
   :language: xml

(Only the highlighted lines need to be added.)

Viewing the Application in a Browser
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can finally examine our application in a browser (See
:ref:`wiki2-start-the-application`).  Launch a browser and visit
each of the following URLs, check that the result is as expected:

- ``http://localhost:6543/`` invokes the
  ``view_wiki`` view.  This always redirects to the ``view_page`` view
  of the FrontPage page object.  It is executable by any user.

- ``http://localhost:6543/FrontPage`` invokes
  the ``view_page`` view of the FrontPage page object.

- ``http://localhost:6543/FrontPage/edit_page``
  invokes the edit view for the FrontPage object.  It is executable by
  only the ``editor`` user.  If a different user (or the anonymous
  user) invokes it, a login form will be displayed.  Supplying the
  credentials with the username ``editor``, password ``editor`` will
  display the edit page form.

- ``http://localhost:6543/add_page/SomePageName``
  invokes the add view for a page.  It is executable by only
  the ``editor`` user.  If a different user (or the anonymous user)
  invokes it, a login form will be displayed.  Supplying the
  credentials with the username ``editor``, password ``editor`` will
  display the edit page form.

- After logging in (as a result of hitting an edit or add page
  and submitting the login form with the ``editor``
  credentials), we'll see a Logout link in the upper right hand
  corner.  When we click it, we're logged out, and redirected
  back to the front page.
