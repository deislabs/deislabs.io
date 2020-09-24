---
title: "A Fistful of States: More State Machine Patterns in Rust"
description: "State machines in Rust"
date: 2020-08-24 00:00:00 +0000 UTC
authorname: "Kevin Flansburg"
author: "@kflansburg"
authorlink: "https://github.com/kflansburg"
image: "images/twitter-card.png"
tags: ["krustlet", "rust"]
---

Earlier this year, DeisLabs released Krustlet, a project to implement Kubelet 
in Rust. [^Fisher] [^Squillace] Kubelet is the component of Kubernetes that 
runs on each node which is assigned Pods by the control plane and runs them on 
its node. Krustlet defines a flexible API in the `kubelet` crate, which allows 
developers to build Kubelets to run new types of workloads. The project 
includes two such examples, which can run Web Assembly workloads on WASI or 
waSCC runtimes. [^wasmtime] [^waSCC] Beyond this, I have been working to 
develop a Rust Kubelet for traditional Linux containers using the Container 
Runtime Interface (CRI). [^krustlet-cri] [^CRI]

Over the last few releases, Krustlet has focused on expanding the functionality 
of these Web Assembly Kubelets, such as adding support for init containers, 
fixing small bugs, and log streaming. This, in turn, has built quite a bit of 
interest in alternative workloads and node architectures on Kubernetes, as well 
as demonstrated the many strengths of Rust for development of these types of 
applications.

