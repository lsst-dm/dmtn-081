..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 2

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   An outline of how to run a pipeline for alert packaging, streaming, filtering, and consuming.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).


Abstract
================

Primary access to the Large Synoptic Survey Telescope (LSST) alert stream is
planned to be via endpoints accessed through "community brokers".
Community brokers are likely to provide additional services, e.g.
enriching alerts with additional data from other catalogs or providing event
classification, and to allow end-users to filter these enriched alerts.
Supplementary to these community brokers, the LSST will provide a basic
alert filtering service with limited capacity.
The service will only allow access to the contents of each alert, i.e., no
outside information can be used in a filter.

The Data Products Definition Document (DPDD, LSE-163) describes how end-users
will build filters.
End-users will be able to write filters using simple code (likely written
as Python functions) or SQL-like queries.
The Data Management Systems Requirements document (DMSR, LSE-61) gives the
requirements for the alert filtering service, namely DMS-REQ-0342
(existence of filtering service), DMS-REQ-0348 (pre-defined filters),
and DMS-REQ-0343 (performance requirements).
The filtering service is expected to support ``numBrokerUsers`` = 100
simultaneous connected users each allocated a bandwidth capable of
receiving ``numBrokerAlerts`` = 20 full alerts per visit.


Pipeline outline
================

The prototype mini-broker described here includes the following components,
investigated and benchmarked in DMTN-028 :cite:`DMTN-028`:

* Mock alerts following the schema outlined in the DPDD, in Avro format
* A mock alert producer generating and sending alerts at expected LSST scale
* Instances of Kafka and Zookeeper, the proposed alert stream distribution platform
* Filters consisting of simple Python code
* Mock end-users reading filtered streams

The instructions below deploy an alert producer generating 10,000 alerts
per visit at a regular interval of one visit every 39 seconds.
The alerts are sent to a Kafka stream, where they are consumed by groups of
alert filters.
We use here 100 filters, split into five Kafka stream consumers each
having 20 filters.
The filters make simple magnitude cuts on the alerts, and each filter
allows a maximum of 20 alerts per visit to pass the filter.
Alerts that pass a filter are sent to a stream specific to that filter.
Science users can then subscribe to a specified filtered stream.


Deployment
================

All components of the mini-broker prototype can be deployed in Docker
containers.
Below provides instructions for use with Docker Swarm, which deploys
containers over a group of multiple machines joined in a cluster.
These instructions may need to be altered to deploy the containers in another
environment (e.g., Kubernetes).
Containers have been tested both on a single machine and on a cluster
of machines running a Swarm.

The Docker Swarm used for development was built via a Docker for AWS
cloud, consisting of ten m4.4xlarge machines
(16 vCPUs, 64 GiB memory, and 2000 Mbps EBS bandwidth).
Filters on machines with fewer than 16 CPUs were observed to be unable
to process a visit's worth of alerts before the next visit was started.
We tested up to 20 end-user filtered stream consumers and observed
no performance issues.
A larger cluster may be necessary for running at full scale
with 100 end-user consumers.

Services should be started in the following order: Kafka and
Zookeeper, end-user consumers listening to filtered streams,
filters, and then alert producers.

All the relevant code for the mini-broker prototype is available
in two GitHub repos:
https://github.com/lsst-dm/alert_stream
and
https://github.com/lsst-dm/sample-avro-alert.

To build the images for the containers for the alert stream producers,
filters, and consumers, checkout the ``alert_stream`` repo.
From the ``alert_stream`` directory:

.. code-block:: python

  docker build -t "alert_stream" .


Kafka
-------------

This prototype uses Confluent's distribution of Kafka, specifically
Confluent 4.1.1 and Kafka 1.1.0.
Kafka and Zookeeper can be run in containers.
To start Kafka and Zookeeper on a single node using
Docker Compose (i.e., not Swarm mode),
from the ``alert_stream`` directory run:

.. code-block:: python

  docker-compose up -d

In a larger scale Swarm, initiate the Swarm across
all machines and run the commands below to start the services.
It is good to keep Kafka and Zookeeper separate from
other containers by constraining Kafka and Zookeeper to the manager node(s)
and constraining the other services to worker nodes.

.. code-block:: python

  docker network create --driver overlay alert_stream_default

  docker service create \
          --name kafka \
          --network alert_stream_default \
          --constraint node.role==manager \
          -p 9092 \
          -e KAFKA_BROKER_ID=1 \
          -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:32181 \
          -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092 \
          -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \ # remove if starting 3 brokers or more
          -e KAFKA_HEAP_OPTS="-Xmx8g -Xms8g" \
          -e KAFKA_JVM_PERFORMANCE_OPTS="-XX:MetaspaceSize=96m -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M -XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80" \
          confluentinc/cp-kafka:4.1.1

  docker service create \
          --name zookeeper \
          --network alert_stream_default \
          --constraint node.role==manager \
          -p 32181 \
          -e ZOOKEEPER_CLIENT_PORT=32181 \
          -e ZOOKEEPER_TICK_TIME=2000 \
          confluentinc/cp-zookeeper:4.1.1

Kafka and Zookeeper should be listed when running

.. code-block:: python

  docker service ls


End consumers
-------------

Sample consumers can be started by running either ``printStream.py``,
which prints alert contents to the screen,
or ``monitorStream.py``, which prints the status of the latest offset
(number of alerts received).
Both of these scripts are found in the ``alert_stream`` GitHub rep.

To run the mini-broker prototype at full scale with 100 end-users,
100 containers would need to be deployed, each consuming a topic
following the naming scheme
``Filter001``, ``Filter002``, etc... ``Filter100``.

