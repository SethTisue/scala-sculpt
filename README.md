# Sculpt: dependency graph extraction for Scala

Sculpt is a compiler plugin for analyzing the dependency structure of
Scala source code.

## Project status

This is **unfinished**, **unmaintained** software.  We are releasing
it as open source as a public service with the hopes the code will be
useful to someone.

Sculpt is NOT supported under the Lightbend subscription.

## What is it for?

The data generated by the plugin should be useful for all sorts of
refactoring efforts, including carving a monolithic codebase into
independent subprojects.

The plugin analyzes source code, not generated bytecode. The analysis
code is based on code from the incremental compiler in sbt and zinc.
Therefore, the plugin should be an accurate source of information for
developers looking to reduce dependencies in order to reduce
incremental compile times.

## Building the plugin from source

`sbt assembly` will create `target/scala-2.11/scala-sculpt_2.11-0.1.4.jar`.
(The JAR is a fat JAR that bundles its dependency on spray-json.)

## Using the plugin

You can use the compiled plugin with the Scala 2.11 compiler as follows.

Supposing you have `scala-sculpt_2.11-0.1.4.jar` in your current working directory,

Then you can do e.g.:

    scalac -Xplugin:scala-sculpt_2.11-0.1.4.jar \
      -Xplugin-require:sculpt \
      -P:sculpt:out=dep.json \
      Dep.scala

## Sample input and output

Assuming `Dep.scala` contains this source code:

    object Dep1 { val x = 42; val y = Dep2.z }
    object Dep2 { val z = Dep1.x }

then the command line shown above will generate this `dep.json` file:

    [
      {"sym": ["o:Dep1"], "extends": ["pkt:scala", "tp:AnyRef"]},
      {"sym": ["o:Dep1", "def:<init>"], "uses": ["o:Dep1"]},
      {"sym": ["o:Dep1", "def:<init>"], "uses": ["pkt:java", "pkt:lang", "cl:Object", "def:<init>"]},
      {"sym": ["o:Dep1", "def:x"], "uses": ["o:Dep1", "t:x"]},
      {"sym": ["o:Dep1", "def:x"], "uses": ["pkt:scala", "cl:Int"]},
      {"sym": ["o:Dep1", "def:y"], "uses": ["o:Dep1", "t:y"]},
      {"sym": ["o:Dep1", "def:y"], "uses": ["pkt:scala", "cl:Int"]},
      {"sym": ["o:Dep1", "t:x"], "uses": ["pkt:scala", "cl:Int"]},
      {"sym": ["o:Dep1", "t:y"], "uses": ["o:Dep2", "def:z"]},
      {"sym": ["o:Dep1", "t:y"], "uses": ["ov:Dep2"]},
      {"sym": ["o:Dep1", "t:y"], "uses": ["pkt:scala", "cl:Int"]},
      {"sym": ["o:Dep2"], "extends": ["pkt:scala", "tp:AnyRef"]},
      {"sym": ["o:Dep2", "def:<init>"], "uses": ["o:Dep2"]},
      {"sym": ["o:Dep2", "def:<init>"], "uses": ["pkt:java", "pkt:lang", "cl:Object", "def:<init>"]},
      {"sym": ["o:Dep2", "def:z"], "uses": ["o:Dep2", "t:z"]},
      {"sym": ["o:Dep2", "def:z"], "uses": ["pkt:scala", "cl:Int"]},
      {"sym": ["o:Dep2", "t:z"], "uses": ["o:Dep1", "def:x"]},
      {"sym": ["o:Dep2", "t:z"], "uses": ["ov:Dep1"]},
      {"sym": ["o:Dep2", "t:z"], "uses": ["pkt:scala", "cl:Int"]}
    ]

Each line in the JSON file represents an edge between two symbols in a
dependency graph.

The edges are of two types, `extends` and `uses`.

Each symbol is represented in the JSON as an array of strings, where
each string represents a part of the symbol's fully qualified name.

So for example, in the above source code, we see that `Dep1` extends
`scala.AnyRef`:

    {"sym": ["o:Dep1"], "extends": ["pkt:scala", "tp:AnyRef"]},

And we see that `Dep1` uses `scala.Int` in three places:

    {"sym": ["o:Dep1", "def:x"], "uses": ["pkt:scala", "cl:Int"]},
    {"sym": ["o:Dep1", "def:y"], "uses": ["pkt:scala", "cl:Int"]},
    {"sym": ["o:Dep1", "t:x"], "uses": ["pkt:scala", "cl:Int"]},

from this we see that `scala.Int` is used as the return type of
`Dep1.x` and `Dep1.y`, and as the inferred type of the body of
`Dep1.y`.

