# Migration guide matter.js Device API to 0.8

With version 0.8 the matter.js project introduces a new high level API to build devices bases on code generation
for all Matter 1.1 device types, clusters and functionality. In the past we already generated the code for all
cluster definitions and this has now been enhanced for all device types. Additionally, the way custom cluster logic can be
implemented also needed to be adjusted and was enhanced with a very flexible way to choose the wanted cluster features.

This means that developers need to adjust their code to use the new classes and concepts introduced by this change. The
former API (pre 0.8, called "Legacy" for of now) is still 100% functional, but will be deprecated and removed in upcoming
releases and might not get cluster specific adjustments or new features if they are not part of the core library
functionality!
The Controller API is still the same as before, but will be also adjusted in the future to the new concepts where
applicable.

This document tries to give an overview how the commonly used components and classes from the legacy API need to be
adjusted for the new API. Please also check out the FAQ at the end of this document.

## Examples

Matter.js contains several examples to show how devices are built that also can be used in practice as CLI scripts.
These examples were also adjusted and exist for the legacy API (\*Legacy.ts) as well as the new API. This can be used
too to see the differences between the APIs.

## TypeScript relevant settings
Beside the TS module resolution settings already mentioned in the [matter.js README.md](../packages/matter.js/README.md), the new API also requires to use at least `"strictNullChecks": true` or better for code quality `"strict": true` to make sure that all types are correctly determined.

## Components

The following sections shows the legacy and matching new components and tries to show the differences and what they
have in common.

### New:Environment <--> Legacy:MatterServer

The new API introduces an Environment which represents a platform specific basic environment. A "default" Environment is, in our
current case, initialized as soon as matter-node.js is imported. It encapsulates the basic Process, Network, Storage,
Configuration and logging for the rest of the components in one central place. The old API also had parts of this registered automatically but especially configuration and storage was needed to be provided by the developer.
Like in MatterServer the Environment also maintains the MDNS broadcasting and scanning for all nodes that are added later.

The Environment to use can be provided in the configuration of the ServerNode instance you create (see below) as property `environment` - if not provided the default environment is used.

In fact this is all what they have in common, so the differences are:

-   The default storage is defined by the Environment which is initialized - in the case of the Node.js Environment it is the file based key value store "node-localstorage". To exchange the storage to something else you can implement/extend an own Environment class (see [NodeJsEnvironment](../packages/matter-node.js/src/environment/NodeJsEnvironment.ts)) or just overwrite the storage factory (`Environment.default.get(StorageService).factory = (namespace: string) => createMyStorage(namespace);`)
-   Basic configuration can be provided via a config file, CLI parameters or also environment variables. Some defined configuration keys are used by the base environment or the Node.js environment (e.g. MDNS network interface and such), but also custom configuration can be added and access from within every place in the code by accessing the environment. So this also acts as central place to share configuration for the device implementation. Some variables and their usage is documented in the [Examples Readme](../packages/matter-node.js-examples/README.md). Else check the [Environment.ts](../packages/matter.js/src/environment/Environment.ts) and [NodeJsEnvironment.ts](../packages/matter-node.js/src/environment/NodeJsEnvironment.ts).
-   The "ProcessManager" of the environment will, in case of the Node.js environment, also register Process signal handlers to handle Shutdown (SIGINT, SIGTERM, exit) or to trigger logging diagnostic data (SIGUSR2). For other environments this needs to be implemented accordingly.
-   The environment adds the concept of Workers that can execute tasks/jobs/logic and these workers are used to run Nodes and finish when they are ended. With this Matter Servers (old name CommissioningServer)/Server Nodes (new) are registered on the Environment as workers. The ServerNode (see below) has convenient methods to do that registration, so these Workers are ideally encapsulated and are not needed be used directly the developer, but could for own workloads. The Workers are all disposed/ended when the Environment is disposed.
-    Port numbers that were optionally managed by the MatterServer are no longer managed and so the application needs to take care itself!

This Environment component even more simplifies to build devices by making sure base components are handled centrally for all things needed.

The environment related classes are exported unter `matter(-node).js/environment`.

### New:ServerNode <--> Legacy:CommissioningServer

The `new CommissioningServer()` is replaced by `await ServerNode.create()` in the new API and both represent one Matter Server node that starts on a provided port, announces itself as Device in the network to be paired with Controllers. The instance also represents the Matter Root-Endpoint with all mandatory Root clusters. The configuration is provided in a comparable way to the ServerNode as before too and can contain node specific configurations (network, productDescription and commissioning details) and also Root endpoint cluster configurations.
The create() method takes one or two parameters:

