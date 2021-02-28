# AgRob Path Planner

## System

The [AgRob Path Planner](https://gitlab.inesctec.pt/agrob/agrob_pp/) (AgRobPP) is an open-source ROS-based framework for path planning in agricultural terrains, specifically for steep slope vineyards.
AgRobPP was developed within the scope of the [RoMoVi project](https://www.inesctec.pt/en/projects/romovi#about), and was published by the [ROSIN European Project](https://www.rosin-project.eu/) under the [AgRobIT FTP](https://www.rosin-project.eu/ftp/agrobit-mowing-robot).
Typically, these vineyards are located along hills with irregular and sloped terrains representing a challenge for the path planning task; the ground inclination and robot's center of mass need to be considered during the path planning operation.
Path planning operations may also be affected by the agricultural fields' dimensions, as a lot of computational memory could be required.
For example, a steep slope vineyard from a small producer has an area of about 1 hectare, but larger farms span up to 70 hectares.

AgRobPP is composed of three main components:

1. AgRob Vineyard Detector - a satellite image segmentation tool (see [SantosSFS:19](./CITING.md), [SantosASVP:20](./CITING.md));
2. AgRob Topologic Mapper - a tool to construct a topological map (see [SantosSM0R:19](./CITING.md), [SantosASVP:20](./CITING.md));
3. AgRob Path Planner - a path planning framework for uneven terrains (see [SantosSMCLRS:20](./CITING.md)).

AgRob Vineyard Detector is a machine learning tool based in Support Vector Machine (SVM), designed to detect vineyard lines and extract an occupation grid map without a previous robot visit to the terrain.
AgRob Topologic Mapper takes this grid map and divides it into smaller zones, saving them in a graph structure.
AgRob Path Planner is an A* search algorithm that navigates in the free cells of an occupation grid map, considering the robot's orientation and center of mass.
Combined with AgRob Topologic Mapper, the search space gets reduced to the map's strictly necessary zones, making it more efficient in terms of memory.

We have focused our analysis mostly on a minimalistic configuration of this system that includes the path planner, a map server and a pose generator.

## Versioning

We analysed the [most up-to-date](https://gitlab.inesctec.pt/agrob/agrob_pp/-/commit/f57d4efd05e50bc5bf68442c99d4fb96cd806e45) source code under ROS Melodic.
We used the most recent version of HAROS at the time of writing (v3.9).

## Analysis

The AgRob PP system has been analysed in terms of:

- compliance with coding standards;
- compliance with code quality metrics standards;
- automatic model extraction;
- architectural queries;
- property-based testing (and runtime verification).

In the following subsections we provide further details on our approach to the various themes.
The files found under the [`json` directory](./json/) can be copied to the HAROS viz's `data` directory for visualisation.

### Code Quality Analysis

We ran a standard code quality analysis over the source code, including both compliance with coding standards and internal quality metrics.

The analysis found over 7600 issues, most of which related to coding standards and code formatting.
Many of the metrics-related issues are instances of functions that are too long or too complex (as per the definition of cyclomatic complexity).

The following table sums up the analysis and its results.

| Metric            | Value |
| :---------------- | :---: |
| Packages          | 3     |
| C++ lines         | 24.9K |
| Analysis plugins  | 6     |
| Total rules       | 150   |
| Total issues      | 7636  |
| Coding issues     | 7389  |
| Metrics issues    | 444   |

Our approach to the code formatting issues was simply to discard them.
HAROS checks conformity with both the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html) and the [ROS C++ Style Guide](http://wiki.ros.org/CppStyleGuide).
In this case, we followed neither.
Instead, we used the [ClangFormat tool](https://clang.llvm.org/docs/ClangFormat.html) to automatically handle formatting, using a configuration provided by the [ROS Industrial repositories](https://github.com/ros-industrial/industrial_calibration/blob/kinetic-devel/.clang-format).
By ignoring this sub-category, we effectively had to handle 3248 issues.

The following rules identify the most recurrent issues.

> - *Include all required headers for what you use.*
> - *Do not use integer types directly. Use size-specific `typedef`s, for instance from `<cstdint>`.*
> - *Maximum number of function lines of code of 40.*

The first rule is a simple yet solid practice that might be overlooked without static analysis tools.
The code in question compiled successfully because the necessary libraries were transitively included in headers AgRob PP depended on.
Should these headers drop the transitive dependencies in a future update, the code base would no longer build.
The second rule advises against the use of platform-dependent integer types, e.g., `int`.
It recommends the use of fixed-size types, such as `int32_t`, which bolster the portability of the project.
All occurrences of `int` were manually replaced with an appropriately-sized integer type.
This is a typical example of a tedious fix in the later stages of development.
Had the analysis tools been part of the development process from the start, the `int` type would likely not have been used at all.
Lastly, we have a metrics issue -- functions that are over 40 lines of code are deemed too long.
Fixing this issue after-the-fact requires care, so that refactoring the code does not introduce new bugs.
However, this refactoring process led us (indirectly) to uncover a critical fault in the path planner.
Plans are published under two representations (that should always be equivalent): an array of poses (discrete), and an array of parametric curves (continuous).
In some cases, the two outputs diverged.

Overall, after some back-and-forth iterations over the course of two weeks, we were able to go from 3248 non-formatting issues down to 1302 -- a reduction of 60%.

### Model Extraction and Architectural Analysis

Our architectural analysis of this system did not find any relevant issues.
Manual inspection of the models, coupled with several months of prior testing of the system had already ruled out orchestration mistakes.
Regardless, we bolstered our confidence in the system with a small catalog of 14 queries, applied across all architectures, matching problematic patterns such as:

- use of global ROS names;
- use of conditional topics, services or parameters;
- message type mismatches between publishers and subscribers;
- topics with unbounded message queues;
- topics with multiple publishers;
- missing publishers for starting or goal poses.

Should any future changes or additions to the system violate any of these rules, there will be an automatic check in place.


### Property-based Testing

The AgRob PP system was subjected to property-based tests using the property-based test generation [plugin](https://github.com/git-afsantos/haros-plugin-pbt-gen) for HAROS.

Our approach was based on building a dependability case for a high-level property: *after receiving and loading a map, given valid starting and goal poses in this map, the AgRob Path Planner shall produce a valid plan from the starting pose to the goal, if a path between the two exists.*
This is a complex property, spanning the whole system, that we have to break down into smaller, more manageable pieces.
After gathering evidence that each sub-property holds, we can argue that the initial, high-level property does as well.

Overall, the breakdown of a single high-level property led us to specify 30 HPL properties (see the [HPL file](./hpl/agrob_pp.hpl)).
But note that this is not a full specification of the AgRob Path Planner.
The node has other features that have not been captured with this high-level property.
Specification for these features is still ongoing work.
We estimate that, for the 30 testable properties we have so far, it took about two weeks of fine-tuning and on-and-off discussion between roboticists and software engineers until both parties were satisfied.

We could test the path planning node in isolation (a unit test), but, in this case, it is advantageous to test the full architecture (including the map server and the pose generator) at once.
Unit tests would delegate the generation of maps and poses to the test infrastructure.
Generating random maps is of little value (too much noise), and specifying rules for realistic maps takes way too much effort.
It is simpler to include the `map_server`, and repeat the process for different maps.

After spending a full working day (about 8 hours) on testing, discussion, and interpretation of test results, we have found 2 properties that needed further fine-tuning (false assumptions), and 4 bugs without immediate critical effects.

**Bug #1:**
In this simple configuration, some of the published topics of the AgRob Path Planner should not be used.
By stimulating the right inputs, the testing infrastructure found instances in which unexpected messages could be observed.

**Bug #2:**
The planner should only transition to the `PLANNING` state once it receives a starting pose and a goal pose in the `READY` state.
The lack of proper pre-conditions in the message handler meant that poses were always being processed, even, e.g., when loading the map.
Thus, poses sent prior to entering the `READY` state would still trigger a transition to `PLANNING`.

**Bug #3:**
Under some circumstances, a plan would be published, but the planner would not enter the `PLANNING_SUCCESSFUL` state.

**Bug #4:**
We expected no plan to be published when on the `PLANNING_FAILED` state, but the planner did publish empty plans.

