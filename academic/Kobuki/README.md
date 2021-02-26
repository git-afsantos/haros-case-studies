# Kobuki

## System

The Kobuki robot ([Home](http://kobuki.yujinrobot.com/), [ROS Robots](https://robots.ros.org/kobuki/), [GitHub](https://github.com/yujinrobot/kobuki)) is a low-cost mobile research base designed for education and research on state of the art robotics.

At its core, this system is composed of:

- a driver that controls its sensors (bumper, cliff and wheel drop) and actuators (two wheels);
- a safety controller that stops the robot or moves it back to safety when dangerous situations are detected;
- a random walker controller that moves the robot about, without a specific goal;
- a teleoperation controller;
- a velocity multiplexer that assigns different priorities to the various sources of velocity commands;
- a velocity smoother that detects and smoothes out sudden velocity changes;
- an autodocking controller that returns the robot to its battery charger.

We have analysed multiple configurations of this system, but our main focus has been on a configuration that includes the driver, the random walker controller, the safety controller and the velocity multiplexer (*Safe Random Walker*).

## Versioning

We analysed the most up-to-date source code under ROS Indigo, ROS Kinetic and ROS Melodic.
Various versions of HAROS were used, from 3.0 to 3.9.

## Analysis

Kobuki has been the go-to example to try out practically every feature of HAROS available at the time of writing.
As such, it has been analysed in terms of:

- compliance with coding standards;
- compliance with code quality metrics standards;
- automatic model extraction;
- architectural queries;
- runtime verification;
- property-based testing;
- model checking.

In the following subsections we provide further details on our approach to the various themes.
The files found under the [*json* directory](./json/) can be copied to the HAROS viz's `data` directory for visualisation.

### Code Quality Analysis

### Model Extraction and Architectural Analysis

We extracted models of various Kobuki configurations, ranging from individual nodes, such as the safety controller and random walker controller, to full systems, such as the *Safe Random Walker* configuration.
We then used these models for various purposes, such as evaluation of the model extractor's analysis and architectural analysis using a small catalogue of rules.

#### Model Extraction Performance

We used the *Safe Random Walker* configuration in order to evaluate the performance of the HAROS model extractor with the [Model Graph Extraction Difference plug-in](https://github.com/git-afsantos/haros-plugin-model-ged) (see the [project file](./projects/model-perf.yaml)).

To run this analysis we used the following command.

```bash
haros analyse -w haros_plugin_model_ged --no-hardcoded -n -p model-perf.yaml
```

Overall, we have found that, without specifying extraction hints, HAROS is able to extract **93.52%** of the true model correctly (*F1-score*), with a *precision* of **98.48%** and a *recall* of **89.04%**, as shown in the following table.

| Entity      | COR | INC | PAR | MIS | SPU | Precision | Recall | F1-score |
|:---         |:---:|:---:|:---:|:---:|:---:|   :---:   |  :---: |  :---:   |
| Node        | 36  | 0   | 0   | 0   | 0   | 1.0000    | 1.0000 | 1.0000   |
| Parameter   | 185 | 0   | 0   | 0   | 0   | 1.0000    | 1.0000 | 1.0000   |
| Publisher   | 165 | 2   | 0   | 28  | 1   | 0.9821    | 0.8462 | 0.9091   |
| Subscriber  | 123 | 1   | 1   | 6   | 1   | 0.9802    | 0.9427 | 0.9611   |
| Client      | 0   | 0   | 0   | 0   | 0   | 1.0000    | 1.0000 | 1.0000   |
| Server      | 0   | 0   | 0   | 5   | 0   | 1.0000    | 0.0000 | 0.0000   |
| Setter      | 0   | 0   | 0   | 12  | 0   | 1.0000    | 0.0000 | 0.0000   |
| Getter      | 104 | 3   | 0   | 18  | 1   | 0.9630    | 0.8320 | 0.8927   |
| **Overall** |**613**|**6**|**1**|**69**|**3**|**0.9848** |**0.8904**|**0.9352**|

> **COR** - correct attributes in the extracted model
> **INC** - incorrect attributes in the extracted model
> **PAR** - partially correct attributes in the extracted model
> **MIS** - missing attributes in the extracted model
> **SPU** - spurious attributes in the extracted model

In this case, we missed the extraction of Links that come from composite primitives, provided by packages other than the base ROS client libraries (not yet included in the parsing capabilities of HAROS).
For instance, we missed Links (full entities, with multiple attributes) created by the `tf`, `diagnostics` and `dynamic_reconfigure` packages.
The `dynamic_reconfigure` case, in particular, is a recurrent issue that lowers the metrics considerably, as the use of this package entails at the very least 5 different Links, with a single line of C++ code.
We also missed subscriptions of the multiplexer node, which are dynamically created, in a loop, after reading subscriber data from a parameter YAML file.
Lastly, we missed the default values (attributes) for three parameters that are defined in constants of a C++ implementation file (not a header) that is not part of the analysed source code (i.e., we only consider the binaries of that package).
All issues are related to C++ parsing; launch file parsing for this configuration was perfect.

In terms of extraction hints for this configuration, we ended up fixing 6 entities and creating 11 new entities, as shown in the project file and in the following snippet (excerpt).

```yaml
configurations:
    random_walker_hints:
        launch:
            - kobuki_node/launch/minimal.launch
            - kobuki_random_walker/launch/safe_random_walker_app.launch
        hints:
            nodes:
                /mobile_base:
                    publishers:
                        -   topic: /tf 
                            create: true
                            msg_type: tf2_msgs/TFMessage
                            queue_size: 100
                            traceability:
                                package: kobuki_node
                                file: src/library/odometry.cpp
                                line: 31
                                column: 1
                        -   topic: /diagnostics 
                            create: true
                            msg_type: diagnostic_msgs/DiagnosticArray
                            queue_size: 1
                            traceability:
                                package: kobuki_node
                                file: src/library/kobuki_ros.cpp
                                line: 81
                                column: 1
                    getters:
                        ...
                /cmd_vel_mux:
                    publishers:
                        -   topic: /cmd_vel_mux/?
                            original_name: /cmd_vel_mux/output/cmd_vel
                            conditional: false
                        -   topic: /cmd_vel_mux/parameter_descriptions 
                            create: true
                            msg_type: dynamic_reconfigure/ConfigDescription
                            queue_size: 1
                            latched: true
                            traceability:
                                package: yocs_cmd_vel_mux
                                file: src/cmd_vel_mux_nodelet.cpp
                                line: 96
                                column: 32
                        -   topic: /cmd_vel_mux/parameter_updates
                            create: true
                            msg_type: dynamic_reconfigure/Config
                            queue_size: 1
                            latched: true
                            traceability:
                                package: yocs_cmd_vel_mux
                                file: src/cmd_vel_mux_nodelet.cpp
                                line: 96
                                column: 32
                    subscribers:
                        ...
                    servers:
                        ...
                    setters:
                        ...
                    getters:
                        ...
```

Note that the creation of a single entity via hints fixes multiple missed attributes in the previous table.
For instance, we can see from the snippet above that we have created 4 Publishers.
Each Publisher corresponds to 7 attributes, namely `topic`, `msg_type`, `queue_size`, `package`, `file`, `line` and `column`.
Thus, 4 entities with 7 attributes each, yields the 28 missed Publisher attributes in the initial report.

#### Architectural Queries

In order to test the HAROS [Pyflwor query engine plugin](https://github.com/git-afsantos/haros-plugin-pyflwor), we implemented a small catalogue of queries, based on common practices in ROS development.
We ran these queries over the previously mentioned *Safe Random Walker* configuration.
The queries can be found in the [project file](./projects/model-queries.yaml) and ask the following questions.

**Query 1:** *Are global ROS names used?*<br/>
Global ROS names (names beginning with a slash, e.g., `/laser`) are unaffected by most name resolution rules of ROS.
This requires additional attention when creating multiple instances of the same Node within a Configuration, as the global names will collide without proper remappings.
For instance, if two Kobuki robots existed within the same network and they used global names to publish sensor readings, it is likely that each robot would receive sensor data from both.
This situation leads to additional maintenance effort, thus justifying the query.

**Query 2:** *Are there any conditional publishers or subscribers?*<br/>
Conditional publishers and subscribers can be easily spotted in the Visualiser graphs (they use dashed lines).
There is no justification for this query besides these constructs leading to an additional effort in understanding the architecture.
In many cases, such conditional links stem from loops, where topics are subscribed to from a list parameter, and there is no significantly better alternative.

**Query 3:** *Do message types match on both ends?*<br/>
There is a type checking system for messages in ROS, but it is only active during runtime.
When a message type mismatch occurs, a warning is given and messages of the wrong type are discarded.
This lack of communication often has noticeable effects, but bringing this feedback to compile time is simple with our query system, and is an additional step to reduce development time.
The query is formulated for topics only, but implementing a similar type checking mechanism for services would be equally simple.

**Query 4:** *Are there any unbounded message queues?*<br/>
Message queue sizes should be carefully chosen in ROS.
Avoiding unbounded message queues is a given, as they could use up all the available memory if a node does not process its queue fast enough.

**Query 5:** *Are there any message queues of size 1?*<br/>
Queues of size 1 are a very particular case with a niche application.
They are relatively common in practice, but they can easily lead to message loss.
Whether it is an intended effect should be analysed on a case by case basis.
For instance, in teleoperation nodes, it could be intentional.
We want nodes to process the last given command as fast as possible, and we count on the human operator to issue appropriate commands for the current situation.
This is especially important for emergency stop commands.

**Query 6:** *Are there multiple publishers on a single topic?*<br/>
In most cases, having multiple publishers on a single topic is unintended, likely due to a missing name remapping.
For pure data producers, such as sensors, merging multiple publishers under a single topic would likely lead to erroneous behaviour.
For controllers, a single source of direct commands is also desirable, in order to avoid conflicts.
This is why a common solution for multiple command sources is to introduce a multiplexer between the controllers and the actuator.
A valid use of multiple publishers on a single topic would be for sporadic (or high-level) events where the robot has multiple units capable of detecting, or producing, such events.

**Query 7:** *Are there any disconnected topics of the same type, with similar names?*<br/>
This query is slightly more complex than the others, in terms of implementation, but it shows how the query language allows for some programmatic freedom.
We used some heuristics based on string comparison of the Topic names of publishers and subscribers, to try and detect those where the message types match, but the names are only slightly different -- e.g., in `/laser` and `/robot/laser` the proper names match, but the namespace prefix does not.
Many times, this could be an indicator that the developer applied a name remapping on one of the nodes, but forgot to match it in another node, or that they forgot to apply remappings altogether.
Wrong name remappings are another issue in ROS that can be manually detected during runtime (via inspection), but for which there are no built-in compile-time checks of any kind.

**Query 8:** *Are there any uses of the message type* `std_msgs/Empty`*?*<br/>
This is a message type that contains no data, and thus should be treated specially.
It is useful as a trigger for events or actions, but one should consider whether additional data could be of use.

Overall there are 29 query matches, although none seem to be faults in the system.


### Property-based Testing

### Model Checking

