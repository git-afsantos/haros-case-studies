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
The files found under the [`json` directory](./json/) can be copied to the HAROS viz's `data` directory for visualisation.

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

> **COR** - correct attributes in the extracted model<br/>
> **INC** - incorrect attributes in the extracted model<br/>
> **PAR** - partially correct attributes in the extracted model<br/>
> **MIS** - missing attributes in the extracted model<br/>
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

> **Query 1:** *Are global ROS names used?*

Global ROS names (names beginning with a slash, e.g., `/laser`) are unaffected by most name resolution rules of ROS.
This requires additional attention when creating multiple instances of the same Node within a Configuration, as the global names will collide without proper remappings.
For instance, if two Kobuki robots existed within the same network and they used global names to publish sensor readings, it is likely that each robot would receive sensor data from both.
This situation leads to additional maintenance effort, thus justifying the query.

> **Query 2:** *Are there any conditional publishers or subscribers?*

Conditional publishers and subscribers can be easily spotted in the Visualiser graphs (they use dashed lines).
There is no justification for this query besides these constructs leading to an additional effort in understanding the architecture.
In many cases, such conditional links stem from loops, where topics are subscribed to from a list parameter, and there is no significantly better alternative.

> **Query 3:** *Do message types match on both ends?*

There is a type checking system for messages in ROS, but it is only active during runtime.
When a message type mismatch occurs, a warning is given and messages of the wrong type are discarded.
This lack of communication often has noticeable effects, but bringing this feedback to compile time is simple with our query system, and is an additional step to reduce development time.
The query is formulated for topics only, but implementing a similar type checking mechanism for services would be equally simple.

> **Query 4:** *Are there any unbounded message queues?*

Message queue sizes should be carefully chosen in ROS.
Avoiding unbounded message queues is a given, as they could use up all the available memory if a node does not process its queue fast enough.

> **Query 5:** *Are there any message queues of size 1?*

Queues of size 1 are a very particular case with a niche application.
They are relatively common in practice, but they can easily lead to message loss.
Whether it is an intended effect should be analysed on a case by case basis.
For instance, in teleoperation nodes, it could be intentional.
We want nodes to process the last given command as fast as possible, and we count on the human operator to issue appropriate commands for the current situation.
This is especially important for emergency stop commands.

> **Query 6:** *Are there multiple publishers on a single topic?*

In most cases, having multiple publishers on a single topic is unintended, likely due to a missing name remapping.
For pure data producers, such as sensors, merging multiple publishers under a single topic would likely lead to erroneous behaviour.
For controllers, a single source of direct commands is also desirable, in order to avoid conflicts.
This is why a common solution for multiple command sources is to introduce a multiplexer between the controllers and the actuator.
A valid use of multiple publishers on a single topic would be for sporadic (or high-level) events where the robot has multiple units capable of detecting, or producing, such events.

> **Query 7:** *Are there any disconnected topics of the same type, with similar names?*

This query is slightly more complex than the others, in terms of implementation, but it shows how the query language allows for some programmatic freedom.
We used some heuristics based on string comparison of the Topic names of publishers and subscribers, to try and detect those where the message types match, but the names are only slightly different -- e.g., in `/laser` and `/robot/laser` the proper names match, but the namespace prefix does not.
Many times, this could be an indicator that the developer applied a name remapping on one of the nodes, but forgot to match it in another node, or that they forgot to apply remappings altogether.
Wrong name remappings are another issue in ROS that can be manually detected during runtime (via inspection), but for which there are no built-in compile-time checks of any kind.

> **Query 8:** *Are there any uses of the message type `std_msgs/Empty`?*

This is a message type that contains no data, and thus should be treated specially.
It is useful as a trigger for events or actions, but one should consider whether additional data could be of use.

Overall there are 29 query matches, although none seem to be faults in the system.
This is to be expected, since Kobuki has had years of development, testing and maintenance by many people.
Such issues would have been detected and fixed at some point.

### Property-based Testing

