---
title: "Krator: My God, it's Full of States!"
description: "Building state-machine-based Operators in Rust"
date: 2021-01-29 00:00:00 +0000 UTC
authorname: "Kevin Flansburg"
author: "@kflansburg"
authorlink: "https://kflansburg.com/"
image: "images/twitter-card.png"
tags: ["krustlet", "rust"]
---

## Announcing Krator

Last September the Krustlet project refactored our API to employ a state
machine pattern for describing the lifecycle of Pods. The design of this API
was covered in depth in my
[previous blog post](https://deislabs.io/posts/a-fistful-of-states/).
Since then we have made a number of refinements to this interface and extended
our use of the pattern to individual containers within each Pod. These changes
also served to simplify and make more generic our API, and were done in part as
a stepping stone to the crate we are announcing today.

Since introducing this API, we have experienced positive feedback from the
community, including a number of requests to generalize this state machine
architecture so that it can be used to build Kubernetes Operators. It turned
out to be fairly straightforward to extract the "operator" portion of Krustlet
to a new crate, and today we are pleased to announce Krator (pronounced
crater), which stands for [K]ubernetes [R]ust st[at]e machine Operat[or].
Krator will be released alongside the next release of Krustlet, `0.6.0`, at
which point Krustlet will use Krator internally to watch Pods and manage
their state machines.

Implementing an Operator involves a number of moving parts, so I have put
together a fairly minimal example, which I will walk through in the rest of
this post. If you just want to browse the code, it currently lives in the
Krustlet GitHub repository
[here](https://github.com/deislabs/krustlet/tree/master/crates/krator), and
documentation can be found on [docs.rs](https://docs.rs/krator) (once Krustlet
`0.6.0` is released).

## Writing an Operator

As an example, we will create an operator to manage `Moose` custom resources.
Using Krator, we will write a state machine that will describe the possible
states of a `Moose`, and how it should transition between these states. First,
we must write a custom resource definition, which is very easy using the
`CustomResource` derive macro from the `kube` crate!

```rust
use kube_derive::CustomResource;
use serde::{Deserialize, Serialize};

#[derive(CustomResource, Debug, Serialize, Deserialize, Clone, Default)]
#[kube(
    group = "animals.com",
    version = "v1",
    kind = "Moose",
    derive = "Default",
    status = "MooseStatus",
    namespaced
)]
struct MooseSpec {
    /// Shoulder height in meters.
    height: f64,
    /// Body weight in kilograms.
    weight: f64,
    antlers: bool,
}
```

Each `Moose` has a spec which includes height, weight, and whether it has
antlers. `CustomResource` lets us also specify that a resource should support
the Kubernetes `status` API, which is required for a Krator Operator. We
specified that our status type was `MooseStatus` which we will define as
follows:

```rust
#[derive(Debug, Serialize, Deserialize, Clone)]
enum MoosePhase {
    Asleep,
    Hungry,
    Roaming,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct MooseStatus {
    phase: Option<MoosePhase>,
    /// ~ Moose noises ~
    message: Option<String>,
}
```

As our state machine transitions to new states, Krator will automatically
update the object's status with the Kubernetes API. To do this, Krator requires
that this status type implement a trait, `ObjectStatus`:

```rust
impl ObjectStatus for MooseStatus {
    /// Use to generate an error status if an unhandled error occurs.
    fn failed(e: &str) -> MooseStatus {
        MooseStatus {
            message: Some(format!("Error tracking moose: {}.", e)),
            phase: None,
        }
    }

    /// Converts the arbitrary status type to a JSON patch to send to
    /// Kubernetes.
    fn json_patch(&self) -> serde_json::Value {
        // Generate a map containing only set fields.
        let mut status = serde_json::Map::new();

        if let Some(phase) = self.phase.clone() {
            status.insert("phase".to_string(), serde_json::json!(phase));
        };

        if let Some(message) = self.message.clone() {
            status.insert("message".to_string(), serde_json::Value::String(message));
        };

        // Create status patch with map.
        serde_json::json!({ "status": serde_json::Value::Object(status) })
    }
}
```

Next we need to define data types for:

- Data that is shared across state handlers for a single object (moose in this
  case)
- Data that is shared across state machines (shared between objects)

We must also implement a trait, `ObjectState` which lets Krator know how all
of these types relate to the state machine.

```rust
struct MooseState {
    /// All moose names start with 'M'
    name: String,
    /// Food mass in kilograms.
    food: f64,
}

struct SharedMooseState {
    /// Map moose name to set of moose friend's names (moose friendships are directed).
    friends: HashMap<String, HashSet<String>>,
}

#[async_trait::async_trait]
impl ObjectState for MooseState {
    type Manifest = Moose;
    type Status = MooseStatus;
    type SharedState = SharedMooseState;

    /// Clean up data related to a specific object.
    async fn async_drop(self, shared: &mut Self::SharedState) {
        shared.friends.remove(&self.name);
    }
}
```

We are now ready to implement our state machine. Below is a diagram of the
states we will represent. When a `Moose` object is first created, Krator will
initialize the state machine in the `Tagged` state. This will populate our
shared state and then transition to the `Roam` state. The moose will roam the
wilderness and with some probability encounter other moose and become friends.
Eventually, the moose will get hungry and transition to the `Eat` state, which
will replenish its `food` level but also make it tired. It will then transition
to the `Sleep` state for 60 seconds before returning to the `Roam` state. Note
that the final state, `Released`, is only reached when the `Moose` object is
marked for deletion in the Kubernetes API and Krator interrupts the state
machine and transitions to `Released`. In some operators, end states are also
reachable under certain conditions from within the state machine graph, such as
a Pod transitioning to `Completed`.

![Moose State Machine](/images/moose-state-machine.svg)

State implementations are currently a little verbose, so I will include a
single state implementation here. The rest can be found in
[the complete example](https://github.com/deislabs/krustlet/blob/master/crates/krator/examples/moose.rs).
The important information is within the `next` and `status` methods of each
trait implementation. The `status` method is called when each state is
*entered*, so it should describe the state it is associated with. We have
considered using macros for implementing `State`, but have not been satisfied
with Rust's error messages for errors originating from within these macros. If
you have any experience with macros and think you can come up with something
nice, please let us know!

The `next` method is the main code block for each state, sometimes referred to
as the state handler. This method has a number of important arguments:

- It takes ownership of `self`.
- An `Arc<RwLock<_>>` to access shared state. For more information on this API,
  see the
  [Tokio documentation](https://docs.rs/tokio/1.0.2/tokio/sync/struct.RwLock.html).
- A mutable reference to the object state.
- A `Manifest<_>` which acts as a
  [reflector](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes).
  This type implements the `Stream` API for detecting changes to the object
  manifest within a state handler, or you can simply call `latest` to obtain
  a clone of the current object manifest.

```rust
#[derive(Debug, Default)]
/// Moose is roaming the wilderness.
struct Roam;

async fn make_friend(
    name: &str,
    shared: &Arc<RwLock<SharedMooseState>>
) -> Option<String> {
    let mut mooses = shared.write().await;
    for (other_moose, friends) in mooses.friends.iter_mut() {
        if name == other_moose {
            continue;
        }
        if !friends.contains(name) {
            friends.insert(name.to_string());
            return Some(other_moose.to_string());
        }
    }
    return None;
}

#[async_trait::async_trait]
impl State<MooseState> for Roam {
    async fn next(
        self: Box<Self>,
        shared: Arc<RwLock<SharedMooseState>>,
        state: &mut MooseState,
        _manifest: Manifest<Moose>,
    ) -> Transition<MooseState> {
        loop {
            tokio::time::delay_for(std::time::Duration::from_secs(5)).await;
            state.food -= 5.0;
            if state.food <= 50.0 {
                return Transition::next(self, Eat);
            }
            let r: f64 = {
                let mut rng = rand::thread_rng();
                rng.gen()
            };
            if r < 0.05 {
                if let Some(other_moose) = make_friend(&state.name, &shared).await {
                    info!("{} made friends with {}!", state.name, other_moose);
                }
            }
        }
    }

    async fn status(
        &self,
        _state: &mut MooseState,
        _manifest: &Moose,
    ) -> anyhow::Result<MooseStatus> {
        Ok(MooseStatus {
            phase: Some(MoosePhase::Roaming),
            message: Some("Gahrooo!".to_string()),
        })
    }
}
```

To finish our state machine, we must define valid transitions between states.
The compiler will ensure that we do not make invalid transitions, which helps
us to avoid undefined behavior.

```rust
impl TransitionTo<Roam> for Tagged {}
impl TransitionTo<Eat> for Roam {}
impl TransitionTo<Sleep> for Eat {}
```

Alternatively, Krator provides a macro for deriving this trait:

```rust
#[derive(Debug, Default, TransitionTo)]
#[transition_to(Roam)]
/// Moose is sleeping.
struct Sleep;
```

Finally we must implement the `Operator` trait to tie this all together. Note
that the state machine API itself does not require that the `Manifest` is a
valid Kubernetes API resource. This allows the state machine pattern to be used
with a broader set of types (we used it for containers, which are not a
Kubernetes resource type). This, however, will require you to reimplement the
function which drives the state machine, as we did for
[containers](https://github.com/deislabs/krustlet/blob/3b120ec6a00123afd1407d049b61282c67e71676/crates/kubelet/src/container/state.rs#L18).

To get the full benefit of Krator, which watches for changes to your API
resource, spawns state machines, and handles status updates, you will need to
satisfy additional trait bounds which are placed on associated types of the
`Operator` trait. Namely, `Manifest` must be
[`Metadata<Ty = ObjectMeta>`](https://docs.rs/k8s-openapi/0.10.0/k8s_openapi/trait.Metadata.html),
`DeserializeOwned`, and `Default`. `Status` must be `ObjectStatus` (as
described above).

```rust
struct MooseTracker {
    shared: Arc<RwLock<SharedMooseState>>,
}

impl MooseTracker {
    fn new() -> Self {
        let shared = Arc::new(RwLock::new(SharedMooseState {
            friends: HashMap::new(),
        }));
        MooseTracker { shared }
    }
}

#[async_trait::async_trait]
impl Operator for MooseTracker {
    type Manifest = Moose;
    type Status = MooseStatus;
    /// This indicates the starting state of the state machine.
    type InitialState = Tagged;
    /// This is the state that is transitioned to when the object is deleted.
    type DeletedState = Released;
    type ObjectState = MooseState;

    /// Initialize an object state for a new object.
    async fn initialize_object_state(
        &self,
        manifest: &Self::Manifest,
    ) -> anyhow::Result<Self::ObjectState> {
        let name = manifest.metadata().name.clone().unwrap();
        Ok(MooseState { name, food: 100.0 })
    }

    /// Obtain a reference to the SharedState.
    async fn shared_state(&self) -> Arc<RwLock<SharedMooseState>> {
        Arc::clone(&self.shared)
    }
}
```

To run our operator, we initialize an `OperatorRuntime` and `await` on its
`start` method which will run forever.

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    env_logger::init();
    let kubeconfig = kube::Config::infer().await?;
    let tracker = MooseTracker::new();

    // Only track mooses in Glacier NP
    let params = ListParams::default().labels("nps.gov/park=glacier");

    let mut runtime = OperatorRuntime::new(&kubeconfig, tracker, Some(params));
    runtime.start().await;
    Ok(())
}
```

And that is it! While this was a lot to cover in a blog post, we have found
that this pattern leads to a very navigable codebase for even complex operators
like Kubelets. We believe that this framework eliminates a lot of the
error-prone state management involved when writing Operators and can get you
off the ground very quickly. If you have questions or feature suggestions,
please reach out to us on [GitHub](https://github.com/deislabs/krustlet) or
[#krustlet](https://kubernetes.slack.com/app_redirect?channel=krustlet) on
Slack.