For brevity, the following abbreviations are used in the JSON output:

### Terms

abbreviation | meaning
-------------|--------
ov           | object
def          | def
var          | var
mac          | macro
pk           | package
t            | other term

### Types

abbreviation | meaning
-------------|--------
tr           | trait
pkt          | package
o            | object
cl           | class
tp           | other type

### Other

The name of a constructor is always `<init>`.

## Running in "class mode"

The dependency information produced by the default mode is extremely
fine-grained; it goes all the way down to the level of individual
methods.

If you prefer an aggregated higher-level summary, you can run Sculpt
in "class mode" by adding `-P:sculpt:mode=class`. So e.g. a complete
invocation would look like:

    scalac -Xplugin:scala-sculpt_2.11-0.1.4.jar \
      -Xplugin-require:sculpt \
      -P:sculpt:out=classes.json \
      -P:sculpt:mode=class \
      Dep.scala

on the same source code used in the example above, this command line
generates this `classes.json` file:

    [
      {"sym": ["o:Dep1"], "uses": ["o:Dep2"]},
      {"sym": ["o:Dep1"], "uses": ["pkt:java", "pkt:lang", "cl:Object"]},
      {"sym": ["o:Dep1"], "uses": ["pkt:scala", "cl:Int"]},
      {"sym": ["o:Dep1"], "uses": ["pkt:scala", "tp:AnyRef"]},
      {"sym": ["o:Dep2"], "uses": ["o:Dep1"]},
      {"sym": ["o:Dep2"], "uses": ["pkt:java", "pkt:lang", "cl:Object"]},
      {"sym": ["o:Dep2"], "uses": ["pkt:scala", "cl:Int"]},
      {"sym": ["o:Dep2"], "uses": ["pkt:scala", "tp:AnyRef"]}
    ]

Note that all of the nodes are top-level classes, traits, objects, or
type aliases, and all of the edges are of type "uses".

`-P:sculpt:mode=class` is provided as a convenience, but it isn't
strictly needed, in that if you have already run Sculpt in default
mode, you can convert detailed dependencies to class-level
dependencies in the course of an interactive session.  This
is demonstrated in the sample interactive session below.

## Graphs represented as case classes

The same JAR that contains the plugin also contains a suite of case
classes for representing the same information in the JSON files as
Scala objects.

We provide a `load` method for parsing a JSON file into instances
of these case classes, and a `save` method for writing the instances
back out to JSON.

These classes provide a possible starting point for graph analysis and
manipulation, e.g. in the REPL.

### Sample interactive session

Now in a Scala 2.11 REPL with the same JARs on the classpath:

    scala -classpath scala-sculpt_2.11-0.1.4.jar

If we load `dep.json` as follows, we'll see the following graph:

    scala> import com.lightbend.tools.sculpt.cmd._
    import com.lightbend.tools.sculpt.cmd._

    scala> load("dep.json")
    res0: com.lightbend.tools.sculpt.model.Graph = Graph 'dep.json': 15 nodes, 19 edges

    scala> println(res0.fullString)
    Graph 'dep.json': 15 nodes, 19 edges
    Nodes:
      - o:Dep1
      - pkt:scala.tp:AnyRef
      ...
    Edges:
      - o:Dep1 -[Extends]-> pkt:scala.tp:AnyRef
      - o:Dep1.def:<init> -[Uses]-> o:Dep1
      ...

#### Converting to class-level dependencies

If we're interested in class-level dependencies only, we can
call `load` with `classMode = true` in order to aggregate the
dependencies after loading:

    scala> load("dep.json", classMode = true)
    res2: com.lightbend.tools.sculpt.model.Graph = Graph 'dep.json': 7 nodes, 10 edges

#### Cycles and layers reports

When untangling dependencies, circular dependencies are always
especially problematic. We can identify these, list their contents,
sort them by the total number of classes in the cycle, or print
them grouped into layers according to their dependency structure.

The cycles and layers reports operate on class-level dependencies
only, so you must either run the plugin in "class mode", or convert
from default mode to class mode at load time:

Continuing the running example, here's a cycles report:

    scala> import com.lightbend.tools.sculpt.model.Cycles

    scala> println(Cycles.cyclesString(res2.nodes))
    [2] o:Dep1 o:Dep2

The report shows that the codebase contains a single cycle of size 2,
because `Dep1` and `Dep2` mutually reference each other.  ("Cycles" of
a single node are omitted.)