The Kobuki system has been used to evaluate the property-based test generation [plugin](https://github.com/git-afsantos/haros-plugin-pbt-gen) for HAROS.
As a system that has had time to mature, the goal of this experiment was not to find bugs in its implementation, but rather to demonstrate the effectiveness of this testing technique.

#### Methodology

The property-based testing (PBT) technique used by this plugin must be able to reset the system under test (SUT) multiple times, to guarantee that, for each generated example, we run the SUT from a stable (and hopefully deterministic) state.
This series of resets occurs in rapid succession, and is entirely impractical if hardware is involved.
Thus, we opt to modify the tested configurations slightly (in particular, the *Safe Random Walker*) to remove any nodes that directly control hardware (the mobile base driver).
Removing nodes turns out to be beneficial for another reason; it opens up several topics that the plugin can exploit as inputs, versus a system that could, otherwise, be fully connected.

The plugin is evaluated on several fronts, and we used it to test not only the *Safe Random Walker* configuration but also some individual nodes, such as the safety controller.
We first wrote specifications for each node, tested them, and then proceeded to the integrated environment.
Not only does this reflect a more natural testing process adopted during development, it also helps us coming up with properties to write.
Depending on the target system and on the specific nodes we are testing, some of the node's properties might convert, more or less directly, either to properties of the full system or to axioms that we want to assume for other tests.
Note that, despite the additional work in writing the specifications, from the perspective of the testing tool there is no difference between testing individual nodes or entire systems -- the SUT is always treated as a black-box, with input and output message topics.

Given that PBT focuses on discrepancies between implementation and specification, our evaluation process involves two types of testing, *positive testing* and *negative testing*.
The former is the act of testing properties that we know to be true in advance.
Its main purpose is simply to confirm that the plugin does not introduce false positives.
The latter, on the other hand, is the act of testing properties that we know to be false in advance.
In this case, the plugin should be able to find at least one counterexample for most properties.

#### Evaluation Criteria

Instead of simply writing arbitrary properties about the SUT, we follow a systematic method to write specifications.
We start by writing a catalogue of true properties and axioms (assumptions) for each system.
Then, we apply the principles of *specification mutation* to generate **mutants** (i.e., variants) from the initial catalogue of true properties.
Mutations are often relatively small changes, such as changing a single operator or a variable.
If the initial properties are as precise as we can write them, small changes introduced by the mutation process should end up generating a false property, in most cases.

For each property mutant, we are interested in gathering the following metrics:

- the number of required examples until a test fails for the first time;
- the number of required examples until the final test result (includes the *shrinking* phase);
- the number of invalid examples (discarded due to being unable to satisfy assumptions);
- the required time to run each example (both from start to end, including launching the SUT, and to replay the example trace);
- the size of the produced counterexample trace.

With these metrics we are able to calculate aggregate metrics to provide an overview of the tool's performance.
For instance, we are able to calculate:

- the average number of examples needed to find a failure;
- the average number of examples needed to report the final result;
- the mean time to failure;
- the excess messages in the counterexample input trace (i.e., how close it is to being a minimal counterexample);
- specification coverage, a metric that assigns a score to a test suite and tells how effective it is at exercising the given specification.

#### Artefacts

The [project file](./projects/pbt-eval.yaml) contains all the tested configurations and the full catalogue of properties for each configuration.
It can be executed with the following command.

```bash
haros analyse -w haros_plugin_pbt_gen -n -p pbt-eval.yaml
```

The original Kobuki launch files could not be used directly, as these included hardware-related nodes.
The modified versions we used (referred to as part of the `hpl_test` package) can be found in the [launch directory](./launch/).

#### Results

The safety controller node was tested with a specification consisting of 12 axioms, 6 properties and 59 mutants.
We achieved a near-perfect mutant score for this SUT, failing to kill a single mutant with a limit of 200 examples.

The random walker controller node was tested with a specification consisting of 11 axioms, 4 properties and 34 mutants.
We achieved a near-perfect mutant score for this SUT, failing to kill a single mutant with a limit of 200 examples.
The mutant that we failed to kill is basically the same mutant that survived in the safety controller node, only for a different topic.

The multiplexer node was tested with a shorter specification consisting of 2 axioms, 2 properties and 17 mutants.
One of the main reasons for the shorter specification of this node was due to limitations of the specification language at the time.
This SUT achieved a perfect mutant score.
Out of the 17 killed mutants, 4 were reported as flaky tests, i.e., the results were not deterministic.

The full *Safe Random Walker* configuration was tested with a specification consisting of 14 axioms, 5 properties and 45 mutants.
The axioms are a combination of the same axioms used for the safety controller and the random walker controller nodes.
This SUT also achieved a perfect mutant score, as shown in the following table.
Out of the 45 killed mutants, only 2 were reported as flaky tests, with one of them being essentially the same property that led to false negatives in the safety controller and random walker controller nodes.

| Metric          | Value |
|:--------------- | :---: |
| Mutants         | 45    |
| Flaky Tests     | 2     |
| False Negatives | 0     |
| Mutant Score    | 1.0   |

Regarding other performance criteria, as per the following table, we did not observe anything out of the ordinary.
The performance metrics seem to be on par with the tests for individual nodes.
On average, each mutant requires about 20 examples to falsify (including the shrinking phase), considering the median of 16 and the mean of 22, although the tool tends to find an error for the first time relatively early (within the first 3 examples).
Each example takes about 4.0 seconds to execute, and most of this time (3.2 seconds) is spent on launching and tearing down the SUT between examples.
In total, this means that the average time to kill a mutant and obtain a counterexample is about 80 seconds.
If a full ROS system could be reliably reset programatically (without killing the processes and launching new ones), it would take about 16 seconds instead.
Despite the relatively long execution time, there are only 2 mutants for which the tool uses one message more than necessary.
Curiously, neither of the non-minimal counterexamples is associated with the two flaky mutants.

| Metric                      | Mean  | Std. Deviation | Median |
| :-------------------------- | :---: | :------------: | :----: |
| Number of examples          | 22    | 23             | 16     |
| Number of failures          | 15    | 9              | 15     |
| Examples to failure         | 3     | 9              | 1      |
| Invalid examples            | 3     | 12             | 0      |
| Shrink attempts             | 19    | 17             | 15     |
| Time per example (s)        | 3.967 | 1.602          | 3.309  |
| Trace duration (s)          | 0.800 | 1.068          | 0.308  |
| Setup/Teardown time (s)     | 3.167 | 0.535          | 3.001  |
| Input trace size (messages) | 1     | 1              | 1      |
| Excess Messages             | 0     | 0              | 0      |

### Model Checking

