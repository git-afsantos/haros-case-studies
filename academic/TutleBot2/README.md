# TurtleBot 2

## System

The TurtleBot 2 robot ([Home](https://www.turtlebot.com/turtlebot2/), [GitHub](https://github.com/turtlebot)) is a low-cost mobile personal robot designed for education and research on state of the art robotics.

This system builds on top of the Kobuki mobile base.
Among other features, it adds:

- a depth camera component;
- integration with the ROS navigation stack;
- multiple applications for demonstration purposes.

We have analysed multiple configurations of this system but our main focus has been on the *AMCL* configuration, that makes for a demonstration of autonomous navigation.

## Versioning

We analysed the most up-to-date source code under ROS Indigo, ROS Kinetic and ROS Melodic.
Various versions of HAROS were used, from 3.0 to 3.9.

## Analysis

Our analysis of TurtleBot 2 is mostly a step up from the analyses we performed with [Kobuki](../Kobuki/).
We refer the reader to the Kobuki case study summary, and will highlight in this document only some relevant differences.

The files found under the [`json` directory](./json/) can be copied to the HAROS viz's `data` directory for visualisation.

### Model Extraction and Architectural Analysis

We extracted models of various TurtleBot 2 configurations, ranging from individual nodes to full systems, such as the *AMCL* configuration.
We then used these models for various purposes, such as evaluation of the model extractor's analysis and architectural analysis using a small catalogue of rules.

Here we detail the results of our evaluation of HAROS' model extractor, using the *AMCL* configuration and the [Model Graph Extraction Difference plug-in](https://github.com/git-afsantos/haros-plugin-model-ged) (see the [project file](./projects/model-perf.yaml)).

To run this analysis we used the following command.

```bash
haros analyse -w haros_plugin_model_ged --env --no-hardcoded -n -p model-perf.yaml
```

Overall, we have found that, without specifying extraction hints, HAROS is able to extract **94.50%** of the true model correctly (*F1-score*), with a *precision* of **99.35%** and a *recall* of **90.11%**, as shown in the following table.

| Entity      | COR | INC | PAR | MIS | SPU | Precision | Recall | F1-score |
|:---         |:---:|:---:|:---:|:---:|:---:|   :---:   |  :---: |  :---:   |
| Node        | 126 | 0   | 0   | 0   | 0   | 1.0000    | 1.0000 | 1.0000   |
| Parameter   | 1218| 2   | 0   | 0   | 0   | 0.9984    | 0.9984 | 0.9984   |
| Publisher   | 158 | 2   | 0   | 42  | 1   | 0.9814    | 0.7822 | 0.8705   |
| Subscriber  | 117 | 1   | 1   | 18  | 1   | 0.9792    | 0.8577 | 0.9144   |
| Client      | 0   | 0   | 0   | 0   | 0   | 1.0000    | 1.0000 | 1.0000   |
| Server      | 0   | 0   | 0   | 10  | 0   | 1.0000    | 0.0000 | 0.0000   |
| Setter      | 0   | 0   | 0   | 42  | 0   | 1.0000    | 0.0000 | 0.0000   |
| Getter      | 134 | 3   | 0   | 72  | 1   | 0.9710    | 0.6411 | 0.7723   |
| **Overall** |**1753**|**8**|**1**|**184**|**3**|**0.9935** |**0.9011**|**0.9450**|

> **COR** - correct attributes in the extracted model<br/>
> **INC** - incorrect attributes in the extracted model<br/>
> **PAR** - partially correct attributes in the extracted model<br/>
> **MIS** - missing attributes in the extracted model<br/>
> **SPU** - spurious attributes in the extracted model

A large part of this configuration is shared with the *Safe Random Walker* from Kobuki, so the previous issues are also present.
In addition, there are a few more uses of `dynamic_reconfigure`, increasing the number of missed Links, there are more subscribers to set up in the multiplexer node, and launch file parsing is no longer perfect.
We now have a parameter, defined in a launch file, whose value is the text output of a command-line application that is supposed to run in place -- `xacro`, a XML macro utility that is used in ROS to read XML descriptions of physical robot models.
HAROS does not support this feature of running arbitrary commands during launch file interpretation, so it is not capable of determining the parameter's value.

In terms of extraction hints for this configuration, we ended up fixing 7 entities and creating 30 new entities, as shown in the following table.

| Extraction Hints | Fixed Entities | Created Entities | YAML Lines |
| :--------------- | :------------: | :--------------: | :--------: |
| Parameter        | 1              | 0                | 3          |
| Topic Publisher  | 1              | 6                | 61         |
| Topic Subscriber | 1              | 3                | 30         |
| Service Client   | 0              | 0                | 0          |
| Service Server   | 0              | 2                | 16         |
| Param. Setter    | 0              | 7                | 56         |
| Param. Getter    | 4              | 12               | 104        |
| **Overall**      | **7**          | **30**           | **270**    |