And here's the layers report for the same code:

    scala> println(res2.layersString)
    layers =
      """|[1] o:Dep1 o:Dep2
         |[0] cl:java.lang.Object
         |[0] cl:scala.Int
         |[0] tp:scala.AnyRef

The numbers are layer numbers, defined as follows:

* layer 0: classes with no dependencies
* layer 1: classes with only layer 0 dependencies
* layer 2: classes with only layer 0 and 1 dependencies
* ...

Note that some concepts of layered architectures require that layer n
accesses only layer n - 1 and not any lower layers; we are not making
that assumption here.

Here's an example portion of a cycle report for a larger sample codebase:

    [8] tr:api.Agent tr:api.AgentSet tr:api.Link tr:api.Observer tr:api.Patch tr:api.TrailDrawerInterface tr:api.Turtle tr:api.World
    [5] cl:workspace.AbstractWorkspace cl:workspace.DefaultFileManager cl:workspace.Evaluator o:workspace.AbstractWorkspaceTraits o:workspace.Benchmarker
    [4] cl:agent.HorizCylinder cl:agent.Torus cl:agent.VertCylinder o:agent.Topology
    [3] cl:agent.AgentSet cl:agent.ArrayAgentSet o:agent.AgentSet

(The numbers are cycle sizes.)

And here's part of the layer report for the same codebase:

    [14] o:org.nlogo.headless.Main
    [14] o:org.nlogo.headless.Shell
    [13] o:org.nlogo.compile.middle.FrontMiddleBridge
    [13] o:org.nlogo.headless.HeadlessWorkspace
    [13] o:org.nlogo.mirror.ModelRunIO
    [12] o:org.nlogo.compile.back.BackEnd
    [12] o:org.nlogo.compile.middle.MiddleEnd

showing just the topmost layers of the application.

#### Modifying the graph

We can explore the effect of removing edges from the graph using `removePaths`:

    scala> res0.removePaths("Dep2", "java.lang")

    scala> println(res0.fullString)
    Graph 'dep.json': 9 nodes, 8 edges
    Nodes:
      - o:Dep1
      - pkt:scala.tp:AnyRef
      - o:Dep1.def:<init>
      - o:Dep1.def:x
      - o:Dep1.t:x
      - pkt:scala.cl:Int
      - o:Dep1.def:y
      - o:Dep1.t:y
      - ov:Dep1
    Edges:
      - o:Dep1 -[Extends]-> pkt:scala.tp:AnyRef
      - o:Dep1.def:<init> -[Uses]-> o:Dep1
      - o:Dep1.def:x -[Uses]-> o:Dep1.t:x
      - o:Dep1.def:x -[Uses]-> pkt:scala.cl:Int
      - o:Dep1.def:y -[Uses]-> o:Dep1.t:y
      - o:Dep1.def:y -[Uses]-> pkt:scala.cl:Int
      - o:Dep1.t:x -[Uses]-> pkt:scala.cl:Int
      - o:Dep1.t:y -[Uses]-> pkt:scala.cl:Int

Saving the graph back to a JSON model and loading it again:

    scala> save(res0, "dep2.json")

    scala> load("dep2.json")
    res5: com.lightbend.tools.sculpt.model.Graph = Graph 'dep2.json': 8 nodes, 8 edges

    scala> println(res5.fullString)
    Graph 'dep2.json': 8 nodes, 8 edges
    Nodes:
      - o:Dep1
      - pkt:scala.tp:AnyRef
      - o:Dep1.def:<init>
      - o:Dep1.def:x
      - o:Dep1.t:x
      - pkt:scala.cl:Int
      - o:Dep1.def:y
      - o:Dep1.t:y
    Edges:
      - o:Dep1 -[Extends]-> pkt:scala.tp:AnyRef
      - o:Dep1.def:<init> -[Uses]-> o:Dep1
      - o:Dep1.def:x -[Uses]-> o:Dep1.t:x
      - o:Dep1.def:x -[Uses]-> pkt:scala.cl:Int
      - o:Dep1.def:y -[Uses]-> o:Dep1.t:y
      - o:Dep1.def:y -[Uses]-> pkt:scala.cl:Int
      - o:Dep1.t:x -[Uses]-> pkt:scala.cl:Int
      - o:Dep1.t:y -[Uses]-> pkt:scala.cl:Int

## Future work

Possible future directions include:

* aggregation of dependency data at higher "zoom levels" (per-package, per-source-file)
* user interface (perhaps via IDE integration)
* automatic identification of problematic dependencies
* “what-if” analyses exploring the effect of proposed code changes
* offer a means of declaring and enforcing desired architectural constraints (allowed and forbidden dependencies)

## Similar/related work

* https://github.com/matanster/extractor
