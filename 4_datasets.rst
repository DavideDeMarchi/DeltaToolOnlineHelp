Dataset summary, stations filtering and selection
=================================================

.. toctree::
   :maxdepth: 3


This section explains how to interact with the dataset display page to analyse the dataset content and to filter and select the input stations for an experiment. The functions available are exactly the same both for assessment and forecasting experiments.

Stations are displayed on a table and on a map and the two views are always linked.

Summary panel
-------------

.. figure:: graphics/summary_panel.png
   :width: 500px

   Summary panel showing stations type and area info

The top-right section of the "Dataset Summary page" displays info on the loaded dataset content, in terms of storage occupation, also reporting the total, filtered and selected number of stations.

.. note::

   The "Dataset Summary page" allows the user to browse the dataset and to decide on which stations to run his experiments. Users can do this by means of two distinct subsetting mechanism: stations can be **filtered** using their alphanumeric attributes (the columns displayed on the stations table) and/or can be **(yellow) selected** by clicking on the rows of the stations table or on the station points on the map, operations that are better described in the :ref:`Stations table` and in the :ref:`Stations map` sections. In the window that collects parameters for running an experiment, the current number of filtered and (yellow) selected stations is clearly displayed, and the user can decide which subset to use as input for the experiment (see :ref:`Run an experiment` page for reference).

The summary panel also shows the date and time the dataset was loaded and last modified (for instance when an experiment is executed or removed). The two charts below are linked to the stations table and the stations map, and graphically visualise the share of stations by Type (background, industrial, traffic) and by Area (urban, suburban, rural).


Stations toolbar
----------------

.. figure:: graphics/stations_toolbar.png

   Toolbar containing functions to interact with the list of stations


Stations table
--------------

.. figure:: graphics/stations_table.png

   Stations table
   

Stations map
------------

.. figure:: graphics/stations_map.png

   Stations map
