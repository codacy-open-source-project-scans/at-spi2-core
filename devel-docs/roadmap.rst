Accessibility infrastructure roadmap
====================================

Note that this roadmap refers exclusively to **cleanups** and **paying
technical debt**. It does not yet refer to actually changing the
accessibility interfaces, or the way accessibility works in general.
Possible API changes are discussed in the :doc:`at-spi3`.

High level goals, detailed below
--------------------------------

-  Make at-spi2-core/xml the authoritative version of the a11y
   interfaces.

-  Remove all the hand-written DBus code in at-spi2-core; replace with
   gdbus as generated from the XML files.

-  Add good end-to-end tests, to complement the ones already in
   at-spi2-atk.

-  Remove as much duplication / leftovers from the CORBA days /
   compatibility hacks as we can.

-  Give people the confidence that the accessibility stack works, so
   it’s their responsibility to make their apps and gnome-shell
   accessible.

Short term tasks
----------------

-  Merge these repositories into a single one: at-spi2-core, atk,
   at-spi2-atk. They are tightly coupled, and it will be much easier to
   test/refactor them if the code is in the same repo. See “`merge the
   repositories <#merge-the-repositories>`__” below for details.

- Replace manually-written DBus code in at-spi2-core for automatically
   generated code. See “`Remove hand-written DBus code
   <#remove-hand-written-dbus-code>`__” below for details.

-  `Make the XML interfaces the single source of
   truth <#make-the-xml-interfaces-the-single-source-of-truth>`__.

Details on big tasks
--------------------

Merge the repositories
~~~~~~~~~~~~~~~~~~~~~~

Merging at-spi2-core, atk, and at-spi2-atk into a single repository will
make end-to-end testing easier. As of 2022/May/27, at-spi2-atk has a few
end-to-end tests, which do not manage to exercise all the code in
libatspi or in atk. Having all the code in the same repository will make
it easier to refactor and test things. Some reasons to merge:

-  at-spi2-atk has the real end-to-end integration tests for
   at-spi2-core; it tests the part for an accessible application and the
   accessibility technology together, with the registry daemon in the
   middle. It would be nice to run these tests automatically when any of
   at-spi2-core or atk changes.

-  Both at-spi2-atk and pyatspi2 have “dummy” implementations of the ATK
   interfaces to use with their tests. It may be possible to deduplicate
   that.

-  Doing refactoring across all modules will be easier if their tests
   are in a single place.

-  We only have to get the CI working once, instead of three times.

Some cool things in at-spi2-core’s CI that now work for all the
accessibility middleware:

-  CI pipelines run the test suite and generate a code coverage report.
   This can be used to fine-tune tests, or decide how to refactor things
   for finer-grained testing.

-  Static analysis. Fix broken C code as early as possible.

-  Address Sanitizer - runtime analysis; does not fully work yet, but
   the basics are there.

-  Infrastructure to test build configurations for various distro
   images, e.g. dbus-daemon for openSUSE, versus dbus-broker for Fedora
   - those have different code paths.

Make the XML interfaces the single source of truth
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the past the at-spi2-core/xml files, which define the DBus interfaces
for accessibility, have gotten out of sync with at-spi2-core’s own C
code, because it does DBus calls by hand instead of using an
autogenerated binding - there weren’t any at the time it was written.
Also they have gotten out of sync with external copies, like GTK.
Perhaps we can add some bits to GTK’s CI scripts to guarantee that this
doesn’t happen, or make GTK just reference at-spi2-core/xml directly.

It may not be possible to “disappear” libatspi2 and rely only on an
autogenerated binding for the XML DBus interfaces everywhere, but that
should certainly be a goal.

Remove hand-written DBus code
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

My initial goal is to replace it with gdbus from GIO (an auto-generated
C binding to XML interfaces). I want to experiment with a few strategies
to make this easier. Right now all the DBus code is interwoven with
logic from at-spi2-core, so the first thing is to separate them. Then,
replace the DBus code in one shot for gdbus.

For example, there’s quite a bit of code like this:

.. code:: c

   static void
   handle_some_method (DBusMessage *message)
   {
     foo = demarshal_a_bit ();
     bar = find_the_accessible ();
     bar_set_foo (bar, foo);

     baz = demarshal_a_bit ();
     bar_set_baz (bar, baz);

     while (some_logic_here)
     {
       qux = demarshal_something ();
       blah_blah (qux, bar);
     }

     /* etc */
   }

So I would first split it into a part that demarshals everything, and
then one that does the logic:

.. code:: c

   typedef struct {
     Accessible accessible;
     Foo foo;
     Baz baz;
     Qux quxes[];
   } SomeMethodArgs;

   handle_some_method (DBusMessage *message)
   {
     SomeMethodArgs args;

     if (demarshal_some_method_args (message, &args) != SUCCESS)
     {
       return ERROR;
     }

     do_the_thing (args.accessible, args.foo, args.baz, args.quxes);
   }

Once everything is split apart, it’s a lot easier to replace the
demarshalers with gdbus calls, and the rest of the logic can hopefully
remain unchanged.

As a side benefit, this may allow testing the logic without having to
worry about inter-process communication. Test the thing; assume
communication works.

It may not be possible to separate all such cases so cleanly. However,
if a certain “sequence” requires intermediate IPC, then that is a good
indication for a less granular interface to add later: instead of
querying for an object and then for each of its properties / children /
etc., maybe send the whole object’s tree in a single call.

(Future goal: this separation of IPC vs. logic may make it easier, to
port the accessibility middleware to Rust - something I’d really like to
do.)

End-to-end tests
~~~~~~~~~~~~~~~~

There are already some end-to-end tests in at-spi2-atk/tests. They
create a mock AT and a mock accessible application, and ensure that what
they communicate through the accessibility bus matches on both ends.

These tests are good! Let’s add more extensive ones to help test things
like event throttling, unstable applications, compatibility APIs, etc.

Should we merge pyatspi2 in here?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Maybe! It is a compatibility layer, through and through, and maybe we
can disappear it gradually if we change Orca in lockstep.
