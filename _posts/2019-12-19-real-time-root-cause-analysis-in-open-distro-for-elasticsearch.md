---
layout: posts
author: Partha Kanuparthy, Joydeep Sinha, Karthik Kumarguru, Adithya Chandra, Balaji Kannan
comments: true
title: "Real Time Root Cause Analysis in Open Distro for Elasticsearch"
categories:
- odfe-updates
feature_image: "https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2019/03/26/open_disto-elasticsearch-logo-800x400.jpg"
---
The Open Distro for Elasticsearch Performance Analyzer captures Elasticsearch and JVM activity, as well as lower-level resource usage (e.g., disk, network, CPU and memory) of these activities. Based on this instrumentation, Performance Analyzer computes and exposes diagnostic metrics, with the goal of enabling Elasticsearch users and administrators to measure and understand bottlenecks in their Elasticsearch clusters. The Open Distro for Elasticsearch PerfTop client provides real time visualization of these diagnostic metrics to surface bottlenecks to Elasticsearch users and operators.

Today, we are open sourcing the Root Cause Analysis functionality for Open Distro for Elasticsearch. The source code can be found [here](https://github.com/opendistro-for-elasticsearch/performance-analyzer-rca). We are excited to continue building out the Root Cause Analysis framework as a part of Open Distro for Elasticsearch, and invite developers in the larger search community to join in and collaborate with us on development, design, testing. The feature includes a rich mix of distributed data flow graph processing, gRPC for networking, basic statistics for metric evaluation, systems work, and UI.

We have built the Root Cause Analysis  framework on top of Performance Analyzer to support root cause analysis (RCA) of performance and reliability problems for Elasticsearch instances. This framework conducts real-time root cause analysis of such problems using Performance Analyzer metrics. Root cause analysis can significantly improve operations, administration and provisioning of Elasticsearch clusters, and it can enable Elasticsearch client teams to tune their workloads to reduce errors.

## Model

We define a root cause as a function of one or more symptoms. A symptom is an operation applied to one or more metrics and/or other symptoms. Root causes may also be a function of other root causes. Note that these operations may involve aggregations; for example, a symptom could consume a time average of a metric. In addition, for confidence, a root cause could be a computation over a sufficiently long window of time. This definition does not allow for cycles in the dependency graph between metrics and root causes. The following equations show an example of these relationships:

![RCA dependency graph]({{ site.baseurl }}/assets/media/blog-images/rca-dependency-graph.png){: .blog-image }

Note that any of the functions above can take metadata as inputs, such as thresholds.

Root causes can span the entire stack, from the infrastructure layers (e.g., the OS, host, virtualization layers and the network) to the Java Virtual Machine to the Elasticsearch engine. Root causes also include problems related to the input workload to Elasticsearch.

## System Design

The root cause analysis framework helps diagnose root causes in real time. Based on the recursive model definition above, we build an acyclic data flow graph that takes metric streams generated by the Performance Analyzer plugin as input. Nodes of the data flow graph include computations such as metrics output (source nodes), aggregations, symptoms and root causes (sink nodes). The data flow graph across all root causes would span all nodes of an Elasticsearch cluster (including master nodes).

Edges of the graph transfer the output of a parent node to all child nodes.The framework treats this output as an opaque stream since the data format between nodes is a contract between each pair of nodes. The framework explicitly requires nodes to send timestamps — this is necessary for a node to diagnose issues with a parent node and handle staleness in data (e.g., data delivered late). Message delivery is ordered and provides at most once semantics (i.e., messages could be dropped to keep up with stream rate); small message loss isn’t a significant issue for root cause analysis because such algorithms rely significantly on statistical data. The following figure shows the above equations as a data flow graph (sources and sinks are shaded):

![RCA data flow graph]({{ site.baseurl }}/assets/media/blog-images/rca-dataflow-graph.png){: .blog-image }

The framework is  also fault tolerant for Elasticsearch, JVM and infrastructure performance and reliability problems. It exposes an API to query the current (or recent) set of diagnoses across some nodes or the entire cluster. The output  could be used by diagnostic tools (e.g., the Performance Analyzer PerfTop) or automated control plane actions.

All RCAs must be registered with the framework. This allows the framework to de-dup computations and optimize the streaming runtime. It exposes root causes and their context for applications to consume. Note that the framework resides in the agent process, so it is isolated from failures and performance problems in the Elasticsearch JVM. The architecture is shown below.

INSERT image

![RCA architecture]({{ site.baseurl }}/assets/media/blog-images/rca-architecture.png){: .blog-image }

The framework is designed to be fast and compute root causes in parallel. It executes each graph node in topological order as defined in the analysis graph. Nodes with no dependency are executed in parallel. If a host depends on a remote data stream for RCA computation, it subscribes to the data stream on startup. Subsequently, the output of every RCA execution on the upstream host is streamed to the downstream subscriber.

## Summary

In this blog post, we introduced the real time root cause analysis feature in Open Distro for Elasticsearch. It runs asynchronously as a side-car agent, and has very low overhead, which makes it suitable to run within the cluster without impacting cluster performance. We covered the basic concepts used in the framework and the system architecture, which makes root cause analysis process seamless.

We’re planning to build out functionality around identifying JVM bottlenecks and handling complex root causes for performance. We are excited for the future of real-time root cause analysis for Elasticsearch and welcome you to come join in and contribute with us in building the [root cause analysis](https://github.com/opendistro-for-elasticsearch/performance-analyzer-rca) framework in Open Distro for Elasticsearch.

## About the Authors

Partha Kanuparthy is a Principal Engineer working on database services at Amazon Web Services. His work spans distributed systems and databases, networking and machine learning. He actively contributes to open source software, and most recently, to Open Distro for Elasticsearch.

Joydeep Sinha is a Senior software engineer working on search services at Amazon Web Services. He is interested in distributed and autonomous systems. He is an active contributor to Open Distro for Elasticsearch.

Karthik Kumarguru is a Software engineer working on search services at Amazon Web Services. His primary interests are distributed systems and networking. He is an active contributor to Open Distro for Elasticsearch.

Adithya Chandra is a Senior software engineer working on search services at Amazon Web Services. He actively presents his work on root cause analysis and performance engineering most recently at Devoxx and is also an  active contributor to OpenDistro for Elasticsearch.

Balaji Kannan is an Engineering Manager working on search services at Amazon Web Services. He spent most of his career building vertical search engine and big data platforms. He contributes to Open Distro for Elasticsearch.