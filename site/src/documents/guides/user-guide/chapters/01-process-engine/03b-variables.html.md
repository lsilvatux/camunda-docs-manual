---

title: 'Process Variables'
category: 'Process Engine'

---

This section describes the concepts of variables in processes. Variables can be used to add data to process runtime state or, more particular, variable scopes. Various API methods that change the state of these entities allow updating of the attached variables. In general, a variable consists of a name and a value. The name is used for identification across process constructs. For example, if one activity sets a variable named *var*, a follow-up activity can access it by using this name. The value of a variable is a Java object.

## Variable Scopes and Variable Visibility

All entities that can have variables are called *variable scopes*. These are executions (which include process instances) and tasks. As described in the  [Concepts section](ref:#process-engine-process-engine-concepts-executions), the runtime state of a process instance is represented by a tree of executions. Consider the following process model where the red dots mark active tasks:

<center><img class="img-responsive" src="ref:asset:/guides/user-guide/assets/img/variables-3.png" /></center>

The runtime structure of this process is as follows:

<center><img class="img-responsive" src="ref:asset:/guides/user-guide/assets/img/variables-4.png" /></center>

There is a process instance with two child executions, each of which has created a task. All these five entities are variable scopes and the arrows mark a parent-child relationship. A variable that is defined on a parent scope is accessible in every child scope unless a child scope defines a variable of the same name. The other way around, child variables are not accessible from a parent scope. Variables that are directly attached to the scope in question are called *local* variables. Consider the following assignment of variables to scopes:

<center><img class="img-responsive" src="ref:asset:/guides/user-guide/assets/img/variables-5.png" /></center>

In this case, when working on *Task 1* the variables *worker* and *customer* are accessible. Note that due to the structure of scopes, the variable *worker* can be defined twice, so that *Task 1* accesses a different *worker* variable than *Task 2*. However, both share the variable *customer* which means that if that variable is updated by one of the tasks, this change is also visible to the other.

Both tasks can access two variables each while none of these is a local variable. All  three executions have one local variable each.

Now let's say, we set a local variable *customer* on *Task 1*:

<center><img class="img-responsive" src="ref:asset:/guides/user-guide/assets/img/variables-6.png" /></center>

While two variables named *customer* and *worker* can still be accessed from *Task 1*, the *customer* variable on *Execution 1* is hidden, so the accessible *customer* variable is the local variable of *Task 1*.

In general, variables are accessible in the following cases:

* Instantiating processes
* Delivering messages
* Task lifecycle transitions, such as completion or resolution
* Setting/getting variables from outside
* Setting/getting variables in a [Delegate](ref:#process-engine-delegation-code)
* Expressions in the process model
* Scripts in the process model
* (Historic) Variable queries


## Setting and Retrieving Variables - Overview

To set and retrieve variables, the process engine offers a Java API that allows to set variables from Java objects and retrieve them in the same form. Internally, the engine persists variables to the database and therefore applies serialization. For most applications, this is a detail of no concern. However, sometimes when working with custom Java classes, the serialized value of a variable is of interest. Imagine the case of a monitoring application that manages many process applications. It is decoupled from those applications' classes and therefore cannot access custom variables in their Java representation. For these cases, the process engine offers a way to retrieve and manipulate the serialized value. This boils down to two APIs:

* **Java Object Value API**: Variables are represented as Java objects. These objects can be directly set as values and retrieved in the same form. This is the more simple API and is the recommended way when implementing code as part of a process application.
* **Typed Value API**: Variable values are wrapped in so-called *typed values* that are used to set and retrieve variables. A typed value offers access to meta data such as the way the engine has serialized the variable and, depending on the type, the serialized representation of the variable.

As an example, the following code retrieves and sets two integer variables using both APIs:

```java
// Java Object API: Get Variable
Integer val1 = (Integer) execution.getVariable("val1");

// Typed Value API: Get Variable
IntegerValue typedVal2 = execution.getVariableTyped("val2");
Integer val2 = typedVal2.getValue();

Integer diff = val1 - val2;

// Java Object API: Set Variable
execution.setVariable("diff", diff);

// Typed Value API: Set Variable
IntegerValue typedDiff = Variables.integerValue(diff);
execution.setVariable("diff", typedDiff);
```

The specifics of this code are described in more detail in the sections on the [Java Object Value API](ref:#process-engine-process-variables-java-object-api) and the [Typed Value API](ref:#process-engine-process-variables-typed-value-api).

## Supported Variable Values

The process engine supports the following variable value types:

<center><img class="img-responsive" src="ref:asset:/guides/user-guide/assets/img/variables-1.png" /></center>

Depending on the actual value of a variable, a different type is assigned. Out of the available types, there are nine *primitive* value types, meaning that they correspond to standard JDK classes:

* `boolean`: Instances of `java.lang.Boolean`
* `bytes`: Instances of `byte[]`
* `short`: Instances of `java.lang.Short`
* `integer`: Instances of `java.lang.Integer`
* `long`: Instances of `java.lang.Long`
* `double`: Instances of `java.lang.Double`
* `date`: Instances of `java.util.Date`
* `string`: Instances of `java.lang.String`
* `null`: `null` references

Furthermore, the value type `object` represents custom Java objects. When such a variable is persisted, its value is serialized according to a serialization procedure. These procedures are configurable and exchangeable.

### Object Value Serialization

When an `object` value is passed to the process engine, a *serialization format* can be specified to tell the process engine to store the value in a specific format. Based on this format, the engine looks up a *serializer*. The serializer is able to serialize a Java object to the specified format and deserialize it from a representation in that format. That means, there may be different serializers for different formats and it is possible to implement custom serializers in order to store custom objects in a specific format.

The process engine ships one built-in object serializer for the format `application/x-java-serialized-object`. It is able to serialize Java objects that implement the interface `java.io.Serializable` and applies standard Java object serialization.

The desired serialization format can be specified when setting a variable using the Typed Value API. On top of that, the process engine configuration has an option `defaultSerializationFormat` that is used when no specific format is requested. This option defaults to `application/x-java-serialized-object`.

<div class="alert alert-info">
  <p><strong>Using Custom Objects in Task Forms:</strong></p>
  <p>Note that the built-in serializer converts objects to byte streams that can only be interpreted with the Java class at hand. When implementing task forms that are based on complex objects, a text-based serialization format should be used since the tasklist cannot interpret these byte streams. See the box <i>Serializing Objects to XML and JSON</i> for details on how to integrate serialization formats like XML and JSON.</p>
</div>

<div class="alert alert-info">
  <p><strong>Serializing Objects to XML and JSON:</strong></p>
  <p>The <a href="ref:#data-formats-xml-json-other">camunda Spin plugin</a> provides serializers that are capable of serializing object values to XML and JSON. They can be used when it is desired that the serialized objects values can be interpreted by humans or when the serialized value should be meaningful without having the corresponding Java class.</p>
  <p>When using a pre-built camunda distribution, camunda Spin is already preconfigured and you can try these formats without further configuration.</p>
</div>

## Java Object API

The most convenient way of working with process variables from Java is to use their Java object representation. Wherever the process engine offers variable access, process variables can be accessed in this representation given that for custom objects the engine is aware of the involved classes. For example, the following code sets and retrieves a variable for a given process instance:

```java
com.example.Order order = new com.example.Order();
runtimeService.setVariable(execution.getId(), "order", order);

com.example.Order retrievedOrder = (com.example.Order) runtimeService.getVariable(execution.getId(), "order");
```

Note that this code sets a variable at the highest possible point in the hierarchy of variable scopes. This means, if the variable is already present (whether in this execution or any of its parent scopes), it is updated. If the variable is not yet present, it is created in the highest scope, i.e. the process instance. If a variable is supposed to be set exactly on the provided execution, the *local* methods can be used. For example:

```java
com.example.Order order = new com.example.Order();
runtimeService.setVariableLocal(execution.getId(), "order", order);

com.example.Order retrievedOrder = (com.example.Order) runtimeService.getVariable(execution.getId(), "order");
com.example.Order retrievedOrder = (com.example.Order) runtimeService.getVariableLocal(execution.getId(), "order");
// both methods return the variable
```

Whenever a variable is set in its Java representation, the process engine automatically determines a suitable value serializer or raises an exception if the provided value cannot be serialized.

## Typed Value API

In cases in which it is important to access a variable's serialized representation or in which the engine has to be hinted to serialize a value in a certain format, the typed-value-based API can be used. In comparison to the Java-Object-based API, it wraps a variable value in a so-called *Typed Value*. Such a typed value allows for a richer representation of variable values.

In order to easily construct typed values, camunda BPM offers the class `org.camunda.bpm.engine.variable.Variables`. This class contains static methods that allow to create single typed values as well as create a map of typed values in a fluent way.

### Primitive Values

The following code sets a single `String` variable by specifying it as a typed value:

```java
StringValue typedStringValue = Variables.stringValue("a string value");
runtimeService.setVariable(execution.getId(), "stringVariable", typedStringValue);

StringValue retrievedTypedStringValue = runtimeService.getVariableTyped(execution.getId(), "order");
String stringValue = retrievedTypedStringValue.getValue(); // equals "a string value"
```

Note that with this API, there is one more level of abstraction around the variable value. Thus, in order to access the true value, it is necessary to *unwrap* the actual value.

### Object Values

Of course, for plain `String` values, the Java-Object-based API is more concise. Let us therefore consider an example with a custom object value:

```java
com.example.Order order = new com.example.Order();
ObjectValue typedObjectValue = Variables.objectValue(order).create();
runtimeService.setVariableLocal(execution.getId(), "order", typedObjectValue);

ObjectValue retrievedTypedObjectValue = runtimeService.getVariableTyped(execution.getId(), "order");
com.example.Order retrievedOrder = (com.example.Order) retrievedTypedObjectValue.getValue();
```

This again is equivalent to the Java-Object-based API. However, it is now possible to tell the engine which serialization format to use when persisting the value. For example, `ObjectValue typedObjectValue = Variables.objectValue(order).serializationDataFormat("application/x-java-serialized-object").create();` creates a value that gets serialized by the engine's built-in Java object serializer. Also, a retrieved `ObjectValue` instance provides additional variable details:

```java
// returns true
boolean isDeserialized = retrievedTypedObjectValue.isDeserialized();

// returns the format used by the engine to serialize the value into the database
String serializationFormat = retrievedTypedObjectValue.getSerializationFormat();

// returns the serialized representation of the variable; the actual value depends on the serialization format used
String serializedValue = retrievedTypedObjectValue.getValueSerialized();

// returns the class com.example.Order
Class<com.example.Order> valueClass = retrievedTypedObjectValue.getObjectType();

// returns the String "com.example.Order"
String valueClassName = retrievedTypedObjectValue.getObjectTypeName();
```

The serialization details are useful when the calling application does not possess the classes of the actual variable value (i.e. `com.example.Order` is not known). In these cases, `runtimeService.getVariableTyped(execution.getId(), "order")` will raise an exception since it immediately tries to deserialize the variable value. In such a case, the invocation `runtimeService.getVariableTyped(execution.getId(), "order", false)` can be used. The additional boolean parameter tells the process engine to not attempt deserialization. In this case, the invocation `isDeserialized()` will return `false` and invocations like `getValue()` and `getObjectType()` will raise exceptions. Calling `getValueSerialized()` and `getObjectTypeName()` is a way to access the variable nonetheless.

Similarly, it is possible to set a variable from its serialized representation:

```java
String serializedOrder = "...";
ObjectValue serializedValue =
  Variables
    .serializedObjectValue(serializedOrder)
    .serializationDataFormat("application/x-java-serialized-object")
    .objectTypeName("com.example.Order")
    .create();

runtimeService.setVariableLocal(execution.getId(), "order", serializedValue);

ObjectValue retrievedTypedObjectValue = runtimeService.getVariableTyped(execution.getId(), "order");
com.example.Order retrievedOrder = (com.example.Order) retrievedTypedObjectValue.getValue();
```

<div class="alert alert-warning">
  <p><strong>Be Aware of Inconsistent Variable State:</strong></p>
  <p>When setting a serialized variable value, no checking is done whether the structure of the serialized value is compatible with the class the variable value is supposed to be an instance of. When setting the variable from the above example, the supplied serialized value is not validated against the structure of <code>com.example.Order</code>. Thus, an invalid variable value will only be detected when <code>runtimeService#getVariableTyped</code> is called.</p>
</div>

### Setting Multiple Typed Values

Similar to the Java-Object-based API, it is also possible to set multiple typed values in one API call. The `Variables` class offers a fluent API to construct a map of typed values:

```java
com.example.Order order = new com.example.Order();

VariableMap variables =
  Variables.create()
    .putValueTyped("order", Variables.objectValue(order))
    .putValueTyped("string", Variables.stringValue("a string value"));
runtimeService.setVariablesLocal(execution.getId(), "order", variables);
```

## Interchangeability of APIs

Both APIs offer different views on the same entities and can therefore be combined as is desired. For example, a variable that is set using the Java-Object-based API can be retrieved as a typed value and vice versa. As the class `VariableMap` implements the `Map` interface, it is also possible to put plain Java objects as well as typed values into this map.

Which API should you use? The one that fits your purpose best. When you are certain that you always have access to the involved value classes, such as when implementing code in a process application like a `JavaDelegate`, then the Java-Object-based API is easier to use. When you need to access value-specific meta data such as serialization formats, then the Typed-Value-based API is the way to go.

## Input/Output Variable Mapping

To improve the reusability of source code and business logic, camunda BPM offers input/output
mapping of process variables. This can be used for tasks, events and subprocesses.

In order to use the variable mapping, the camunda extension element [inputOutput][] has to be added
to the element. It can contain multiple [inputParameter][] and [outputParameter][] elements that
specify which variables should be mapped. The `name` attribute of an [inputParameter][] denotes
the variable name inside the activity (a local variable to be created), whereas the `name` attribute of an [outputParameter][]
denotes the variable name outside of the activity.

The content of an input/outputParameter specifies the value that is mapped to the corresponding
variable. It can be a simple constant string or an expression. An empty body sets the variable
to the value `null`.

```xml
<camunda:inputOutput>
  <camunda:inputParameter name="x">foo</camunda:inputParameter>
  <camunda:inputParameter name="willBeNull"/>
  <camunda:outputParameter name="y">${x}</camunda:outputParameter>
  <camunda:outputParameter name="z">${willBeNull == null}</camunda:outputParameter>
</camunda:inputOutput>
```

Even complex structures like [lists][list] and [maps][map] can be used. Both can also
be nested.

```xml
<camunda:inputOutput>
  <camunda:inputParameter name="x">
    <camunda:list>
      <camunda:value>a</camunda:value>
      <camunda:value>${1 + 1}</camunda:value>
      <camunda:list>
        <camunda:value>1</camunda:value>
        <camunda:value>2</camunda:value>
        <camunda:value>3</camunda:value>
      </camunda:list>
    </camunda:list>
  </camunda:inputParameter>
  <camunda:outputParameter name="y">
    <camunda:map>
      <camunda:entry key="foo">bar</camunda:entry>
      <camunda:entry key="map">
        <camunda:map>
          <camunda:entry key="hello">world</camunda:entry>
          <camunda:entry key="camunda">bpm</camunda:entry>
        </camunda:map>
      </camunda:entry>
    </camunda:map>
  </camunda:outputParameter>
</camunda:inputOutput>
```

A script can also be used to provide the variable value. Please see the corresponding
[section][script-io] in the scripting chapter for how to specify a script.

A simple example of the benefit of input/output mapping is a complex calculation which
should be part of multiple processes definitions. This calculation can be developed as isolated
delegation code or a script and be reused in every process, even though the processes use a different variable set.
An input mapping is used to map the different process variables to
the required input parameters of the complex calculation activity. Accordingly, an output mapping allows to utilize the
calculation result in the further process execution.

In more detail, let us assume such a calculation is implemented by a Java Delegate class `org.camunda.bpm.example.ComplexCalculation`.
This delegate requires a `userId` and a `costSum` variable as input
parameters. It then calculates three values, `pessimisticForecast`, `realisticForecast` and `optimisticForecast`,
which are different forecasts of the future costs a customer faces. In a first process, both input variables are available as process variables but with different names (`id`, `sum`). From the three results, the process only uses `realisticForecast` which it depends on by the name `forecast` in follow-up activities. A corresponding input/output mapping looks as follows:

```xml
<serviceTask camunda:class="org.camunda.bpm.example.ComplexCalculation">
  <extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="userId">${id}</camunda:inputParameter>
      <camunda:inputParameter name="costSum">${sum}</camunda:inputParameter>
      <camunda:outputParameter name="forecast">${realisticForecast}</camunda:outputParameter>
    </camunda:inputOutput>
  </extensionElements>
</serviceTask>
```

In a second process, let us assume the `costSum` variable has to be calculated from properties of three different maps. Also, the process
depends on a variable `avgForecast` as the average value of the three forecasts. In this case, the mapping looks as follows:

```xml
<serviceTask camunda:class="org.camunda.bpm.example.ComplexCalculation">
  <extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="userId">${id}</camunda:inputParameter>
      <camunda:inputParameter name="costSum">
        ${mapA[costs] + mapB[costs] + mapC[costs]}
      </camunda:inputParameter>
      <camunda:outputParameter name="avgForecast">
        ${(pessimisticForecast + realisticForecast + optimisticForecast) / 3}
      </camunda:outputParameter>
    </camunda:inputOutput>
  </extensionElements>
</serviceTask>
```

### Multi-instance IO Mapping

Input mappings can also be used with multi-instance constructs, in which the mapping is applied for every instance that is created. For example, for a multi-instance subprocess with five instances, the mapping is executed five times and the involved variables are created in each of the five subprocess scopes such that they can be accessed independently.

<div class="alert alert-info">
  <p><strong>No output mapping for multi-instance constructs:</strong></p>
  <p>The engine does not support output mappings for multi-instance constructs. Every instance of the output mapping would overwrite the variables set by the previous instances and the final variable state would become hard to predict.</p>
</div>


[inputOutput]: ref:/api-references/bpmn20/#custom-extensions-camunda-extension-elements-camundainputoutput
[inputParameter]: ref:/api-references/bpmn20/#custom-extensions-camunda-extension-elements-camundainputparameter
[outputParameter]: ref:/api-references/bpmn20/#custom-extensions-camunda-extension-elements-camundaoutputparameter
[list]: ref:/api-references/bpmn20/#custom-extensions-camunda-extension-elements-camundalist
[map]: ref:/api-references/bpmn20/#custom-extensions-camunda-extension-elements-camundamap
[script-io]: ref:#process-engine-scripting-use-scripts-as-inputoutput-parameters