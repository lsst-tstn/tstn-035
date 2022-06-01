:tocdepth: 1

.. sectnum::

Abstract
========

This tech-note explores how the Rubin Observatory Control System can handle Targets of Opportunity (ToO). It is outside the scope of this document the selection criteria, e.g. which ToOs will be executed.  Here we will focus on the operational aspects; from receiving and curating requests to actually executing the observations.

Introduction
============

During the lefetime of the Legacy Survey of Space and Time (LSST) we expect several important astronomical events that may require rapididly follow-up observations with the Vera Rubin Observatory, including, but not limited to Gravitational-wave events :cite:`2022ApJS..260...18A`.

The importance of rapididly responding to astronomical events was recognized early on, which is reflected in specific Scheduler requirements; SCD-REQ-0018 and SCD-REQ-0019 on :lse:`369`.
These requirements reads:


   SCD-REQ-0018

      Specification: The Scheduler shall accept a list of TOO observations that can be configured with
      sufficient lead time.

   SCD-REQ-0019

      Specification: The Scheduler shall accept unscheduled TOOs as they are communicated.


In the following sections we will explore how to address those requirements with the currently existing system functionalities and what are the missing functionalities that need to be addressed in order to fulfill them.

Basic Concepts
==============

For the purpose of this document it is important for us to understand, at least at a higher-level, how observations are executed by the Rubin Observatory Control System.

During normal operations there are two main components involved in driving the system; the ScriptQueue and the Scheduler Commandable SAL Components (CSCs).

The ScriptQueue CSC execute SAL Scripts that are responsible for actually driving the telescope and camera to take data.
Users have access to several SAL Scripts that can execute numerous activities like, preparing the system for observations, actually executing observing sequences and much more.

On the other hand, the Scheduler CSC computes targets and queues them on the ScriptQueue for execution.
This interaction is done automatically, once the CSCs are enabled and resumed.
 
While the Scheduler is driving the observations, users can also interact with the ScriptQueue to execute additional operations alongside the Scheduler.
This interoperation allows us, for instance, to execute user-defined sequence of observations between scheduler-driven operations.

The Scheduling Algorithm
------------------------

The Scheduler CSC itself only implements the business logic required to compute an observing queue and to execute them on the ScriptQueue.
The actual algorithm responsible for digesting the telemetry stream and producing targets interfaces with the CSC by means of a well defined API, called the ``Driver``.

For the time being the project has settle on the "Feature Based Scheduler" as the baseline application for implementing the scheduling algorithm.

Integration between the Featurer Based Scheduler and the Scheduler CSC is already completed and been heavily used to drive observations with the Auxiliary Telescope.

Handling ToOs
=============

The Rubin Observatory Control System design allow ToOs to be handled in a number of different ways.
It is likely that, during operations. more than one mode of operation will be used, depending on the ToO conditions and project guidelines.

For now we will focus on the two main different ways to respond to ToOs; autonomously through the Scheduler CSC and manually through the ScriptQueue.

Autonomous Response With the Scheduler
--------------------------------------

In principle, in order to autonomously respond to ToOs the Scheduler CSC needs first to be configured with a Feature Based Scheduler setup that is capable or responding to inputs from a ToO alert system.

The Feature Based Scheduler already has a `ToO_survey`_, that is currently successfully used for simulation purposes and that could also be used for operations.

.. _ToO_survey: https://rubin-sim.lsst.io/api/rubin_sim.scheduler.surveys.ToO_survey.html#rubin_sim.scheduler.surveys.ToO_survey

With that in place, the only additional element required by the Scheduler is a source of ToOs events.

ToO Events as Telemetry
~~~~~~~~~~~~~~~~~~~~~~~

Currently the Scheduler CSC colects telemetry from the EFD and hands them over to the ``Driver``, which is in charge of formatting it in a way the scheduling algorithm understands.
By using a general purpose interface for colecting and passing telemetry from the EFD to the scheduling algorithm, it allows us to easily add new data sources as long as the data is in the EFD.

It is natural then to expect that, any new information needed by the Scheduler, should also be in the EFD.

By writting ToO events to the EFD the Scheduler CSC will naturally be able to access them.
Next, all that is required is for the ``Driver`` be able to parse the information in a way the scheduling algorithm understands.

This design also has the advantage that it allows us to control which ToO events the Scheduler should consider by filtering which events are written to the EFD, or adding metadata to indicate whether an event was certified or not.

The diagram below summarizes the design of the system.

.. figure:: /_static/ToO_diagram.png
   :name: fig-too-diagram
   :target: ../_images/ToO_diagram.png
   :alt: ToO automatic response system design.

   Design of the system to autonomously handle ToO with the Scheduler CSC.

The service in charge of reading/receiving ToO alerts from external sources is the ``ToO Alert Producer``, shown in the :ref:`diagram above <fig-too-diagram>`.
This service will also be in charge of certifying/validating which ToO events the Scheduler will consider.

Furthermore, the ``ToO Alert Producer`` can be designed in such a way to support multiple ``ToO Alert Stream`` sources.

Manual Response With the ScriptQueue
------------------------------------

As mentioned above, in order for the Scheduler CSC to be able to respond to a certain type of ToO, it needs to be pre-configured with a setup capable of processing the incoming data stream and acting accordingly.
This process should work on those cases where we know what to expect and are capable of planning ahead of time (e.g. Gravitational-wave events that got pre-approved by the project).

Nevertheless, it might be that an important unplanned event occurs and requires immediate response, for which the Scheduler CSC won't be capable of handling.

For those situations, a list of observations can be generated and added to the ScriptQueue.
Since the Scheduler and the ScriptQueue are designed to seamlessly interoperate, it is possible the manually add a series of observations to the queue without worrying about the Scheduler.

In these cases, the ScriptQueue will execute the observations, the Scheduler will wait for the execution to complete and then resume operations.

It is worth noticing that this use-case might be a good application of a service like `SkyPortal`_, which allows users to interactively select and process ToO events and generate sequences of observations.

.. _SkyPortal: https://skyportal.io

The main draw-back of this approach is that we lose a lot of the responsiveness provided by the Scheduler to the overall observatory conditions.
For example, the Scheduler is able to take into account the wind speed and direction when deciding which targets to observe, a condition that can change pretty drastically in a short timescale.
It is also possible to account for things like the telescope azimuth phase-wrap, thus avoiding long slews to unwrap it.
All these conditions, and many more, are not easily taken into account if the sequence of observations is pre-computed externally and sent to the ScriptQueue.

Therefore, although we acknowledge that this is a viable option for responding to unplanned ToOs, we expect to be able to respond to most of them autonomously through the Scheduler CSC.

.. Filtering Events 
.. ----------------

.. TODO


.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
