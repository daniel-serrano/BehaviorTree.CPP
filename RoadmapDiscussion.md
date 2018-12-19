# 1. Roadmap: input/output ports in TreeNode

One of the goals of this project is to separate the role of the Component
Developer from the Behavior Designed and System Integrator.

As a consequence, in the contect of Behavior Trees, we want to write custom
ActionNodes and ConditionNodes __once__ and __never__ touch that source code again.

Using the same precompiled nodes, it should be possible to build any tree.

We realized that there is a major design flow that undermines this goal: the way
dataflow between nodes is done, i.e. using the BlackBoard.


As described in [issue #18](https://github.com/BehaviorTree/BehaviorTree.CPP/issues/18)
there are several issues:

- To know which entries of the BB are read/written, you should read the source code.
- As a consequence, external tools such as __Groot__ have no idea of which BB entries are accessed.
- If there is a name clashing (multiple nodes use the same key for different purposes),
 the only way to solve that is modifying the source code. 

SMACH solved this problem using [input and output ports](http://wiki.ros.org/smach/Tutorials/User%20Data)
and remapping to connect them.

# 2. Suggested changes

Goals of the new design:

- The [TreeNodeManifest](https://github.com/BehaviorTree/BehaviorTree.CPP/blob/master/include/behaviortree_cpp/bt_factory.h#L33)
should contain information about input and outputs ports, to make this information available
to external tools.

- Avoid name clashing using key remapping.

- We want to solve the previous problems but trying to keep the API as consistent
as possible with the previous one.

## 2.1 Deprecate TreeNode::blackboard()

Accessing directly the BB allows the users to do whatever they wants.
There is no way to introspect which entries are accessed.

Therefore, the only reasonable thing to do is to deprecate `TreeNode::blackboard()`

The problem is that `SimpleActionNodes` and `SimpleDecoratorNodes` 
will loose the ability to access ports.

## 2.2 Solution A

###  2.2.1 NameParameters as input ports

We know that NodeParameters are a mechanism to add "arguments" to a Node.

It is possible to point to the entry of the BB, instead of parsing a static value.
After few months, it became clear that this is the rule rather than the exception.

In probably 80-90% of the cases, NodeParameters are passed through the BB.

Furthermore, `requiredNodeParameters` is already an effective way to 
automatically create a manifest.

As a consequence, we may consider NodeParameters a valid implementation of an
__input port__.

From a practical point of view, we should encourage the use of
`TreeNode::getParam` as much as possible and deprecate `TreeNode::blackboard()::get`

### 2.2.1 Output Ports

We need to add automatically the output ports to the TreeNodeManifest.

To do that, we can just add the static method

       const std::set<std::string>& providedOutputPorts() //outputs
      
for consistency, we might consider to change the signature of  `requiredNodeParameters()` to
   
       const std::set<std::string>& requiredNodeParameters() //inputs

In other words, requiredNodeParameters provides only the key, but not a default value;
in fact, we have seen that there is little practical use for a default value.

The new manifest definition would become:

```c++
struct TreeNodeManifest
{
    NodeType type;
    std::string registration_ID;
    std::set<std::string> required_parameters;
    std::set<std::string> provided_outputs;
};
```

About remapping, to avoid name cashing it is sufficient to provide remapping
at run-time __only for the output ports__.

We don't need remapping of input ports, because the name of the entry is 
already provided at run-time (in the XML).

From the user prospective, `TreeNode::blackboard()::set(key,value)` is replaced by a new method
`TreeNode::setOutput(key,value)`.

Example:

If the remapping __["goal","navigation_goal"]__ is passed and the user invokes

      setOutput("goal", "kitchen");

The actual entry to be written will be the `navigation_goal`.


## 2.3 Solution B

An alternative solution is to make no distintion between input and output ports.

This would make the code more consistent with the old one, but would break the API.

### 2.3.1 New manifest 

```c++

enum PortType { INPUT, OUTPUT, INOUT };

typedef std::unordered_map<std::string, PortType> PortsList;

// New Manifest
struct TreeNodeManifest
{
    NodeType type;
    std::string registration_ID;
    PortsList ports;
};

// What was previously MyNode::requiredNodeParameters() becomes:

static const PortsList& MyNode::providedPorts();

```

In other words, requiredNodeParameters, which used to focus only on inputs,
is substituted by another static method that provides both inputs and outputs.

### 2.3.1 from XML attributes to ports in/out/remaping 

Let's illustrate this change with a practical example.

In this example __path__ is an output port in `ComputePath` but an input port
in `FollowPath`.

```XML
    <SequenceStar name="navigate">
        <Action ID="ComputePath" endpoints="${navigation_endpoints}" path="${navigation_path}"  />
        <Action ID="FollowPath"  path="${navigation_path}" />
    </SequenceStar>
```

You may notice that no distinction is made in the XML between inputs and outputs.

The actual entries to be read/written are the one specified in the remapping:

 - navigation_endpoints
 - navigation_path 

Since these names are specified in the XML, name clashing can be avoided without 
modifying the source code.

The C++ code might be:

```C++
class ComputePath: public SyncActionNode
{
  public:
    ComputePath(const std::string& name, const NodeParameters& params): 
        SyncActionNode(name, params){}

    NodeStatus tick() override
    {
        auto end_points = getParam<EndPointsType>("endpoints");
        // do your stuff
        setOutput("path", my_computed_path);
        // return result...
    }
    
    static const PortsList& providedPorts()
    {
        static PortsList ports_list = { {"endpoints", INPUT}, 
                                        {"path",      OUTPUT} };
        return ports_list;
    } 
};

class FollowPath: public AsyncActionNode
{
  public:
    FollowPath(const std::string& name, const NodeParameters& params): 
        AsyncActionNode(name, params){}

    NodeStatus tick() override
    {
        auto path = getParam<PathType>("path");
        // do your stuff
        // return result...
    }
    
    static const PortsList& providedPorts()
    {
        static PortsList ports_list = { {"path", INPUT} };
        return ports_list;
    } 
};
```

# 3. Further changes

### Major (breaking) changes in the signature of TreeNodes

__Under development...__

It might make sense to change the signature of the TreeNode constructor from:

    TreeNode(const string& name, const NodeParameters& params)

to:

    TreeNode(const string& name, const NodeConfiguration& config) 

where:

```c++
struct NodeConfiguration
{
    // needed to register this in the constructor
    BlackBoard::Ptr blackboard;
    
    // needed to register this in the constructor 
    std::string registration_ID;
    
    // input/output parameters. Might be strings or pointers to BB entries
    NodeParameters parameters;
};
```

This would solve multiple problems, including:

- The fact that BB are not available in the constructor.
- Potential errors when `setRegistrationName()` in not called.
