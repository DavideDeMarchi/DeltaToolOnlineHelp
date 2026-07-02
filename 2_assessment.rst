Assessment
==========

.. toctree::
   :maxdepth: 3
   
Load your dataset
-----------------

Delta Tool Online enables the users to upload their own datasets containing stations and air quality monitoring data. The accepted formats are the **Delta Tool Legacy format** (startup.ini file and two CSV files for each station, one for the observations and one for the model), and the **MQOR format**.

The **Delta Tool legacy format** is described in detail in the `Assessment inputs section <fmm_assess/TECH_SPEC_fmm_assess.html#inputs>`_.

.. warning::

    While the underlying **fmm_assess** library supports both CSV and NetCDF forma for stations data, currently only the CSV version of the format is supported by the Delta Tool Online
    
The **MQOR format** is described in the `MQOR main repository <https://code.europa.eu/jrcairqualitymodelling/mqor>`_.


Load the sample dataset
^^^^^^^^^^^^^^^^^^^^^^^

The simplest way to start using the Delta Tool Online is to click the "Load sample dataset" button in the Dataset upload dialog-box. This function loads, inside the user storage space, a simple dataset consisting of around 50 stations covering all european countries. After the loading, a download of the dataset can be useful to better understand the correct input format for uploading your own dataset. Please refer to the :ref:`Stations toolbar` section to see how to download your current dataset.


Load a dataset in the Delta Tool legacy format
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. figure:: graphics/LoadDatasetDT.png

   Load dataset using the legacy Delta Tool format


Load a dataset in the MQOR format
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. figure:: graphics/LoadDatasetMQOR.png

   Load dataset using the MQOR format


After the loading
^^^^^^^^^^^^^^^^^

.. figure:: graphics/ConsistencyChecks.png
    

   Consistency checks on the uploaded dataset
   

.. figure:: graphics/DatasetDisplay.png

   Display of the uploaded dataset
   

To start analysing the content of your dataset and to filter/select the input stations for your experiments, please see :ref:`dataset-summary`.


Run an experiment
-----------------

.. figure:: graphics/AssessmentRun.png

   Input parameters for running an experiment


Analyse experiment results
--------------------------


Numerical results
^^^^^^^^^^^^^^^^^

.. figure:: graphics/AssessmentNumeric.png

   Numerical results of an assessment experiment
   

Charts outputs
^^^^^^^^^^^^^^

.. figure:: graphics/AssessmentCharts.png

   Graphical results of an assessment experiment


Map output
^^^^^^^^^^

.. figure:: graphics/AssessmentMap.png

   Map visualization of output indicators per station
   

Compare two experiments
-----------------------
