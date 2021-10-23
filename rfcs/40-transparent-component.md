# Feature Name: `transparent-component` (WIP)

## Summary

A new attribute is introduced to `#[derive(Component)]` which allows newtypes to be implictly converted to their inner type when being read or written in queries or accessors. A new attribute is introduced to `#[derive(Bundle)]` which allows such components to be automatically wrapped in their newtype by the `Bundle::insert` impl. (TODO: how does bundle insert actually work, I haven't looked :P)

## Motivation

<!--- Why are we doing this? What use cases does it support? -->

TODO

## User-facing explanation

<!---
Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.
-->

TODO

The only scenario in which a *value* of the wrapper type is necessary is when inserting a component. This makes sense, because there could be multiple `Component`s wrapping a given inner type, and Bevy would have no way to discern which the user intends to insert. 

## Implementation strategy

### Necessary ECS-level changes

The fundamental change made by this RFC is adding new associated type `QueryItem` (name can be bikeshedded) to the `Component` trait. All sites which provide access to the value of a component `C` will be updated to return `C::QueryItem` (sort of, see below) rather than `C` itself. An exhaustive list of such sites appears to be:
- `EntityRef::{get, get_mut_unchecked}`
- `EntityMut::{get_mut, get_mut_unchecked}`
- `World::{get, get_mut}`
- `<ReadFetch<'w, 's, T> as Fetch>::{archetype_fetch, table_fetch}`
- `<WriteFetch<'w, 's, T> as Fetch>::{archetype_fetch, table_fetch}`

### Preserving type inference
A naive implementation of this RFC would make the following ECS-level changes:
- Alter the implementation of `Fetch` for `ReadFetch` so that `<ReadFetch<'w, 's, T> as Fetch>::Item = &'w T::QueryItem` rather than `&'w T`.
- Alter the implementation of the change-detection smart pointer `Mut<T>` so that `<Mut<T> as Deref>::Target = T::QueryItem` rather than `T`. This permits us to leave `WriteFetch` alone, since its `Item` associated type is already `Mut<T>`.
- Alter the signature of `EntityRef::get::<T>` and `World::get::<T>` (which is implemented in terms of the former) to return `T::QueryItem`.

Unfortunately, the simplicity of this implementation comes at a significant price. Previously, this code would compile:
```rust
#[derive(Component)]
struct Foo;

fn set_foo(world: &World, entity: Entity, foo: Foo) {
    *world.get_mut(entity).unwrap() = Foo;
}
```
The full signature of `World::get_mut` is `fn get_mut<T: Component>(&mut self, entity: E) -> Option<Mut<T>>`, so there is an inferred generic parameter here. However, if that signature remains unchanged while the `DerefMut` target type of `Mut<T>` becomes `T::QueryItem`, then Rust is no longer capable of inferring this generic parameter. This is because the compiler cannot 'look backwards' through associated types. The compiler will never attempt to deduce `T` from the knowledge that `<T as Component>::QueryItem = Foo`, even if only one `impl Component` with that value for `QueryItem` exists. As a result, keeping this signature would force *all* uses of `World::get_mut` and similar, whether with a transparent component or otherwise, to explicitly specify the `Component` type with a turbofish. This is considered an unacceptable ergonomic loss. 

In order to preserve type inference, we must ensure that the type variable which corresponds to the return type of `get_mut` is not 'hidden' behind an associated type. This means two things: first, we cannot perform the conversion from `T` to `T::QueryItem` as part of `Mut`. `get_mut` will have to return a `Mut<T::QueryItem>` directly. Second, we will have to 'lift' the associated type `T::QueryItem` into a full type parameter on `World::get_mut`. This is achieved by introducing a new trait `AsComponent` and altering the signature of `get_mut` to the following:
```rust
fn get_mut<T: Component, U: AsComponent<T>>(&mut self, entity: E) -> Option<Mut<U>>
```
We will get to the precise definition of `AsComponent` later, but an implementation of `AsComponent<C> for T` should be understood to mean "the type `T`" can be read from ECS storage labeled by the component `C`. Thus, now when using `get_mut`, you must now provide two pieces of information: the component whose data you would like to read from ECS storage (`T`), and *how* (i.e. as what type) you would like to read it. Introducing an additional variable may sound like it would only make the situation with inference worse, but in fact the type inference algorithm is far less conservative in this case. In particular, if `U` is known, and there is only one implementation of `AsComponent` for `U`, the compiler will happily deduce that `T` is this impl's generic. (This also works backwards in most cases.) This property allows the `fn set_foo` written to compile without changes.

### The `AsComponent` trait
The trait `AsComponent` is defined as follows:
```rust
pub trait AsComponent<C: Component> {
    fn map_ref(component: &C) -> &Self;
    fn map_mut(component: &mut C) -> &mut Self;
}
```
As previously stated, an implementation of `AsComponent<C> for T` should be understood to mean "the type `T`" can be read from ECS storage labeled by the component `C`. Consequently, `AsComponent<Self>` is added as a required bound on `Component::QueryItem`. The methods `map_ref` and `map_mut` are used to produce a `&T::QueryItem` or `Mut<T::QueryItem>` from the reference to `T` obtained from ECS storage. 

There will be a blanket `impl<T: Component> AsComponent<T> for T`, encoding the fact that all components can be read from ECS storage as themself. Note that this impl cannot be changed to `impl<T: Component> AsComponent<T> for T::QueryItem`, because this would again require type inference to search for a `T` with only knowledge of one of its associated types. Such an impl is also useful in its own right: the lack of injectivity between component 'labels' and component value types could complicate certain code that is generic over a type implementing `Component`. With the above impl, such code can choose to access components in a way that does preserve injectivity when using `World::get_mut` or similar methods. Additionally, the blanket impl ensures that users who write their own `Component` impls do not have to additionally write `impl AsComponent<MyComponent> for MyComponent`, a small ergonomic win. Finally, it also prevents downstream crates from adding impls like `impl<T: Component> AsComponent<MyType> for T`, which would otherwise break type inference for all component types when in scope. 

### Changes to procedural macros
TODO: Describe how to implement changes to procedural macros

### Preserving type inference even better

TODO: Describe three-trait solution for perfectly backwards-compatible method calls

## Drawbacks

<!--- Why should we not do this? -->

TODO

## Rationale and alternatives

<!---
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What objections immediately spring to mind? How have you addressed them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?
-->

TODO

## Prior art
<!---
Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.
-->

TODO

## Unresolved questions
- Is it worth moving component-accessing methods on `World`, `EntityRef`, and `EntityMut` to three new traits to achieve perfectly backwards-compatible method signatures, or is changing `world.get::<T>` to `world.get::<_, T>` acceptable?
- Would it be better to have `AsComponent<C: Component<QueryItem=Self>>` and replace the blanket impl with `impl<T: Component<QueryItem=Self>> AsComponent<T> for T`? Would this even work?
- How will inserting and removing transparent components work?
- How do transparent components interact with bundles?

## Future possibilities

If `#[derive(Resource)]` becomes a thing, we may or may not want to support an analogous feature of 'transparent resources.' 
