.. Delta Tool Online documentation master file, created by
   sphinx-quickstart on Wed Jul  1 08:08:59 2026.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Delta Tool Online
=================

Delta Tool Online provides a graphical user interface to the Python libraries `fmm_assess: FAIRMODE Modelling-Assessment Evaluation Tool <https://code.europa.eu/jrcairqualitymodelling/deltatool-assessment>`_ and `fmf_eval: FAIRMODE Forecast-Evaluation Tool <https://code.europa.eu/jrcairqualitymodelling/deltatool-forecast>`_, released as open-source on the European Commission platform `code.europa.eu JRC Air Quality Modelling project <https://code.europa.eu/jrcairqualitymodelling>`_.

Full description of the two underlying libraries is available at :ref:`Assessment computation details` and :ref:`Forecasting computation details`. In order to understand the indicators calculated by the two library, please refer to these sections: :doc:`/fmm_assess/INDICATORS.md` and :doc:`/fmf_eval/INDICATORS.md`. To fully understand the output plots, see the two detailed guides: `Plots reading guide for assessment <fmm_assess/PLOTS.html>`_ and :doc:`/fmm_assess/PLOTS.md` and :doc:`/fmf_eval/PLOTS.md`.

Delta Tool Online is available at `this URL <https://jeodpp.jrc.ec.europa.eu/eu/vaas/voila/render/fairmode/fairmode/DeltaTool.ipynb>`_, which can be accessed by providing EU-login account credentials.

The tool enables to upload users own datasets following the legacy Delta Tool format (startup.ini plus folder of observation and model CSV files) or the MQOR format (attributes and short term CSV, as documented `here <https://code.europa.eu/jrcairqualitymodelling/mqor>`_). Once loaded, stations data can be used for running experiments and calculate indicators and charts according to the Air Quality Directive 2881/2024.


.. note::

      While a user can run many experiments, only one dataset is allowed for each user. Please keep in mind that your uploaded data will be automatically deleted after 10 days of inactivity.


.. toctree::
   :maxdepth: 3
   :caption: Contents:
   
   self
   1_mainpage
   2_assessment
   3_forecasting
   4_datasets
   5_fmm_assess
   6_fmf_eval
