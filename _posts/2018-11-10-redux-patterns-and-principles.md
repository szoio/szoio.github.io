---
layout: post
slug: react-and-higher-order-components
title: React patterns and principles
date: 2018-11-10
---

In this blog we look at how to use Redux effectively in a React App. The unidirectional circular data flow
in the Redux pattern is helpful in simplifying the reasoning around a UI App. However the benefits are
frequently discarded by not using Redux effectively, and result a compromised user experience or performance and/or the need for complex workarounds. We discuss how to use the Redux effectively to avoid common pitfalls, and delivery a responsive, flicker-free user experience with minimal spinners.

The fundamental issue revolves around data validity and refreshes. A strategy is required to manage this effectively.
Data in a React app is generally loaded on demand by calling into a backend webservice.
This is done when entering a page for the first time. This is generally handled by fetching the data,
and giving some user feedback that some loading is occurring. Often something like a spinner is displayed in the meantime.
This is ok first time around, though there are nicer ways in some cases.

Logic is required in the frontend components to detect if all the required data is present. If data is missing, rendering is postponed, and data is fetched. This may involve several round trips, as the first request may, for example, request a list of items, and subsequent requests may be required to fetch individual items.

But what about when the user navigates away, and then returns to the same page? The ideal experience is:
* The user goes straight back to what they saw before, and can carry on doing things immediately
* Changes that the front end knows about are observable immediately (e.g. form submitted with new data). This is known as *optimistic rendering*
* If there are external data changes, these are loaded asynchronously in the background. The user is not required to refresh the page to get these
* Updated data is rerendered as soon as they are available, and doesn't disturb any part of the screen except the part that gets changed
* There is no flickering and minimal UI props and crutches like spinners

Apps are like cars, even more important than how they look is how they drive. If you observe closely, the most pleasant UXs to drive will essentially work this way, and there are plenty that don't.

The first big mistake is to delete the data in the store to force the frontend to call the backend for updated data. If you are doing this, you shoudn't be using Redux. It is better in this case to just use local state.

So fundamentally we if we don't delete data, how do we force a data fetch? One option is to do it automatically every time. But this is hard to control, and may result in multiple fetches that are difficult to control.

So that brings us to the first principle. *A validity indicator.* Every datum in the redux store needs an indicator, some kind of flag, to denote whether it is valid or not. A bit of ugliness, but a necessary evil. The flag needs to be at the appropriate level of granularity. This granularity should correspond roughly with the level of granularity of the data fetches themselves, or slightly greater, but not less.

The next common mistake is to give little thought to structure of the data in the redux store itself. Often whatever is returned from the webservice is simply dumped into the some part of the store as-is. The problem is data returned from web APIs is often quite denormalised.
For the redux store, our preference is for data have a *unique representation* in the redux store, and this is our second principle. We only want to fetch data from one place and we only want to have to invalidate a datum in one place.
Having a unique representation, with a high level of granularity means the store needs to be relatively normalised.

The following conceptual module works well for the redux store structure. Data is one of:

* *Singleton object*. A single instance for a given entity type.
* *Object keyed by id*. State store for a given entity type consists of a map of objects keyed by entity ID.
* *Entity relationship*. Any instance of entity `A` can be associated with instances of entity `B` via a map of IDs of `A` to lists of IDs of `B`.

This reduces the redux store to a flat, relatively normalised graph-like structure.

For example, consider a simple social media domain model, consisting of users and groups. The redux store may consist of:

* Users. Map of user details objects, keyed by user ID
* Friends. This is a map of user ID to a list of user IDs.
* Groups. Map of group details objects, keyed by group ID.
* Group memberships. Map of group ID to a list of user IDs.

```JavaScript
{a: }
```

Note that the question may arise about how to represent reflexive relationships. E.g. for group memberships we could store:

* Map of group ID to a list of user IDs (list of users belonging to a group).
* Map of user ID to a list of group IDs (list of groups a user belongs to).
* Both of the above.