For the `v0.5.0` release, we turned our attention to the internal architecture 
of Krustlet, in particular how Pods move through their lifecycle, how 
developers write this logic, and how updates to the Kubernetes control plane 
are handled. [^Pod Lifecycle] We settled upon a state machine implementation 
which should result in fewer bugs and greater fault tolerance. This refactoring 
resulted in substantial changes for consumers of our API; however we believe 
this will result in code that is much easier to reason about and maintain. For 
an excellent summary of these changes, and description of how you can migrate 
code that depends on the `kubelet` crate, please see Taylor Thomas' excellent 
[Release Notes](https://github.com/deislabs/krustlet/releases/tag/v0.5.0).
In this post I will share a deep dive into our new architecture and the 
development journey which led to it. 

## The Trailhead 

Before `v0.5.0`, developers wishing to implement a Kubelet using Krustlet 
primarily needed to implement the `Provider` trait, which allowed them to write 
methods for handling events like `add`, `delete`, and `logs` for Pods scheduled 
to that node. This offered a lot of flexibility, but was a very low-level API. 
We identified a number of issues with this architecture:

* The entire lifecycle of a Pod was defined in 1 or 2 monolithic methods of the 
  `Provider` trait. This resulted in messy code and a very poor understanding 
  of error handling in the many phases of a Pod's lifecycle. 
* Pod and Container status patches to the Kubernetes control plane were 
  scattered throughout the codebase, both in the `kubelet` crate and the 
  `Provider` implementations. This made it very difficult to reason about what 
  was actually reported back to the user and when, and involved lot of repeated 
  code. 
* Unlike the Go Kubelet, if Krustlet encountered an error it would report the 
  error back to Kubernetes and then (most of the time) end execution of the 
  Pod. There was no built-in notion of the reconciliation loop that one expects 
  from Kubernetes. 
* We recognized that a lot of these issues were left to each developer to 
  solve, but were things that any Kubelet would need to handle. We wanted to 
  move this kind of logic into the `kubelet` crate, so that each provider did 
  not have to reinvent things. 

## Our Mission

At its core, Kubernetes relies on declarative (mostly immutable) manifests, and 
controllers which run reconciliation loops to drive cluster state to match this 
configuration. Kubelet is no exception to this, with its focus being 
indivisible units of work, or Pods. Kubelet simply monitors for changes to Pods 
that have been assigned to it by `kube-scheduler`, and runs a loop to attempt 
to run this work on its node. In fact, I would describe Kubelet as no different 
from any other Kubernetes controller, except that it has the additional 
first-class capability for streaming logs and exec sessions. However these 
capabilities, as they are implemented in Krustlet, are orthogonal to this 
discussion.

Our goal with this rewrite was to ensure that Krustlet would mirror the official
Kubelet's behavior as closely as possible. We found that many details about
this behavior are undocumented, and spent considerable time running the 
application to infer its behavior and inspecting the Go source code. Our 
understanding is as follows:

 * The Kubelet watches for Events on Pods that have been scheduled to it by 
   `kube-scheduler`.
 * When a Pod is added, the Kubelet enters a control loop to attempt to run the 
   Pod which only exits when the Pod is `Completed` (all containers 
   exit successfully) or `Terminated` (Pod is marked for deletion via the 
   control plane and execution is interrupted). 
 * Within the control loop, there are various steps such as `Image Pull`, 
   `Starting`, etc., as well as back-off steps which wait some time before 
   retrying the Pod. At each of these steps, the Kubelet updates the control 
   plane. 

We recognized this pretty quickly as a finite-state machine design pattern,
which consists of infallible state handlers and valid state transitions. This 
allows us to address the issues mentioned above:

 * Break up the `Provider` trait methods for running the Pod into short, 
   single-focus state handler methods. 
 * Consolidate status patch code to where a Pod enters a given state. 
 * Include error and back-off states in the state graph, and only stop 
   attempting to execute a Pod on `Terminated` or `Complete`.
 * Move as much of this logic into `kubelet` as possible so that providers need 
   only focus on implementing the state handlers. 
   
With this architecture it becomes very easy to understand the behavior of the 
application, and strengthens our confidence that the application will not enter 
undefined behavior. In addition, we felt that Rust would allow us to achieve 
our goals while presenting an elegant API to developers, and with full 
compile-time enforcement of our state machine rules.

## Our Animal Guide 

When first discussing the requirements of our state machine, and the daunting 
task of integrating it with the existing Krustlet codebase, we recalled an 
excellent blog post, 
[Pretty State Machine Patterns in Rust](https://hoverbear.org/blog/rust-state-machine-pattern/),
by Ana Hobden (hoverbear), which I think has inspired a lot of Rust developers.
The post explores patterns in Rust for implementing state machines which 
satisfy a number of constraints and leverage Rust's type system. I encourage 
you to read the original post, but for the sake of this discussion I will 
paraphrase the final design pattern here: 

```rust
struct StateMachine<S> {
    state: S,
}

struct StateA;

impl StateMachine<StateA> {
    fn new() -> Self {
        StateMachine {
            state: StateA
        }
    }
}

struct StateB;

impl From<StateMachine<StateA>> for StateMachine<StateB> {
    fn from(val: StateMachine<StateA>) -> StateMachine<StateB> {
        StateMachine {
            state: StateB 
        }
    }
}

struct StateC;

impl From<StateMachine<StateB>> for StateMachine<StateC> {
    fn from(val: StateMachine<StateB>) -> StateMachine<StateC> {
        StateMachine {
            state: StateC 
        }
    }
}

fn main() {
    let in_state_a = StateMachine::new();

    // Does not compile because `StateC` is not `From<StateMachine<StateB>>`.
    // let in_state_c = StateMachine::<StateC>::from(in_state_a);

    let in_state_b = StateMachine::<StateB>::from(in_state_a);

    // Does not compile because `in_state_a` was moved in the line above.
    // let in_state_b_again = StateMachine::<StateB>::from(in_state_a);

    let in_state_c = StateMachine::<StateC>::from(in_state_b);
}
```

Ana introduces a number of requirements for a good state machine 
implementation, and achieves them with concise and easily interpretable code. 
In particular, these requirements (some based on the definition of a state 
machine, and some on ergonomics) were a high priority for us:

* One state at a time.
* Capability for shared state.
* Only explicitly defined transitions should be permitted.
* Any error messages should be easy to understand. 
* As many errors as possible should be identified at **compile-time**.

In the next section I will discuss some additional requirements that we 
introduced and how these impacted the solution. In particular, we relaxed some
of Ana's goals in exchange for greater flexibility, while satisfying those 
listed above.

## Tribulation

We were off to a great start, but it was time to consider how we want 
downstream developers to interact with our new state machine API. In 
particular, while the Kubelets we are familiar with all follow roughly the same 
Pod lifecycle, we wanted developers to be able to implement arbitrary state 
machines for their Kubelet. For example, some workloads or architectures may 
need to have additional provisioning states for infrastructure or data, or 
to introduce post-run states for proper garbage collection of resources. 
Additionally, it felt like an anti-pattern to have a parent method (`main` in 
the example above) which defines the logic for progressing through the states, 
as this felt like having two sources of truth and was not something we could 
implement on behalf of our downstream developers for arbitrary state machines. 
Ana had discussed how to hold the state machine in a parent structure using an 
`enum`, but it felt clunky to introduce large match statements which could 
introduce runtime errors.

We knew that to allow arbitrary state machines we would need a `State` trait to 
mark types as valid states. We felt that it would be possible for this trait to 
have a `next()` method which runs the state and then returns the next `State` 
to transition to, and we wanted our code to be able to simply call `next()` 
repeatedly to drive the machine to completion. This pattern, we soon found,
introduced a number of challenges.

```rust
/// Rough pseudocode of our plan.

trait State {
    /// Do work for this state and return next state.
    async fn next(self) -> impl State;
}

fn drive_state_machine(mut state: impl State) {
    loop {
        state = state.next().await;
    }
}
```

### What does `next()` return?

Within our loop, we are repeatedly overwriting a local variable with 
*different* types that all implement `State`. Without our parent method, or 
wrapper `enum`, there was not a straightforward way for Rust to know how to 
store these objects on the stack. Needless to say, the Rust compiler was 
displeased. We spent some time pair programming with the Rust Playground on 
solutions to this, and settled on using trait objects (`Box<dyn State>`), which 
moves the object itself to the heap. 

Utilizing the heap violates one of Ana's original goals, and we learned that 
the use of trait objects introduces a lot of (necessary) limitations on the 
trait itself, including that it disallows generic methods and referencing `Self` in return types. 
[^Trait Objects] This does not prevent us from achieving our goals, but is 
limiting and reduces performance due to the use of dynamic dispatch. We will 
continue to explore stack-based solutions.

With the use of trait objects, there are only two options for the return type 
of `next()`, which we captured with an `enum`:

```rust
pub enum Transition {
    /// Transition to a new state.
    Next(Box<dyn State>),
    /// End execution of state machine with result.
    Complete(Result<()>)
}
```

### A Bit of Folly 

We briefly explored one option for avoiding boxed traits: rather than iterating 
on a trait object, we could define a recursive function, which accepts a state,
runs its handler, and then calls itself with the next state.  This would place 
all of our state objects on the stack by growing it with each recursive call,
and felt like a very interesting solution which leveraged Rust's new 
`impl Trait` feature, as well as the `async_recursion` crate. Of course we 
realized that this was not acceptable because Pods regularly get caught in 
loops (such as image pull / image pull backoff), which would grow the stack 
without limit. Another drawback was that with concrete generic types, 
`Transition` had to have an `enum` variant for each state that could be 
transitioned to in a given state handler, making it impractical to support more 
than a few outgoing edges in the state graph.

```rust
/// Represents result of state execution and which state to transition to.
pub enum Transition<S, E> {
    /// Advance to next state.
    Advance(S),
    /// Transition to error state.
    Error(E),
    /// This is a terminal node of the state graph.
    Complete(Result<()>),
}

#[async_recursion::async_recursion]
/// Recursively evaluate state machine until a state returns Complete.
pub async fn run_to_completion(state: impl State) -> Result<()> {
    let transition = { state.next().await? };

    match transition {
        Transition::Advance(s) => run_to_completion(s).await,
        Transition::Error(s) => run_to_completion(s).await,
        Transition::Complete(result) => result,
    }
}
``` 

### Kubernetes-specific Behavior

A simpler task was adding behavior to the `State` trait to support our needs.
This included adding context to the `next()` method, such as the Pod manifest
and a generic type, `PodState`, to serve as shared data between state handlers. 
This data is *not* shared between Pods, so state handlers from different Pods 
can execute simultaneously. For any state shared between Pods, we chose to 
leave it to developers to implement concurrency controls, which should make it 
much more obvious to them when one Pod's execution is blocking another's. 

Next, we added a second method that would be called upon when entering a state, 
which should produce a JSON patch for the Pod status associated with that 
state. This update is then sent to the Kubernetes control plane by our API, and 
this is ideally the *one* place in the code where Pod status patches are 
applied. 

With all this in mind, we end up with a trait and a function to iteratively 
drive it to completion:

```rust
#[async_trait::async_trait]
/// A trait representing a node in the state graph.
pub trait State<PodState>: Sync + Send + 'static + std::fmt::Debug {
    /// Run state handler and return next state or complete.
    async fn next(
        // We *are* able to use `Self` here, allowing us to move the object, 
        // and preventing reuse.
        self: Box<Self>,
        // Pod state shared between state handlers.
        pod_state: &mut PodState,
        // Pod manifest.
        pod: &Pod, 
    ) -> anyhow::Result<Transition<PodState>>;

    /// Returns JSON status patch to apply when entering this state.
    async fn json_status(
        &self,
        pod_state: &mut PodState,
        pod: &Pod,
    ) -> anyhow::Result<serde_json::Value>;
}

/// Iteratively evaluate state machine until it returns Complete.
pub async fn run_to_completion<PodState: Send + Sync + 'static>(
    client: &kube::Client,
    state: impl State<PodState>,
    pod_state: &mut PodState,
    pod: &Pod,
) -> anyhow::Result<()> {
    let api: Api<KubePod> = Api::namespaced(client.clone(), pod.namespace());

    let mut state: Box<dyn State<PodState>> = Box::new(state);

    loop {
        let patch = state.json_status(pod_state, &pod).await?;
        let data = serde_json::to_vec(&patch)?;
        api.patch_status(&pod.name(), &PatchParams::default(), data)
            .await?;

        let transition = { state.next(pod_state, &pod).await? };

        state = match transition {
            Transition::Next(s) => {
                s.state
            }
            Transition::Complete(result) => {
                break result;
            }
        };
    }
}
```

### How to enforce edges 

The last issue that we tackled was how to enforce constraints on which state 
transitions are valid. When we moved to using trait objects, we effectively 
made our state machine a fully connected graph. When objects are boxed and 
returned as trait objects, their type is effectively lost, except for the trait 
they implement. This meant that state handlers could return anything that is 
`State`, and it would compile. We wanted to find a way to explicitly define
valid state transitions, as shown in Ana's solution, and transparently
enforce this with the API we had developed.

This resulted in a devious plan. Under the guise of providing a nice 
static method to handle Boxing for the user, we would add a `where` clause to 
ensure that a directed edge trait is implemented between the two `State`s. 
Finally, the state itself would be wrapped in a struct with a private field, 
which would prevent manual construction of `Transition::Next` without using 
this static method (outside of the `kublet` crate, at least).

```rust
/// Implementor can transition to state `S`.
pub trait TransitionTo<S> {}

pub struct StateHolder { 
    // This field is private.
    state: Box<dyn State> 
}

impl Transition {
    pub fn next<ThisState: State, NextState: State>(
        _t: ThisState, 
        n: NextState
    ) -> Transition where ThisState: TransitionTo<NextState>,
    {
        Transition::Next(StateHolder { state: Box::new(s) })
    }
}
```

A state handler could then transition to a valid state by returning:

```rust
Transition::next(self, NextStateObject)
```

The drawback here is that it is a little clunky to pass `self` to this 
static method. We explored the use of generics or `PhantomData` to clean up 
this API, however the nature of using trait objects means that we cannot 
reference `Self` in the return type of `next()`, so that type information 
cannot be part of `Transition`. This would seem to prevent type inference here, 
and require the user to explicitly specify `self` in one form or another.  

## Our Trail Camp for this Release

Having developed our state machine API, developers can now implement a Kubelet 
by defining their states and edges, and then supplying three new associated
types and one new method when implementing the `Provider` trait:

* `InitialState: State` is the entrypoint of the state machine.
* `TerminatedState: State` is jumped to when a Pod is marked for deletion.
* `PodState` is the type used for storing state that is shared between the 
  state handlers of a Pod.
* `fn initialize_pod_state` is called to create a `PodState` for a new Pod. Any 
  state shared between Pods should be injected here. 

We suggest placing each state type and it's `State` implementation in its own 
file or module for easier navigation of the code, a pattern that was used for 
both our WASI and waSCC Kubelets. Here is an example definition of a state 
machine for a very simple provider that starts and then runs until it is 
terminated:

```rust
struct PodState;

struct Starting;

#[async_trait::async_trait]
impl State<PodState> for Starting {
    async fn next(
        self: Box<Self>,
        pod_state: &mut PodState,
        pod: &Pod,
    ) -> anyhow::Result<Transition<PodState>> {
        // TODO: start workload 
        Ok(Transition::next(Running))
    }

    async fn json_status(
        &self,
        pod_state: &mut PodState,
        pod: &Pod,
    ) -> anyhow::Result<serde_json::Value> {
        Ok(serde_json::json!(null))
    }
}

struct Running;

#[async_trait::async_trait]
impl State<PodState> for Running {
    async fn next(
        self: Box<Self>,
        pod_state: &mut PodState,
        pod: &Pod,
    ) -> anyhow::Result<Transition<PodState>> {
        // Run forever
        loop {
            tokio::time::delay_for(std::time::Duration::from_secs(10)).await;
        }
    }

    async fn json_status(
        &self,
        pod_state: &mut PodState,
        pod: &Pod,
    ) -> anyhow::Result<serde_json::Value> {
        Ok(serde_json::json!(null))
    }
}

impl TransitionTo<Running> for Starting {}

struct Terminated;

#[async_trait::async_trait]
impl State<PodState> for Terminated {
    async fn next(
        self: Box<Self>,
        pod_state: &mut PodState,
        pod: &Pod,
    ) -> anyhow::Result<Transition<PodState>> {
        // TODO: interrupt workload
        Ok(Transition::Complete(Ok(())))
    }

    async fn json_status(
        &self,
        pod_state: &mut PodState,
        pod: &Pod,
    ) -> anyhow::Result<serde_json::Value> {
        Ok(serde_json::json!(null))
    }
}

struct ExampleProvider;

impl Provider for ExampleProvider {
    type PodState = PodState;

    type InitialState = Starting;

    type TerminatedState = Terminated;

    /// Use this hook to inject state shared between Pods into `PodState` 
    /// before it is passed to the state machine.
    async fn initialize_pod_state(
        &self, 
        pod: &Pod
    ) -> anyhow::Result<Self::PodState> {
        Ok(PodState)
    }
}
```

## Lessons Learned

We believe that this API leads to much more maintainable `Provider` 
implementations, and allows us to enjoy compile-time enforcement of our state 
machine constraints. We will continue to make less-intrusive refinements to 
this API, with the main goal of improving ergonomics.

One area for refinement, which we explored during this process and will 
continue to explore, is the use of macros for defining states. As it stands, 
there is a fair amount of boilerplate that must be implemented for each state. 
We found, however, that compiler errors originating from macros were extremely 
opaque, and we would like to identify a better solution. 

In general, we were pleasantly surprised by this "fearless refactoring". In our
planning phase, Rust's type system allowed us to develop specific functionality
using some placeholder types in Rust Playground, and be confident the code 
could drop into our larger codebase. Then, we added our new functionality and
worked iteratively to integrate it into the crate: commenting out some old 
code, finding all of its call sites and use cases, and then working in our new 
code. All in, the `kubelet` crate itself took around fifteen hours to convert.

Next, it was time to update our included WASI and waSCC Kubelets. We quickly 
discovered that the layout of our repository (all-in-one, using `cargo` 
workspaces) presented some issues. We could not split up our two conversions 
into separate pull requests without our test cases failing on both, it was 
clear that we had outgrown this monorepo style. We made do for this project, 
but have taken steps to begin splitting these crates into separate 
repositories. 

One final lesson, the waSCC Kubelet is slightly simpler than the WASI Kubelet. 
While designing the API to be as flexible as possible, I primarily referenced
the waSCC code, as we had decided that I would convert it before we attempted 
the WASI conversion. This resulted in some growing pains (felt mostly by Matt 
Fisher, who took the lead on the WASI conversion), as we found that container 
status patches were much more complex for this Kubelet, and were necessary to 
satisfy our end-to-end test suite. Going forward, we will be extending our 
state machine API to simplify the process of updating single container 
statuses.

## The Open Road

I hope that this has been an interesting deep dive into our new architecture, 
as well as some good lessons learned from the trenches. I was pleasantly 
surprised at the practicality of such an invasive refactoring in the Rust 
ecosystem. The team got a much better feel for some of the more nuanced aspects 
of Rust's type system, which will be very useful when trying to leverage it 
in the future.

Besides this release, the Krustlet project has a busy and exciting Fall 
scheduled, leading up to a `v1.0.0` release loosely planned for Q1 2021. Our 
major initiatives are: 

* Volume Mounting via Container Storage Interface (CSI).

* Networking via Container Networking Interface (CNI).

* Turn-key setup with several major Kubernetes flavors.

# References

[^Fisher]: [Introducing Krustlet, the WebAssembly Kubelet](https://deislabs.io/posts/introducing-krustlet/),

[^Squillace]: [WebAssembly meets Kubernetes with Krustlet](https://cloudblogs.microsoft.com/opensource/2020/04/07/announcing-krustlet-kubernetes-rust-kubelet-webassembly-wasm/)

[^wasmtime]: [Wasmtime](https://wasmtime.dev)

[^waSCC]: [waSCC](https://wascc.dev) 

[^krustlet-cri]: [krustlet-cri on GitHub](https://github.com/kflansburg/krustlet-cri)

[^CRI]: [Introducing Container Runtime Interface (CRI) in Kubernetes](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)

[^Pod Lifecycle]: [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)

[^Trait Objects]: [Object Safety Is Required for Trait Objects](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#object-safety-is-required-for-trait-objects)