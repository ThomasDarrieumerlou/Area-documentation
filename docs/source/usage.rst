Usage
=====

.. _installation:

Set Up Data base
-----------------

Make sure that the area user has all permissions over the Area database and that the database exists.

.. code-block:: console

   (.venv) $ cat area.sql | mysql -u area -p Area




.. _start:

Start Project
----------------

To start the server use :

.. code-block:: console

   (.venv) $ make run t=server

To start the web client use :

.. code-block:: console

   (.venv) $ make run t=web

To use the mobile client download the apk.


start Docker
-------------

To start the project we use containerisation software.
It is called Docker and its purpose is to compile our project on a container.

The docker component is divided into two parts:
the first one is the docker compose, the neuralgic centre of the docker, it is him who will allow to containerize the different parts, but he cannot act all alone here is the second point
In the second part we treat more precisely the parts with dockerfiles which are in the functioning like makefile. It will allow to install all the tools necessary in the containers for the various parts.

This command up the db and environement :

.. code-block:: console
   
   (.env) $ docker-compose up


Next you run this command to run project :

.. code-block:: console
   
   (.venv) $ docker-compose run .

