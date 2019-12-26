# Redis Day Seattle Jan 2020
Rust / Redis PubSub / K8  API


Sample RUST / Redis code that will watch over Pods in the default cluster and write to PubSub when a condition 's occur.
## Installation
To use the openapi generated types:

```toml
[dependencies]
kube = { version = "0.21.2", features = ["openapi"] }
k8s-openapi = { version = "0.6.0", default-features = false, features = ["v1_15"] }
```

otherwise:

```toml
[dependencies]
kube = "0.21.2"
```

The latter is fine in a CRD-only use case.

## Usage
See the [examples directory](./examples) for how to watch over resources in a simplistic way.

**[API Docs](https://docs.rs/kube/)**

See [version-rs](https://github.com/clux/version-rs) for a super light (~100 lines), actix*, prometheus, deployment api setup.

See [controller-rs](https://github.com/clux/controller-rs) for a full actix* example, with circleci, and kube yaml.

NB: actix examples with futures are currently working with git/alpha dependencies.

## Api
It's currently recommended to compile with the "openapi" feature if you want an easy experience with accurate native object representations:

```rust
let pods = Api::v1Pod(client).within("default");

let p = pods.get("blog").await?;
println!("Got blog pod with containers: {:?}", p.spec.containers);

let patch = json!({"spec": {
    "activeDeadlineSeconds": 5
}});
let patched = pods.patch("blog", &pp, serde_json::to_vec(&patch)?).await?;
assert_eq!(patched.spec.active_deadline_seconds, Some(5));

pods.delete("blog", &DeleteParams::default()).await?;
```

See the `pod_openapi` or `crd_openapi` examples for more uses.

## Informer
The main abstraction from `kube::api` is `Informer<K>`. This is a struct with the internal behaviour for watching kube resources, but maintains only a queue of `WatchEvent` elements along with the last seen `resourceVersion`.

You tell it what type `KubeObject` implementing object you want to use. You can use `Object<P, U>` to get an automatic implementation by using `Object<PodSpec, PodStatus>`.`

The spec and status structs can be as complete or incomplete as you like. For instance, using the complete structs from [k8s-openapi](https://docs.rs/k8s-openapi/0.5.1/k8s_openapi/api/core/v1/struct.PodSpec.html):

```rust
type Pod = Object<PodSpec, PodStatus>;
let api = Api::v1Pod(client);
let inf = Informer::new(api.clone()).init().await?;
```

The main feature of `Informer<K>` is being able to subscribe to events while having a streaming `.poll()` open:

```rust
let pods = inf.poll().await?.boxed(); // starts a watch and returns a stream

while let Some(event) = pods.next().await { // await next event
    handle_event(&api, event?).await?; // pass the WatchEvent to a handler
}
```

How you handle them is up to you, you could build your own state, you can call a kube client, or you can simply print events. Here's a sketch of how such a handler would look:

```rust
async fn handle_event(pods: &Api<PodSpec, PodStatus>, event: WatchEvent<PodSpec, PodStatus>) -> anyhow::Result<()> {
    match event {
        WatchEvent::Added(o) => {
            let containers = o.spec.containers.into_iter().map(|c| c.name).collect::<Vec<_>>();
            println!("Added Pod: {} (containers={:?})", o.metadata.name, containers);
        },
        WatchEvent::Modified(o) => {
            let phase = o.status.phase.unwrap();
            println!("Modified Pod: {} (phase={})", o.metadata.name, phase);
        },
        WatchEvent::Deleted(o) => {
            println!("Deleted Pod: {}", o.metadata.name);
        },
        WatchEvent::Error(e) => {
            println!("Error event: {:?}", e);
        }
    }
    Ok(())
}
```

The [node_informer example](./examples/node_informer.rs) has an example of using api calls from within event handlers.

## Reflector
The other big abstractions exposed from `kube::api` is `Reflector<K>`. This is a cache of a resource that's meant to "reflect the resource state in etcd".

It handles the api mechanics for watching kube resources, tracking resourceVersions, and using watch events; it builds and maintains an internal map.

To use it, you just feed in `T` as a `Spec` struct and `U` as a `Status` struct, which can be as complete or incomplete as you like. Here, using the complete structs via [k8s-openapi's PodSpec](https://docs.rs/k8s-openapi/0.5.1/k8s_openapi/api/core/v1/struct.PodSpec.html):

```rust
let api = Api::v1Pod(client).within(&namespace);
let rf = Reflector::new(api).timeout(10).init().await?;
```

then you should `poll()` the reflector, and `state()` to get the current cached state:

```rust
rf.poll().await?; // watches + updates state

// Clone state and do something with it
rf.state().await.into_iter().for_each(|(name, p)| {
    println!("Found pod {} ({}) with {:?}",
        name,
        p.status.unwrap().phase.unwrap(),
        p.spec.containers.into_iter().map(|c| c.name).collect::<Vec<_>>(),
    );
});
```

Note that `poll` holds the future for 300s by default, but you can (and should) get `.state()` from another async context (see reflector examples for how to spawn an async task to do this).

If you need the details of just a single object, you can use the more efficient, `Reflector::get` and `Reflector::get_within`.

## Examples
Examples that show a little common flows. These all have logging of this library set up to `trace`:

```sh
# watch pod events
cargo run --example pod_informer --features=openapi
# watch event events
cargo run --example event_informer --features=openapi
# watch for broken nodes
cargo run --example node_informer --features=openapi
```

or for the reflectors:

```sh
cargo run --example pod_reflector --features=openapi
cargo run --example node_reflector --features=openapi
cargo run --example deployment_reflector --features=openapi
cargo run --example secret_reflector --features=openapi
cargo run --example configmap_reflector --features=openapi
```

for one based on a CRD, you need to create the CRD first:

```sh
kubectl apply -f examples/foo.yaml
cargo run --example crd_reflector --no-default-features
```

then you can `kubectl apply -f crd-baz.yaml -n default`, or `kubectl delete -f crd-baz.yaml -n default`, or `kubectl edit foos baz -n default` to verify that the events are being picked up.

For straight API use examples, try:

```sh
cargo run --example crd_api --no-default-features
cargo run --example crd_openapi --features=openapi
cargo run --example pod_openapi --features=openapi
```

## Timing
All watch calls have timeouts set to `10` seconds as a default (and kube always waits that long regardless of activity). If you like to hammer the API less, you can either call `.poll()` less often and the events will collect on the kube side (if you don't wait too long and get a Gone). You can configure the timeout with `.timeout(n)` on the `Informer` or `Reflector`.

## Raw Api
You can elide the large `k8s-openapi` dependency if you only are working with Informers/Reflectors, or you are happy to supply partial or complete definitions of the native objects you are working with:

```rust
#[derive(Deserialize, Serialize, Clone)]
pub struct FooSpec {
    name: String,
    info: String,
}
let foos = RawApi::customResource("foos")
    .version("v1")
    .group("clux.dev")
    .within("default");

type Foo = Object<FooSpec, Void>;
let rf : Reflector<Foo> = Reflector::raw(client, resource).init().await?;

let fdata = json!({
    "apiVersion": "clux.dev/v1",
    "kind": "Foo",
    "metadata": { "name": "baz" },
    "spec": { "name": "baz", "info": "old baz" },
});
let req = foos.create(&PostParams::default(), serde_json::to_vec(&fdata)?)?;
let o = client.request::<Foo>(req).await?;

let fbaz = client.request::<Foo>(foos.get("baz")?).await?;
assert_eq!(fbaz.spec.info, "old baz");
```

If you supply a partial definition of native objects then you can save on reflector memory usage.

The `node_informer` and `crd_reflector` examples uses this at the moment
, (although `node_informer` is cheating by supplying k8s_openapi structs manually anyway). The `crd_api` example also shows how to do it for CRDs.

## License
Apache 2.0 licensed. See LICENSE for details.