Which of these we choose would be a subjective choice, depending on how the data in the store is updated, and how it is used. If we choose the 3rd option, we would need a very good reason, as it violates the principle of unique representation.

The next point is the observation that any data we get out of the state store has two essential purposes.
* Determining whether we need to go to the backend service to fetch or refresh data
* Passing to view components for rendering, often via some transformations

Further to this, from this observation is that any subset of the redux store data has two projections:
* A *valid data* projection, which is the the available data with all data flagged as invalid filtered output
* An *all data* projection, in which we don't take notice of the invalid flag.

So we never need to expose the the data with any flags, we simply pass the *valid data* to the data fetcher, and *all data* to the renderer.

Finally, we consider when rendering needs to take place. It should be as simple as selecting a subset of the redux store, and rendering when this data changes. This is in the `shouldComponentUpdate` lifecycling method. But what does it mean when the data changes? JavaScript doesn't by default do deep comparisons. We could use one of the 3rd party tools available, or roll our own to do it. But a better option is to do a small amount of extra work in the reducer.

So the final Redux principle is the concept of maintaining refrence equality. This means that on any update to the redux store, the reducer checks that if an object has changed, and never overwrites any object in the redux store that hasn't changed. If our reducers can offer this guarantee, we can propagate this guarantee right through the component heirarchy. It means we can trivially make all our components
[Pure components](https://reactjs.org/docs/react-api.html#reactpurecomponent). This is a substantial performance enhancement. The price to pay for this is the reducer has to do a little extra work. This work only has to be done once, whereas the `shouldComponentUpdate` is generally called orders of magnitude more often than the work performed by reducers.

To get this all to work we're going to need some boiler plate. First is an immutable update function, which we call `immutableUpdate`. This function takes `start` and a `change` objects as input. This is kind of an enhanced version of `Object.assign`, with behaviour similar to `Object.assign({}, start, change)`, that will replace the merge the `start` and `change` objects, with the `change` object getting precedence.

However it also preserves reference equality wherever possible.

To do it have the following additional properties:
* If `start` and `change` coincide (in terms of deep equality) `start` is returned as is
* If there is any discrepancy between `start` and `change`, a new object is returned. But reference equality is preserved for all portions of `start` that are which coincide with start
* The input arguments are never modified

To make this clear, we expect, for example, the following [Jest](https://jestjs.io) test to pass:

```JavaScript
test( 'immutable update', () => {
    const ghi = { g: { h: 'i' } }
    const start = {
        a: 1,
        b: { c: 2, d: 3 },
        e: [ 1, 2, 3, 4 ],
        f: ghi
    }

    // if we update the original content with the existing content for any key, the original object is kept
    expect( immutableUpdate( start, { a: 1 } ) === start ).toBe( true )
    expect( immutableUpdate( start, { b: { c: 2, d: 3 } } ) === start ).toBe( true )
    expect( immutableUpdate( start, { a: 1, b: { c: 2, d: 3 } } ) === start ).toBe( true )
    expect( immutableUpdate( start, { e: [ 1, 2, 3, 4 ] } ) === start ).toBe( true )
    expect( immutableUpdate( start, { f: { g: { h: 'i' } } } ) === start ).toBe( true )

    // if we update the new content, the new content is copied
    let updated = immutableUpdate( start, {
      a: 2,
      f: { g: { h: 'i' }
    } } )

    // new values will be updated
    expect( updated.a ).toBe( 2 )
    // however reference equality is maintained for other fields that are not modified
    expect( updated.f ).toBe( ghi )
}
```
*Note that the Jest operator [`toBe`](https://jest-bot.github.io/jest/docs/expect.html#tobevalue) uses the [`Object.is`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is)
operator for comparisons.*

An embarrassingly ugly implementation of this function, that uses [deep-equal](https://github.com/substack/node-deep-equal) for object comparisons, is available [in this Gist](https://gist.github.com/szoio/c2d26c6a8ddac508bb4eb8ea1e5974d7).
