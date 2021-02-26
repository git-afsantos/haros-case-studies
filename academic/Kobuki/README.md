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
In particular, we used this configuration in order to evaluate the performance of the HAROS model extractor, using the [Model Graph Extraction Difference plug-in](https://github.com/git-afsantos/haros-plugin-model-ged) (see the [project file](./projects/model-perf.yaml)).

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

In this case, we missed the extraction of Links that come from composite primitives, provided by packages other than the base ROS client libraries (not yet included in the parsing capabilities of HAROS).
For instance, we missed Links (full entities, with multiple attributes) created by the `tf`, `diagnostics` and `dynamic_reconfigure` packages.
The `dynamic_reconfigure` case, in particular, is a recurrent issue that lowers the metrics considerably, as the use of this package entails at the very least 5 different Links, with a single line of C++ code.
We also missed subscriptions of the multiplexer node, which are dynamically created, in a loop, after reading subscriber data from a parameter YAML file.
Lastly, we missed the default values (attributes) for three parameters that are defined in constants of a C++ implementation file (not a header) that is not part of the analysed source code (i.e., we only consider the binaries of that package).
All issues are related to C++ parsing; launch file parsing for this configuration was perfect.

In terms of extraction hints for this configuration, we ended up fixing 6 entities and creating 11 new entities, as shown in the project file and in the following snippet.

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
                        -   parameter: /mobile_base/battery_capacity
                            default_value: 16.5
                        -   parameter: /mobile_base/battery_low
                            default_value: 14.0
                        -   parameter: /mobile_base/battery_dangerous
                            default_value: 13.2
                        -   parameter: /mobile_base/diagnostic_period
                            create: true
                            param_type: double
                            traceability:
                                package: kobuki_node
                                file: src/library/kobuki_ros.cpp
                                line: 81
                                column: 1
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
                        -   topic: /cmd_vel_mux/?
                            original_name: /cmd_vel_mux/safety_controller
                            conditional: false
                        -   topic: /cmd_vel_mux/random_walker
                            create: true
                            msg_type: geometry_msgs/Twist
                            queue_size: 10
                            traceability:
                                package: yocs_cmd_vel_mux
                                file: src/cmd_vel_mux_nodelet.cpp
                                line: 207
                                column: 11
                    servers:
                        -   service: /cmd_vel_mux/set_parameters
                            create: true
                            srv_type: dynamic_reconfigure/Reconfigure
                            traceability:
                                package: yocs_cmd_vel_mux
                                file: src/cmd_vel_mux_nodelet.cpp
                                line: 96
                                column: 32
                    setters:
                        -   parameter: /cmd_vel_mux/yaml_cfg_file
                            create: true
                            param_type: str
                            traceability:
                                package: yocs_cmd_vel_mux
                                file: src/cmd_vel_mux_nodelet.cpp
                                line: 96
                                column: 32
                        -   parameter: /cmd_vel_mux/yaml_cfg_data
                            create: true
                            param_type: str
                            traceability:
                                package: yocs_cmd_vel_mux
                                file: src/cmd_vel_mux_nodelet.cpp
                                line: 96
                                column: 32
                    getters:
                        -   parameter: /cmd_vel_mux/yaml_cfg_file
                            conditional: false
                        -   parameter: /cmd_vel_mux/yaml_cfg_file
                            create: true
                            param_type: str
                            traceability:
                                package: yocs_cmd_vel_mux
                                file: src/cmd_vel_mux_nodelet.cpp
                                line: 96
                                column: 32
                        -   parameter: /cmd_vel_mux/yaml_cfg_data
                            create: true
                            param_type: str
                            traceability:
                                package: yocs_cmd_vel_mux
                                file: src/cmd_vel_mux_nodelet.cpp
                                line: 96
                                column: 32
```

Note that the creation of a single entity via hints fixes multiple missed attributes in the previous table.
For instance, we can see from the snippet above that we have created 4 Publishers.
Each Publisher corresponds to 7 attributes, namely `topic`, `msg_type`, `queue_size`, `package`, `file`, `line` and `column`.
Thus, 4 entities with 7 attributes each, yields the 28 missed Publisher attributes in the initial report.


### Property-based Testing

### Model Checking

