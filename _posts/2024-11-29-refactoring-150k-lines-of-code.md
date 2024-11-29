---

layout: post
author: Al Calzone
title: "Refactoring 150k lines of code, part 1"
summary: |-
  Or: <i>Trying out Rust's traits in TypeScript</i>
# image: chameleon-coding.jpeg
# feature: chameleon-coding.jpeg
comments: true

---

Since I started working on Z-Wave JS, quite a bit of time has passed. The project will turn 7 next March, and for the last 2.5 years of that, I've been able to work on it full-time ([sort of](https://github.com/zwave-js/node-zwave-js/issues/6521)).

It has grown a lot since then, and if you can trust TypeScript's diagnostics, Z-Wave JS consists of roughly 150k lines of code spread across roughly 700 files, including test files. This is not counting supporting projects like the firmware update service, Z-Wave JS Server, the repository bot, etc...

Although I try to stay up to date in terms of tooling and best practices, a codebase this size inevitably accumulates some technical debt.

## Everything points at the driver

One prime example is how everything in Z-Wave JS was built around the `Driver` instance. The driver is the core of the library. It controls the serial interface, handles receiving and sending of commands, and is used to access the network cache and device-specific configuration. Any action a library user wants to perform goes through a driver instance at some point.

So from the very beginning, almost every class in the library has required a reference to the driver instance. This has led to a lot of classes having a constructor that looks like this:

```ts
export class CommandClass {
  public constructor(
    driver: Driver,
    options: CommandClassOptions
  ) {
    this.driver = driver;

    // other constructor logic
  }

  // other class logic
}
```

The problem with that? The `Driver` class is huge, with far over 100 methods and properties. Testing/mocking this dependency is impossible. Breaking API changes need to be carried out in hundreds of files, in possibly thousands of locations.

## Abstractions to the rescue?

When I released version 10 of Z-Wave JS two years ago, I tackled this a bit by introducing two abstractions `ZWaveHost` and `ZWaveApplicationHost`, which the `Driver` class implemented both.

`ZWaveHost` provided a smaller subset of functionality that was easier to test and mock and was used where a reference to the driver was previously stored.
`ZWaveApplicationHost` built on top of that and added functionality that was more tightly coupled to the driver instance. It's main purpose was removing dependencies between packages of the monorepo, so code that previously required a driver instance could be moved to a package the main package depended on.

The constructors now looked roughly like this:

```ts
export class CommandClass {
  public constructor(
    host: ZWaveHost,
    options: CommandClassOptions
  ) {
    this.host = host;

    // other constructor logic
  }

  // other class logic
}
```


This made things a bit better, but these abstractions were still beefy. I mean, just look at them:

```ts
/** Host application abstractions to be used in Serial API and CC implementations */
export interface ZWaveHost {
  /** The ID of this node in the current network */
  ownNodeId: number;
  /** The Home ID of the current network */
  homeId: number;

  /** How many bytes a node ID occupies in serial API commands */
  readonly nodeIdType?: NodeIDType;

  /** Management of Security S0 keys and nonces */
  securityManager: SecurityManager | undefined;
  /** Management of Security S2 keys and nonces (Z-Wave Classic) */
  securityManager2: SecurityManager2 | undefined;
  /** Management of Security S2 keys and nonces (Z-Wave Long Range) */
  securityManagerLR: SecurityManager2 | undefined;

  /**
   * Retrieves the maximum version of a command class that can be used to communicate with a node.
   * Returns 1 if the node claims that it does not support a CC.
   * Throws if the CC is not implemented in this library yet.
   */
  getSafeCCVersion(
    cc: CommandClasses,
    nodeId: number,
    endpointIndex?: number,
  ): number;

  /**
   * Retrieves the maximum version of a command class the given node/endpoint has reported support for.
   * Returns 0 when the CC is not supported or that information is not known yet.
   */
  getSupportedCCVersion(
    cc: CommandClasses,
    nodeId: number,
    endpointIndex?: number,
  ): number;

  /**
   * Determines whether a CC must be secure for a given node and endpoint.
   */
  isCCSecure(
    cc: CommandClasses,
    nodeId: number,
    endpointIndex?: number,
  ): boolean;

  getHighestSecurityClass(nodeId: number): MaybeNotKnown<SecurityClass>;

  hasSecurityClass(
    nodeId: number,
    securityClass: SecurityClass,
  ): MaybeNotKnown<boolean>;

  setSecurityClass(
    nodeId: number,
    securityClass: SecurityClass,
    granted: boolean,
  ): void;

  /**
   * Returns the next callback ID. Callback IDs are used to correlate requests
   * to the controller/nodes with its response
   */
  getNextCallbackId(): number;

  /**
   * Returns the next session ID for supervised communication
   */
  getNextSupervisionSessionId(nodeId: number): number;

  getDeviceConfig?: (nodeId: number) => DeviceConfig | undefined;

  __internalIsMockNode?: boolean;
}

/** Host application abstractions that provide support for reading and writing values to a database */
export interface ZWaveValueHost {
  /** Returns the value DB which belongs to the node with the given ID, or throws if the Value DB cannot be accessed */
  getValueDB(nodeId: number): ValueDB;

  /** Returns the value DB which belongs to the node with the given ID, or `undefined` if the Value DB cannot be accessed */
  tryGetValueDB(nodeId: number): ValueDB | undefined;
}

/** A more featureful version of the ZWaveHost interface, which is meant to be used on the controller application side. */
export interface ZWaveApplicationHost extends ZWaveValueHost, ZWaveHost {
  /** Gives access to the configuration files */
  configManager: ConfigManager;

  options: ZWaveHostOptions;

  // TODO: There's probably a better fitting name for this now
  controllerLog: ControllerLogger;

  /** Readonly access to all node instances known to the host */
  nodes: ReadonlyThrowingMap<number, IZWaveNode>;

  /** Whether the node with the given ID is the controller */
  isControllerNode(nodeId: number): boolean;

  sendCommand<TResponse extends ICommandClass | undefined = undefined>(
    command: ICommandClass,
    options?: SendCommandOptions,
  ): Promise<SendCommandReturnType<TResponse>>;

  waitForCommand<T extends ICommandClass>(
    predicate: (cc: ICommandClass) => boolean,
    timeout: number,
  ): Promise<T>;

  schedulePoll(
    nodeId: number,
    valueId: ValueID,
    options: NodeSchedulePollOptions,
  ): boolean;
}
```

