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
   :align: center

   Summary panel showing stations type and area info

The top-right section of the "Dataset Summary" page displays information about the loaded dataset's content, including its storage size and the total, filtered, and selected number of stations.

.. note::

    The "Dataset Summary" page allows you to browse the dataset and decide which stations to include in your experiments. You can do this using two distinct subsetting mechanisms: stations can be filtered using their alphanumeric attributes (the columns displayed in the stations table), and/or they can be (yellow) selected by clicking on rows in the stations table or on station points on the map. These operations are described in more detail in the :ref:`Stations table` and :ref:`Stations map` sections. The window for configuring experiment parameters clearly displays the current number of filtered and (yellow) selected stations, allowing you to choose which subset to use as the input for your experiment (see the :ref:`Run an experiment` page for reference).

The summary panel also shows the date and time the dataset was loaded and last modified (for example, when an experiment is executed or removed). The two animated pie charts below are linked to the stations table and the stations map; they graphically visualize the distribution of stations by Type (background, industrial, traffic) and by Area (urban, suburban, rural).

Finally, the bottom part of the summary panel displays the number of stations for each pollutant defined in the dataset's **startup.ini** file. Like the pie charts, this section is dynamically updated by the filtering mechanism.


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
