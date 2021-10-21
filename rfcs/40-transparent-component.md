# Feature Name: `transparent-component`

## Summary

A new attribute is introduced to `#[derive(Component)]` which allows newtypes to be implictly converted to their inner type when being read or written in queries or accessors. A new attribute is introduced to `#[derive(Bundle)]` which allows such components to be automatically wrapped in their newtype by the `Bundle` `insert` impl.

## Motivation

Why are we doing this? What use cases does it support?

## User-facing explanation

The only scenario in which a *value* of the wrapper type is necessary is when inserting a component. This makes sense, because there could be multiple `Component`s wrapping a given inner type, and Bevy would have no way to discern which the user intends to insert. 

Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## Implementation strategy

### Basic ECS-level changes necessary

The fundamental change made by this RFC is adding new associated type `QueryItem` (name can be bikeshedded) to the `Component` trait. All sites which provide access to the value of a component `C` will be updated to return `C::QueryItem` rather than `C` itself. An exhaustive list of such sites appears to be:
- `EntityRef::get`
- `EntityMut::get_mut`
- `World::{get, get_mut}`
- `<ReadFetch<'w, 's, T> as Fetch>::{archetype_fetch, table_fetch}`
- `<WriteFetch<'w, 's, T> as Fetch>::{archetype_fetch, table_fetch}`

### Changes to procedural macros
TODO: Describe how to implement changes to procedural macros

### Preserving type inference
A naive implementation would attempt to implement this RFC's ECS-level changes by making the following modifications:
- Alter the implementation of `Fetch` for `ReadFetch` so that `<ReadFetch<'w, 's, T> as Fetch>::Item = &'w T::QueryItem` rather than `&'w T`.
- Alter the implementation of the change-detection smart pointer `Mut<T>` so that `<Mut<T> as Deref>::Target = T::QueryItem` rather than `T`. This permits us to leave `WriteFetch` alone, since its `Item` associated type is already `Mut<T>`.
- Alter the signature of `EntityRef::get::<T>` and `World::get::<T>` (which is implemented in terms of the former) to return `T::QueryItem`.

Unfortunately, the simplicity of this implementation comes at an unacceptable price. Previously, this code would compile:
```rust
#[derive(Component)]
struct Foo;

fn set_foo(world: &World, entity: Entity, foo: Foo) {
    *world.get_mut(entity).unwrap() = Foo;
}
```
The full signature of `World::get_mut` is `World::get_mut<T: Component>(&mut self, entity: E) -> Option<Mut<T>>`, so there is an inferred generic parameter here. However, if that signature remains unchanged while the `DerefMut` target type of `Mut<T>` becomes `T::QueryItem`, then Rust is no longer capable of inferring this generic parameter, as the type inference algorithm will not attempt to deduce a generic parameter knowing only an associated type of that parameter. This would force *all* uses of `World::get_mut` and similar, whether with a transparent component or otherwise, to explicitly specify the `Component` type with a turbofish. This is considered ergonomically unacceptable. 

In order to preserve type inference, `Mut<T>`'s `Deref` impl will *not* perform the conversion from `T` to `T::QueryItem`. Instead, it will be made possible (TODO: how?) to construct a `Mut<T::QueryItem>` directly from storage for `T`. The methods `get` and `get_mut` on `World`, `EntityRef`, and `EntityMut` will be changed from inherent methods to trait methods from traits `GetEntityComponent<T, Q>`, `GetComponent<T, Q>`, `GetComponentMut<T, Q>`. 

```
pub trait ComponentAccess<T, Q> where T: Component<QueryItem=Q> {
    // TODO
}
```
Each invocation of the `#[derive(Component)]` macro on a type `T` will generate an appropriate `impl ComponentAccess<T, T::QueryItem> for World`. 

TODO

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What objections immediately spring to mind? How have you addressed them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect Bevy as a whole in a holistic way.
Try to use this section as a tool to more fully consider other possible
interactions with the engine in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
If a feature or change has no direct value on its own, expand your RFC to include the first valuable feature that would build on it.