To deploy, e.g., the monitoring script using Docker on a single node,
consuming the stream for the 10th filter, run:

.. code-block:: python

  docker run -it --rm \
             --name=monitor010 \
             --network=alert_stream_default \
             alert_stream python bin/monitorStream.py Filter010

Output is directed to the screen.

To deploy the same process as a Swarm service, instead run:

.. code-block:: python

  docker service create \
          --name monitor010 \
          --network alert_stream_default \
          --constraint node.role==worker \
          -e PYTHONUNBUFFERED=0 \
          alert_stream python bin/monitorStream.py Filter010


Filters
-------------

Each individual filter is written as a class containing a function
that operates on the contents on an alert and returns true or false.
Filters are added by adding additional classes to ``filters.py``.
The filtering code limits the number of passing alerts to 20
alerts per visit.

In this prototype, each filter class name should include the
3 digit filter number, following the format of ``Filter001``,
etc., as the filter class name is the name of the filtered
stream from which the end-consumers read.

The filtering code takes as input a range of numbers of filters
to run at once, which consume a single instance of the unfiltered
stream in parallel.
For example, running

.. code-block:: python

    python bin/filterStream.py my-stream 1 10

will deploy filters 1 through 10 in a group.
Each group needs to read its own instance of the full stream.
To avoid performance issues which will result in lagging
filters, the number of groups (i.e., the number of instances
of the full stream flowing in this system) should be kept
to a minimum.
See DMTN-028 :cite:`DMTN-028`.
In this prototype, we recommend here to run five filter
groups, each acting on a group of 20 filters
(i.e., 1-20, 21-40, 41-60, 61-80, 81-100).
Each group of filters can be deployed in its own
Docker container.

To deploy, e.g., the first group of filters on an
unfiltered stream called ``full-stream`` on a single node,
run the following:

.. code-block:: python

    docker run -it --rm \
               --network=alert_stream_default \
               alert_stream python bin/filterStream.py full-stream 1 20

Alternatively, as a service in a Docker Swarm, run:

.. code-block:: python

      docker service create \
              --name filtergroup1 \
              --network alert_stream_default \
              --constraint node.role==worker \
              -e PYTHONUNBUFFERED=0 \
              alert_stream python bin/filterStream.py full-stream 1 20

This constrains filter groups to worker nodes, separate from
Kafka and Zookeeper.
You can also ensure that each filter group is deployed on its
own node by taking advantage of constraint ``node.id`` instead.


Alerts
---------------

The alerts used here have realistic content generated by
the Sims/Commissioning team, but lack object histories
and stamps.
(TODO: Add stamps.)
Alerts are in Avro format with a schema following the
schema detailed in the DPDD.
The schema and sample data can be found in the repo
https://github.com/lsst-dm/sample-avro-alert.

Included in the Docker image are a small number of
Avro files for testing.
Each file contains one visit of alerts.
The alerts were generated without a signal-to-noise
cut, and therefore each file contains more alerts than
expected per visit.
(TODO: Cut on signal-to-noise.)
However, the alert producer code limits the number of
alerts sent to Kafka to 10,000 per visit.


Alert producers
---------------

The ``sendAlertStream.py`` script reads Avro files
from the ``data`` directory and
produces alerts to Kafka, one visit every 39 seconds.
The code can use the sample files included in the
image or files mounted as a volume into a Docker
container to the internal ``data`` directory.
In this prototype only one alert producer is used.
Multiple producers could also be used by scaling up
the number of Docker containers and modifying the code
to produce the number of alerts which will yield
a total of 10,000 across all producers.

To run one alert producer on a single node, mounting
a local directory of Avro files inside the container, run:

.. code-block:: python

    docker run -it --rm \
               --name=sender \
               -v $PWD:/home/alert_stream/data:ro \
               --network=alert_stream_default \
               alert_stream python bin/sendAlertStream.py full-stream

To alternatively deploy the alert producer as a Swarm
service, run the following:

.. code-block:: python

    docker service create \
                  --name sender \
                  --network alert_stream_default \
                  -v $PWD:/home/alert_stream/data:ro \
                  -e PYTHONUNBUFFERED=0 \
                  alert_stream python bin/sendAlertStream.py full-stream

The local Avro files must be on the same node or otherwise
accessible to the alert producer container.


Evaluating results
==================

At the end of a successful run of the alert distribution
and mini-broker pipeline, end-user consumer containers should be
able to receive 20 filtered alerts from each visit, and,
at minimum, all components should process each visit's alerts
in enough time such that end-consumers do not receive a
filtered stream that lags behind the sequence of observations.
For details about the timing and performance of the component of
the pipeline from alert serialization to submission to the
distribution stage, see DMTN-028 :cite:`DMTN-028`.

The alert producer writes to stdout the time at which
the first serialized alert is read from an Avro file
and the time at which the last alert has been submitted for
distribution.
End-consumer containers should receive and process 20 alerts per visit
before the next visit has started.
Two consumer types are provided here.
One consumer prints alerts to stdout and prints a status message that
is only produced when reaching the end of a stream after processing
all messages available at that time.
The monitor consumer drops alert contents, instead printing only
end-of-stream status messages.
The status messages contain the time at which the last available message
has been processed and the running total (offset)
of the number of alerts processed.
An example end-of-stream message is provided below:

.. code-block:: python

    topic:full-stream, partition:0, status:end, offset:1000, key:None, time:1528496269.734

If status messages are produced, the end-user consumer
containers have been able to process messages faster than
they are submitted for distribution.
The difference between the time logged by the status messages and the
time logged by the alert producer gives the end-to-end
alert submission to end-user consumer receipt time.


.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :encoding: latex+latin
    :style: lsst_aa