-   The definition of the RootEndpoint as first parameter. It can be omitted when it is the default RootEndpoint, or it is the definition including all relevant adjusted clusters or needed features. Check out [DeviceNodeFull.ts](../packages/matter-node.js-examples/src/examples/DeviceNodeFull.ts) or the [Testing Apps](../chip-testing/src/AllClustersTestInstance.ts) on how to extend the Root Cluster. See also details and examples below when we show the "Endpoint" component.
-   The configuration of the node as second (or if definition above is omitted as first) parameter. Provide the default configuration for all relevant clusters and such here. The configuration should also contain a unique id property for the Node.

When the node is created you add "Endpoints" to it which is comparable (and re-uses the name, but hs a different implementation and interface!) to the Device instances added in the legacy API. Please make sure to use the correct Endpoint class depending on if aou use the Legacy API or the new API!

Afterward you start the node. Here you have two options:

-   **`node.run()`**: This registers the node as worker on the environment and runs the server and resolves when the node gets closed! SO there is no code executed after this await until the devices was closed. Use this if you just have such a single node in one Node.js process and nothing else is needed and all additional logic is done by event handlers that were attached earlier in code.
-   **`node.bringOnline()`**: This registers the node as worker on the environment and start the server. The promise resolves when the node is online and announced in the network, so additional code can be executed afterwards.

The following methods are also existing on the ServerNode:

-   **`start()`**: This starts the node. Use only if the node was stopped before.
-   **`cancel()`**: This brings the node offline and removes all network sockets but leave state and structure intact, so it can be started again.
-   **`factoryReset()`**: This factory resets the device. If started it is stopped and restarted afterward.
-   **`destroy()`**: This destroys the node, taking it offline and removing it from the environment workers-

### New:Endpoint <--> Legacy:Endpoint and Device-Classes/Clusters

A "Endpoint" describes an endpoint which is added in the Matter endpoint structure.

**Note**
The name "Endpoint" is now defined twice - one time by both APIs, but have different exports and interfaces!

-   Legacy-API: Endpoint is on the "@project-chip/matter.js/device" export
-   New-API: Endpoint is on the "@project-chip/matter.js/endpoint" export
    Do not mix them up!

As the ServerNode above each Endpoint consists of an endpoint type definition and configuration for the Endpoint and the contained clusters.

The Device class exports to use for the new API are all located in "@project-chip/matter.js/devices/<Devicename>" and can be imported as needed to prevent importing too many classes.
The main difference between the new and the legacy Device classes are that the new ones are working generically, so that special convenience shortcut methods like in the legacy classes do not exist. But the new API adds a lot more flexibility.

There are several options to define and interact with the Endpoints that are described now.

#### Add a simple DeviceType

The most simple way is to add a device endpoint to the server node and get the Endpoint instance out of it

```javascript
const endpoint = await serverNode.add(OnOffLightDevice, { id: "myonofflight" });
```

Alternatively the Endpoint can be created beforehand and added then

```javascript
const endpoint = new Endpoint(OnOffLightDevice, { id: "myonofflight" });
await serverNode.add(endpoint);
```

The third alternative is to add the endpoints directly when configuring the ServerNode

```javascript
const serverNode = await ServerNode.create({
    id: "myNode",

    //... serverNode configuration

    parts: [
        {
            type: OnOffLightDevice,
            id: "myonofflight",

            //... more config for this part
        },
    ],
});
```

**IMPORTANT**
Please note that attribute change events can **not** be added before the endpoint is added to the node!

#### Provide cluster properties/defaults with creation

Each Device type has mandatory clusters that are added by default automatically and use their default values as defined by the Matter specification. To override these defaults you can add them as configuration when adding the node:

```javascript
const endpoint = await serverNode.add(OnOffLightDevice, {
    id: "myonofflight",

    onOff: {
        // OnOff Cluster
        onOff: true, // OnOff attribute from OnOff cluster
    },
});
```

The TypeScript typing should exactly tell you which attributes are existing and provide full typing support.

#### Add/Overwrite Clusters on the device

matter.js provides the Device classes with their mandatory clusters included and the defined or basic feature variants. If you want to overwrite this you can do this by modifying the Type definition:

You can overwrite clusters which are defined by own implemetations

```javascript
class ReportingOnOffServer extends OnOffLightRequirements.OnOffServer {
    // see LightDevice.ts in examples package
    //...
}

// Use the above defined OnOffServer for the OnOff cluster in the Device
const ExampleLight = OnOffLightDevice.with(ReportingOnOffServer);

const endpoint = await serverNode.add(ExampleLight, { id: "myonofflight" });
```

You can also add features for single clusters:

