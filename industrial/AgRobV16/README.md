# AgRob V16

## System

[AgRob V16](http://agrob.inesctec.pt/), part of the [RoMoVi project](https://www.inesctec.pt/en/projects/romovi#about) is a modular robotic platform for hillside agriculture, adapted to sloping and uneven terrains.
It is designed for monitoring, precision spraying, pruning and selective harvesting, particularly in steep slope vineyards.
Among other innovative features, this robotic platform is equipped with an advanced navigation system that allows it to estimate its location even when GPS is not fully available.
Its modularity makes it able to integrate various types of payload, both in the form of sensors and actuators.
The main driver behind this project is to provide commercial solutions for hillside vineyards capable of autonomously executing operations of Monitoring and Logistics.
Such features will help winemakers make important decisions, increasing wine quality and production.

A large part of this project's code base is closed source, and, as such, we are limited in the analysis outputs and artefacts we are able to share.
A notable exception is its path planning component, which is [open source](https://gitlab.inesctec.pt/agrob/agrob_pp/) and will be described as its own [case study](../AgRobPP/).

For the purposes of our case study, we consider a so-called *Basic* configuration of this robot.
This configuration allows a user to alternate between teleoperation and autonomous navigation using a joystick.
Its core components are:

- a mobile base controller;
- laser sensors;
- cameras;
- map servers;
- a custom Joystick Controller node;
- a Supervisor node; and
- a Safety Controller node.

The Safety Controller node is responsible for reading the various sensors (lasers, IMU, cameras) and determining the current state of the robot (e.g., *safe*, *caution* or *danger*).
The Supervisor node is (in concept) a glorified velocity multiplexer.
It receives velocity commands from multiple sources (e.g., joystick, autonomous navigation) and selects the appropriate command to pass down to the mobile base, according to the state reported by the Safety Controller node.
In addition, it also ensures that all velocity commands are within dynamic maximum and minimum velocity limits (that also depend on the safety state).
The Joystick Controller converts joystick commands into velocity commands and changes the robot's mode of operation (via a message sent to the Supervisor) depending on whether certain buttons are pressed.


## Versioning

We analysed the most up-to-date source code for this project under ROS Kinetic and ROS Melodic.
HAROS version 3.9 was used to conduct the experiments.


## Analysis

AgRob V16 has been one of our case study subjects of choice to try out most features of HAROS available at the time of writing.
As such, it has been analysed in terms of:

- compliance with coding standards;
- compliance with code quality metrics standards;
- automatic model extraction;
- architectural queries;
- property-based testing (and runtime verification);
- model checking.

In the following subsections we provide further details on our approach to the various themes.
As previously mentioned, due to this project being closed source, we are only able to share some elements of the analysis.
In particular, we will share some details regarding model extraction, property-based testing and model checking.

The files found under the [`json` directory](./json/) can be copied to the HAROS viz's `data` directory for visualisation.


### Model Extraction and Architectural Analysis

We extracted models of various AgRob V16 configurations, ranging from individual nodes, such as the Safety Controller and Supervisor, to full systems, such as the *Basic* and *Path Planning* configurations.
We then used these models to evaluate the performance of the HAROS model extractor with the [Model Graph Extraction Difference plug-in](https://github.com/git-afsantos/haros-plugin-model-ged) (see the [project file](./projects/model-perf.yaml)).

To run this analysis we used the following command.

```bash
haros analyse -w haros_plugin_model_ged --no-hardcoded --env -n -p model-perf.yaml
```

Given the similarities between the two configurations, for brevity, we will show here the results for the *Path Planning* configuration only, which is the most complex between the two.
Overall, we have found that, without specifying extraction hints, HAROS is able to extract **87.4%** of the true model correctly (*F1-score*), with a *precision* of **98.8%** and a *recall* of **78.36%**, as shown in the following table.

| Entity      | COR | INC | PAR | MIS | SPU | Precision | Recall | F1-score |
|:---         |:---:|:---:|:---:|:---:|:---:|   :---:   |  :---: |  :---:   |
| Node        | 138 | 0   | 0   | 0   | 0   | 1.0000    | 1.0000 | 1.0000   |
| Parameter   | 583 | 2   | 0   | 0   | 0   | 0.9966    | 0.9966 | 0.9966   |
| Publisher   | 166 | 1   | 1   | 42  | 0   | 0.9911    | 0.7929 | 0.8810   |
| Subscriber  | 156 | 0   | 0   | 0   | 0   | 1.0000    | 1.0000 | 1.0000   |
| Client      | 14  | 3   | 3   | 0   | 0   | 0.7750    | 0.7750 | 0.7750   |
| Server      | 0   | 0   | 0   | 15  | 0   | 1.0000    | 0.0000 | 0.0000   |
| Setter      | 0   | 0   | 0   | 156 | 0   | 1.0000    | 0.0000 | 0.0000   |
| Getter      | 339 | 9   | 0   | 156 | 0   | 0.9741    | 0.6726 | 0.7958   |
| **Overall** |**1396**|**15**|**4**|**369**|**0**|**0.9848** |**0.7836**|**0.8740**|

> **COR** - correct attributes in the extracted model<br/>
> **INC** - incorrect attributes in the extracted model<br/>
> **PAR** - partially correct attributes in the extracted model<br/>
> **MIS** - missing attributes in the extracted model<br/>
> **SPU** - spurious attributes in the extracted model

In a similar fashion to the [TurtleBot 2 case study](../../academic/TutleBot2/), many of the extraction issues are actually missing Links (61 in total, mostly parameter getters and setters) due to the use of the `dynamic_reconfigure` package.
There is also a single parameter whose value we able unable to determine, due to its value being the result of a dynamic call to the `xacro` utility.
Lastly, we have a few ROS names that we are no able to resolve fully.
Some of these names are the result of string concatenations, and at least one of the substrings is a dynamic value, set via ROS parameters.
HAROS is unable to resolve this, in its current version.
The purpose of this dynamic value is to promote code reuse with different robot versions (by changing the dynamic value that is set via ROS parameters).
Other names have a namespace component coming from member variables of a class.
For the moment, we assume that member variables can change anywhere in the code (especially if more threads are involved), and treat the value as dynamic.

In terms of extraction hints for this configuration, we ended up fixing 14 entities and creating 61 new entities, as shown in the project file and in the following table.

| Extraction Hints | Fixed Entities | Created Entities | YAML Lines |
| :--------------- | :------------: | :--------------: | :--------: |
| Parameter        | 1              | 0                | 3          |
| Topic Publisher  | 1              | 6                | 62         |
| Topic Subscriber | 0              | 0                | 0          |
| Service Client   | 3              | 0                | 6          |
| Service Server   | 0              | 3                | 24         |
| Param. Setter    | 0              | 26               | 208        |
| Param. Getter    | 9              | 26               | 226        |
| **Overall**      | **14**         | **61**           | **529**    |


### Property-based Testing

The AgRob V16 system has been used to evaluate the property-based test generation [plugin](https://github.com/git-afsantos/haros-plugin-pbt-gen) for HAROS.

#### Methodology

The property-based testing (PBT) technique used by this plugin must be able to reset the system under test (SUT) multiple times, to guarantee that, for each generated example, we run the SUT from a stable (and hopefully deterministic) state.
This series of resets occurs in rapid succession, and is entirely impractical if hardware is involved.
Thus, we opt to modify the tested configurations slightly (in particular, the *Basic* configuration) to remove any nodes that directly control hardware.
Removing nodes turns out to be beneficial for another reason; it opens up several topics that the plugin can exploit as inputs, versus a system that could, otherwise, be fully connected.

The plugin is evaluated on several fronts, and we used it to test not only the *Basic* configuration but also some individual nodes, such as the safety controller and the mission supervisor.
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

Instead of simply writing arbitrary properties of the SUT, we follow a systematic method to write specifications.
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

The original AgRob V16 launch files could not be used directly, as these included hardware-related nodes.
The modified versions we used (referred to as part of the `hpl_test` package) can be found in the [launch directory](./launch/).

#### Results

The safety controller node was tested with a specification consisting of 26 axioms, 5 properties and 54 mutants.
We achieved a perfect mutant score for this SUT.
Out of the 54 killed mutants, 3 were reported as flaky tests, i.e., the results were not deterministic.

The Joystick Controller node was tested with a specification consisting of 21 axioms, 3 properties and 29 mutants.
We achieved a very high mutant score for this SUT, failing to kill three mutants with a limit of 200 examples.
The mutants that we failed to kill are not related to each other, but share the same underlying problem: they all require producing specific floating-point values (exactly `-1.0`) in two specific indexes of an array of size 8.
These mutants are not different from many others that were successfully killed, in any measure of significance.
Thus, we believe that the cause was randomness; it just so happened that Hypothesis did not explore these specific values for these mutants within the maximum number of examples, while it did for other mutants.

The Supervisor node was tested with a specification consisting of 10 axioms, 4 properties and 36 mutants.
This SUT achieved a near-perfect mutant score, failing to kill two mutants with a limit of 200 examples.
The mutants that we failed to kill are related to the same property (although irrelevant) and require very specific messages to detect the counterexample, in input traces that are, at least, 3 messages long.

One of the properties of the Supervisor node (below) turned out to reveal two errors in the system (that have been reported to the developers and promptly fixed).

```python
globally: no /husky_velocity_controller/cmd_vel {angular.z < -0.5 or angular.z > 0.5}
```

This property states that the Supervisor does not allow any velocity commands whose angular velocities (the speed at which the robot turns) are higher than 0.5 meters per second.
The first error, discovered with this property, is the use of `else if` instead of `if`, in a certain part of the source code.
In the buggy version of the code, a message that contained both linear and angular velocities beyond the allowed limits would have its linear component correctly set to the maximum, but the angular component would remain unchecked.
In practice, this would lead to the robot turning around too fast.
The second error is that the condition over angular velocities only checked for positive angular velocities (i.e., to turn right).
The robot was allowed to turn left at speeds beyond the intended limit.

The full *Basic* configuration was tested with a specification consisting of 52 axioms, 4 properties and 48 mutants.
The axioms are essentially a collage of all axioms for individual nodes.
This SUT achieved a mutant score that is much lower than in previous cases, as shown in the following table.
Overall, the tool was unable to kill 11 mutants with a limit of 200 examples.
Although the surviving mutants are related to different properties, the respective counterexamples share a common structure and require a specific message ordering, in input traces that are 3 or 4 messages long.

| Metric          | Value   |
|:--------------- | :-----: |
| Mutants         | 48      |
| Flaky Tests     | 4       |
| False Negatives | 11      |
| Mutant Score    | 0.77083 |

Regarding other performance criteria, as per the following table, our results match the same scenarios as with testing the individual nodes, with a relatively large number of examples required to falsify properties.
On average, each mutant requires about 18 examples to falsify (including the shrinking phase), although the tool tends to find an error for the first time relatively early (within the first 2 examples).
Each example takes about 6 seconds to execute, and half of this time is spent on launching and tearing down the SUT between examples.
This means that the average time to kill a mutant and obtain a counterexample is two minutes or less.
We registered 5 outliers (besides the false negatives), taking about 400 examples to provide a verdict and raising the mean number of examples to 93.
Two of the outliers are flaky tests.
There are only four mutants for which the tool uses more messages than necessary, and two of them are flaky tests.

| Metric                      | Mean  | Std. Deviation | Median |
| :-------------------------- | :---: | :------------: | :----: |
| Number of examples          | 93    | 139            | 18     |
| Number of failures          | 32    | 72             | 14     |
| Examples to failure         | 41    | 64             | 1      |
| Invalid examples            | 0     | 0              | 0      |
| Shrink attempts             | 53    | 129            | 13     |
| Time per example (s)        | 7.094 | 2.893          | 5.903  |
| Trace duration (s)          | 4.073 | 2.083          | 2.965  |
| Setup/Teardown time (s)     | 3.021 | 0.810          | 2.937  |
| Input trace size (messages) | 2     | 2              | 1      |
| Excess Messages             | 0     | 1              | 0      |


### Model Checking

The AgRob V16 system has been used as a case study for the HAROS model checking [plugin](https://github.com/nmacedo/haros_plugin_mc), a plugin that verifies system-wide HPL properties in ROS applications (see [CarvalhoCMS:20](./CITING.md)).
The backend of this plugin relies on [Electrum](http://haslab.github.io/Electrum/), a formal specification language based on first-order linear temporal logic that extends Alloy.

Considering the *Basic* and *Path Planning* configurations of AgRob V16 as verification targets, a specification of 26 node properties and 4 system-wide safety properties was put together, in order to evaluate the plugin's approach.
Node-specific properties are used as axioms, to describe node behaviour, so that the model checker can focus on verifying the system-wide properties.
The following snippet shows two of the system-wide properties.

```python
globally: /agrobv16/current_state{ data[0] = 3 }
    requires /joy_teleop/joy { button[0] = 0 and button[1] = 1 }

globally:
    /husky/cmd_vel {
        linear.x = 0 and angular.z in [-100 to 100]
        and angular.z != 0
    }
    requires (/scan {ranges[0] in [0 to 4] }
              or /joy_teleop/joy { button[0] = 1 })
```

The first property states that `current_state` only contains the *follow path* operation mode (`data[0] = 3`) if the corresponding buttons have once been pressed in `joy`.
The second property states that `cmd_vel` messages to the base, to rotate in-place (`linear.x = 0 and angular.x != 0`) have been caused by a `scan` message with a range smaller than 40 cm (`ranges[0] in [0 to 4]`), or the system is in teleoperation mode (`button[0] = 1`).

All system-wide properties were checked for the two configurations with increasing scopes for values and messages, in a 2.4 GHz Intel Core i5 with 8GB memory, running the bounded model checking engine of Electrum with SAT4J.
One of the properties was actually shown not to hold for the *Path Planning* configuration due to the introduction of additional nodes: commands to rotate in-place may be published in *follow path* mode by the Path Planner node without obstacles being detected by the lasers, due to accumulated localization errors, which may wrongly identify dangerous situations.

The counterexample obtained with this analysis roughly describes an execution trace where a velocity command from the Path Planner would be identified by the Safety Controller as a dangerous situation, even without being flagged by the lasers, and thus instruct the base to rotate.
This counterexample is only found for scopes of over 5 values/messages, needed to create the messages of the respective execution trace.
The counterexample was found under 1 minute, and the other properties were verified to hold for traces with up to 10 values/messages under 2 minutes.