It did make testing a lot easier, but there was still a glaring issue: The `CommandClass` class I showed above has hundreds of subclasses. Throughout the codebase they are used interchangeably - the calling code often does not know or care which specific subclass it is, so they all share a common API. As a result, these host interfaces define lots and lots of logic that is irrelevant in many cases, but necessary in some.

If you want to mock this, you have to mock it all, or use `any` liberally and have tests fail left and right when things change.

So after dwelling on this for a while, I decided to try something new.

## Taking a page from Rust's book

Unlike many programming languages, the [Rust programming language](https://www.rust-lang.org/) does not have inheritance. Instead, it uses an unconventional approach for defining shared behavior, called [traits](https://doc.rust-lang.org/book/ch10-02-traits.html). Traits are similar to TypeScript's interfaces, often very limited in scope, and composition of traits is encouraged and often necessary.

Compared to TypeScript, it's also a lot more common to see some verbose type definitions in Rust (e.g. `Arc<Mutex<HashMap<String, Box<dyn Any>>>>`), so the end result may feel less strange to Rust veterans.

Anyways, in version 14 of Z-Wave JS, I applied the concept of traits to break the aforementioned interfaces into tiny pieces:
```ts
/** Allows querying the home ID and node ID of the host */
export interface HostIDs {
  /** The ID of this node in the current network */
  ownNodeId: number;
  /** The Home ID of the current network */
  homeId: number;
}

/** Allows looking up Z-Wave manufacturers by manufacturer ID */
export interface LookupManufacturer {
  /** Looks up the name of the manufacturer with the given ID in the configuration DB */
  lookupManufacturer(manufacturerId: number): string | undefined;
}

/** Allows accessing a specific node */
export interface GetNode<T extends NodeId> {
  getNode(nodeId: number): T | undefined;
  getNodeOrThrow(nodeId: number): T;
}
```
...and many, many more.

The same principle applies to the generics used in these trait-like interfaces. `NodeId` is the bare minimum required to identify a node,
```ts
/** Identifies a node */
export interface NodeId extends EndpointId {
  readonly id: number;
}

/** Identifies an endpoint */
export interface EndpointId {
  readonly virtual: false;
  readonly nodeId: number;
  readonly index: number;
}
```
so methods that need to retrieve a node by ID (`GetNode<...>`) can further define which information and functionality from a node they are interested in. For example, a method that wants to access a node, its endpoints, and is interested in which command classes are supported by those endpoints, could look like this:
```ts
function getEndpointInformation(
  context: GetNode<
    & NodeId
    & GetEndpoint<
      & EndpointId
      & SupportsCC
      & ControlsCC
    >
  >
) { ... }
```
To break it down, the parameter passed to this method
- allows accessing a node "object" by ID, which
  - has the required properties to identify that node
  - allows accessing an endpoint "object" by ID, which
    - has the required properties to identify that endpoint
    - allows checking which command classes are **supported** by that endpoint
    - allows checking which command classes are **controlled** by that endpoint

This approach allows for very fine-grained control over the functionality a method expects to have access to. It does not come without downsides though - some signatures become quite verbose. Here are two extreme examples I turned into separate type definitions to keep the method signatures that use them readable:
```ts
// Defines the necessary traits the host passed to a CC API must have
export type CCAPIHost<TNode extends CCAPINode = CCAPINode> =
  & HostIDs
  & GetNode<TNode>
  & GetValueDB
  & GetSupportedCCVersion
  & GetSafeCCVersion
  & SecurityManagers
  & GetDeviceConfig
  & SendCommand
  & GetCommunicationTimeouts
  & GetUserPreferences
  & SchedulePoll
  & LogNode;
```
and my personal (slightly cursed) favorite:
```ts
// Defines the necessary traits an endpoint passed to a CC API must have
export type CCAPIEndpoint =
  & (
    | (
      // Physical endpoints must let us query their controlled CCs
      EndpointId & ControlsCC
    )
    | (
      // Virtual endpoints must let us query their physical nodes,
      // the CCs those nodes implement, and access the endpoints of those
      // physical nodes
      VirtualEndpointId & {
        node: PhysicalNodes<
          & NodeId
          & SupportsCC
          & ControlsCC
          & GetEndpoint<EndpointId & SupportsCC & ControlsCC>
        >;
      }
    )
  )
  & SupportsCC;
```

## Conclusion

Those are some extreme outliers though. All in all, I'm quite happy with the result. There are now ~20 remaining instances where the `Driver` instance is passed around, and I'm confident I can get rid of most of them as well.

How I got from hundreds of references to just those 20 is a tale for another time, though. Stay tuned for part 2!\
_Spoiler: I did not edit 700 files by hand!_