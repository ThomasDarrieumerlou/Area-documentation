Usage
=====

.. _installation:

Set Up Data base
------------

Make sure that the area user has all permissions over the Area database and that the database exists.

.. code-block:: console

   (.venv) $ cat area.sql | mysql -u area -p Area

Sart Project
----------------

To start the server use :

.. code-block:: console

   (.venv) $ make run t=server

To start the web client use :

.. code-block:: console

   (.venv) $ make run t=web

To use the mobile client download the apk.