```javascript
// Create Server type with the needed features
const LiftingWindowCoveringServer = WindowCoveringServer.with("Lift", "AbsolutePosition", "PositionAwareLift");

// If needed, provide a implementation that supports all features as defined by Matter specs
class RollerShade extends LiftingWindowCoveringServer {
    // see IlluminatedRollerShade.ts in examples package
    //...
}

// Use the above defined WindowCovering cluster for the WindowCovering cluster in the Device
// if you do not add an own implementation simply use `WindowCoveringDevice.with(LiftingWindowCoveringServer)`
const endpoint = await serverNode.add(WindowCoveringDevice.with(RollerShade), { id: "myrollershade" });
```

To redefine multiple Clusters just separate them with comma in the with method

```javascript
// Enable the query feature for Identify cluster and the Colortemperature feature for the ColorControl cluster
const endpoint = await serverNode.add(
    ExtendedColorLightDevice.with(IdentifyServer.with("Query"), ColorControlServer.with("ColorTemperature")),
);
```

### Initialize and destroy cluster logic

The legacy API had the two handler methods "initializeClusterServer" and "destroyClusterServer" to initialize and
cleanup own cluster states.

The new API uses `initialize()` to initialize and `async [Symbol.asyncDispose]()` to cleanup the cluster state.

As an example you can check the [OnOffServer.ts](../packages/matter.js/src/behavior/definitions/on-off/OnOffServer.ts) implementation.

### Dynamic Getter/Setter for cluster attributes

The Legacy API allowed to use nameAttributeGetterGetter and nameAttributeSetter as CLuster command handlers to implement cases where the attribute value needs to be defined dynamically.

**Note**
Because this type of attribute value determination is problematic when it comes to subscriptions and other cases please try to use this only if it is really needed and the values are not relevant to be subscribed. Ideally set the attribite value when it is changed to the new value.

In order to create dynamic getter or setters in the new API you can overwrite the respective attributes in the Cluster state definition in the Server implementation as shown in [OperationalCredentialsServer.ts at the end of the file](../packages/matter.js/src/behavior/definitions/operational-credentials/OperationalCredentialsServer.ts):

The following example shows how to define a dynamic getter and setter for the attribute "currentFabricIndex":

```javascript
    //...
    [Val.properties](session: ValueSupervisor.Session) {
        return {
            get currentFabricIndex() {
                return session.fabric ?? FabricIndex.NO_FABRIC;
            },

            set currentFabricIndex(_value: number) {
                throw ImplementationError("Set not allowed");
            }
        };
    }
```

### Add Own/Custom defined clusters

matter.js also allows to define and add additional clusters to the system. Todo this we need the following components to be created:

-   The Tlv Schema definition of the cluster which is used to encode and decode the data on the matter message TlV level. Also the controller uses this to build cluster client representations to access the data
-   The matter.js Model definition of the cluster which is used by the new API to do additional validations
-   Some glue code to provide typings and such for TypeScript and developer convenience :-)

**Note**
Currently the Tlv-Schema and the Model definition is kind of duplicated code and needs to match in their respective formats. In the future we plan to use a json representation like it is already in use for all official clusters - and then offer code generators also for custom clusters which would create all the relevant code automatically. But the adjusted generators are not yet ready.

The DevicesFullNode.ts contains a [MyFancyFunctionality custom cluster](../packages/matter-node.js-examples/src/examples/cluster/MyFancyOwnFunctionality.ts) that shows how this can be built right now already (with a bit overhead as described). The code contained here in one file is normally split into several files in the generated code.

### React to change events on cluster attributes

To react to change events in your code outside of cluster implementations (there special rule might apply because of the transactionality) you do:

```javascript
// Register for the change event of the onOff attribute for the OnOff cluster of the endpoint
endpoint.events.onOff.onOff$Change.on(value => {
    console.log(`OnOff is now ${value ? "ON" : "OFF"}`);
});
```

**IMPORTANT**
The change handlers are executed in the scope of the called command. This means that exceptions that are thrown in the
state change handlers let the command fail! This also means that the transaction fails and all other changes are rolled
back automatically.

### Read or write attribute values

The Endpoint provides a direct structure to read attributes:

```javascript
// Read onOff attribute from onOff cluster
const onOffValue = endpoint.state.onOff.onOff;
console.log(`current OnOff attribute: ${onOffValue}`);
```

To set one or multiple (!) attributes use the set() method of the endpoint:

```javascript
// Set onOff attribute from OnOff cluster
await endpoint.set({
    onOff: {
        onOff: false,
    },
});
```

You can provide multiple values also from multiple clusters within this endpoint to set together. This means that they are set as a transaction - if one fails, all fail!

### How to get QR Code and pairing details if device is not commissioned?

```javascript
if (!serverNode.lifecycle.isCommissioned) {
    const { qrPairingCode, manualPairingCode } = server.state.commissioning.pairingCodes;

    console.log(QrCode.get(qrPairingCode));
    logger.info(`QR Code URL: https://project-chip.github.io/connectedhomeip/qrcode.html?data=${qrPairingCode}`);
    logger.info(`Manual pairing code: ${manualPairingCode}`);
} else {
    logger.info("Device is already commissioned. Waiting for controllers to connect ...");
}
```

### Which events are available to get notified on commissioning changes?
The Lagacy API used callbacks included in the CommissioningServer configuration. The new API uses the `lifecycle` property of the ServerNode to get notified on commissioning changes.

```javascript
server.lifecycle.commissioned.on(() => console.log("Server was initially commissioned successfully!"));

/** This event is triggered when all fabrics are removed from the device, usually it also does a factory reset then. */
server.lifecycle.decommissioned.on(() => console.log("Server was fully decommissioned successfully!"));
```

These events will not trigger if the node gets added to another controller. If you need these information the relevant event is available on the Commissioning Behavior of the ServerNode.

```javascript
/**
 * This event is triggered when a fabric is added, removed or updated on the device. Use this if more granular
 * information is needed.
 */
server.events.commissioning.fabricsChanged.on((fabricIndex, fabricAction) => {
    let action = "";
    switch (fabricAction) {
        case FabricAction.Added:
            action = "added";
            break;
        case FabricAction.Removed:
            action = "removed";
            break;
        case FabricAction.Updated:
            action = "updated";
            break;
    }
    console.log(`Commissioned Fabrics changed event (${action}) for ${fabricIndex} triggered`);
    console.log(server.state.commissioning.fabrics[fabricIndex]);
});
```

### Which events are available to get notified that a node is online or offline?
The new API provides this information also via events on the ServerNode instance.

```javascript
/** This event is triggered when the device went online. This means that it is discoverable in the network. */
server.lifecycle.online.on(() => console.log("Server is online"));

/** This event is triggered when the device went offline. it is not longer discoverable or connectable in the network. */
server.lifecycle.offline.on(() => console.log("Server is offline"));
```

### Which events are available to get an overview on controller connections/sessions that are established?
Events on session changes are available on the sessions behavior of the ServerNode instance.

```javascript
/**
 * This event is triggered when an operative new session was opened by a Controller.
 * It is not triggered for the initial commissioning process, just afterwards for real connections.
 */
server.events.sessions.opened.on(session => console.log(`Session opened`, session));

/**
 * This event is triggered when an operative session is closed by a Controller or because the Device goes offline.
 */
server.events.sessions.closed.on(session => console.log(`Session closed`, session));

/** This event is triggered when a subscription gets added or removed on an operative session. */
server.events.sessions.subscriptionsChanged.on(session => console.log(`Session subscriptions changed`, session));
```

With `server.state.sessions.sessions` you can get a list of all currently active sessions including the relevant information.

### Can I add Clusters dynamically to an endpoint also after creation?
Yes also this is possible. You can add clusters to an endpoint also after creation. This is done by the `behaviors.require` method of the endpoint.

This example dynamically adds a BridgedDeviceBasicInformation cluster to an endpoint, to dynamically allow the endpoint to be added to a bridge. The second parameter contains the default values for the cluster state of the added cluster.
This do not have any effects on the typings of the relevant endpoint, so especially when using attributes of this added cluster you ned to use special methods to do so:
-   `endpoint.stateOf(BridgedDeviceBasicInformationServer)` to get and
-   `endpoint.setStateOf(BridgedDeviceBasicInformationServer, { ... })` to set the states of this cluster.

```javascript
endpoint.behaviors.require(BridgedDeviceBasicInformationServer, {
    nodeLabel: name,
    productName: name,
    productLabel: name,
    uniqueId: this.devicesOptions[i].uuid[i].replace(/-/g, ''),
    reachable: true,
});
```

### More options?

Take a look at the [DeviceNodeFull.ts](../packages/matter-node.js-examples/src/examples/DeviceNodeFull.ts) example for more interaction points.

## FAQ

### Are the Devices created by the new API compatible with the legacy API?

The devices itself and functionality are at least equal - if not better with the new API because we also did some fixes that were not in the 7.7.x versions.
But most important is that the storage structure has changed between legacy and New API - this means that data are stored in a different way and so a device commissioned with the Legacy API will not work with the new API. You need to delete and recommission the device when migrating!

