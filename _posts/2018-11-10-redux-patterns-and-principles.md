---
layout: post
slug: redux-patterns-and-principles
title: Redux patterns and principles
date: 2018-11-10
---

In this blog we look at how to use Redux effectively in a React App. The unidirectional circular data flow
in the Redux pattern is helpful in simplifying the reasoning around a UI App. However the benefits are
frequently discarded by not using Redux effectively, and result a compromised user experience or performance and/or the need for complex workarounds. We discuss how to use the Redux effectively to avoid common pitfalls, and delivery a responsive, flicker-free user experience with minimal spinners.

Much of the complexity in a front end application, revolves around fetching data from the backend. This is not something that is
discussed frequently, it is almost accepted as a given that this is complexity that simply needs to be managed. However with some
thought and planning, it turns out that a lot of this complexity can be localised, contained, or abstracted away, leaving the front end
developer to get on with the job of making it look fantastic.

The biggest issue revolves around data validity and refreshes. A strategy is required to manage this effectively.
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
* Friends. This is a map of user ID to a list of user IDs
* Groups. Map of group details objects, keyed by group ID
* Group memberships. Map of group ID to a list of user IDs
* Auth info. A singleton object consisting of the user ID of the currently logged in user, and an auth token.

For this simple domain model, here is an example:

```javascript
{
    auth: {
        data: { user: 1, token: 'token' }
        invalid: false
    },
    users: {
        data: {
            1: { userId: 1, name: 'Alice', email: 'alice@alice.com' }
            2: { userId: 2, name: 'Bob'  email: 'bob@bob.com' }
        },
        invalid: [ 2 ]
    },
    groups: {
        details: {
            data: {
                101: { groupId: 101, name: 'Chess' }
                102: { groupId: 102, name: 'Boxing' }
            }
            invalid: [ 101 ]
        }
        members: {
            data: {
                1: [ 101 ]
                2: [  101, 102 ]
            }
            invalid: [ 1 ]
        }
    }
}
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

These would be the projections of our data store:

###### All data

```javascript
{
    auth: {
        user: 1, token: 'token'
    },
    users: {
        1: { userId: 1, name: 'Alice', email: 'alice@alice.com' }
        2: { userId: 2, name: 'Bob'  email: 'bob@bob.com' }
    },
    groups: {
        details: {
            101: { groupId: 101, name: 'Chess' }
            102: { groupId: 102, name: 'Boxing' }
        }
        members: {
            1: [ 101 ]
            2: [  101, 102 ]
        }
    }
}
```

###### Valid data

```javascript
{
    auth: {
        data: { user: 1, token: 'token' }
    },
    users: {
        1: { userId: 1, name: 'Alice', email: 'alice@alice.com' }
    },
    groups: {
        details: {
            102: { groupId: 102, name: 'Boxing' }
        }
        members: {
            2: [  101, 102 ]
        }
    }
}
```

So beyond a certain point, we never need to expose the data with any flags, we simply pass the *valid data* to the data fetcher, and *all data* to the renderer.

Finally, we consider when rendering needs to take place. It should be as simple as selecting a subset of the redux store, and rendering when this data changes. This is in the `shouldComponentUpdate` lifecycling method. But what does it mean when the data changes? JavaScript doesn't by default do deep comparisons. We could use one of the 3rd party tools available, or roll our own to do it. But a better option is to do a small amount of extra work in the reducer.

So the final Redux principle is the concept of maintaining reference equality. This means that on any update to the redux store, the reducer checks that if an object has changed, and never overwrites any object in the redux store that hasn't changed. If our reducers can offer this guarantee, we can propagate this guarantee right through the component heirarchy. It means we can trivially make all our components
[Pure components](https://reactjs.org/docs/react-api.html#reactpurecomponent). This is a substantial performance enhancement. The price to pay for this is the reducer has to do a little extra work. This work only has to be done once, whereas the `shouldComponentUpdate` is generally called orders of magnitude more often than the work performed by reducers.

In a previous article, I advocated for a style of react development involving higher order components. This compositional style is well
suited to realising these performance optimisations.

To get this all to work we're going to need some boiler plate. First is an immutable update function, which we call `immutableUpdate`. This function takes `start` and a `change` objects as input. This is kind of an enhanced version of `Object.assign`, with behaviour similar to `Object.assign({}, start, change)`, that will merge the `start` and `change` objects, with the `change` object getting preference.

However it also perserves object instances, of both outer and inner objects, so preserving reference equality wherever possible.

To do it have the following additional properties:
* If `start` and `change` coincide (in terms of deep equality) `start` is returned as is
* If there is any discrepancy between `start` and `change`, a new object is returned. But reference equality is preserved for all portions of `start` where there is no conflict
* The input arguments are never modified

To make this clear, we expect, for example, the following [Jest](https://jestjs.io) test to pass:

```javascript
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

Then all reducers take advantage of this `immutableUpdate` in their implementation.

Then for the typical component we have the following steps:

1. Connect to redux store
1. Select the subset of the data store we are interested in with our `selector` function.

    This function must only pick select existing objects, and not create any new objects.
1. Call `shouldUpdate` to test data for shallow equals
1. Split the data into the "valid" and the "all" projections.
1. If "valid" data is incomplete, fetch more data.
1. If there is an error, display some sort of error message.
1. If "all" data is incomplete, display a spinner.
1. If "all" data is complete, render it.

Here's what it might look like in Recompose based pseudocode.

```javascript
export default compose(
    connect( ( state ) => ( {
        selectedData: selector( state ),
    } ), dispatch => ( { dispatch } ) ),
    shouldUpdate( ( props, nextProps ) =>
        !shallowEqual( props.__selectedData, nextProps.__selectedData ) ),
    // everything from here down will only happen when data data change
    withProps( ( { selectedData } ) => ( { validAndAll: resolveValidData( selectedData ) } ) ),
    withProps( ( { validAndAll } ) => ( { validAndAll: { allData, validData } } ) ),
    fetchMissingData( validData ),
    transformAndAddHandlers( allData ))( PureViewComponent )
```

`resolveValidData` is a function that splits `selectedData` into `{ allData, validData }` as discussed before.

The key point is in `shouldUpdate` we do a shallow comparison of our `selectedData`, and if it's equal, we don't go any further.
This is a very effective optimisation that should offer significant performance benefits.

Of course there's a lot going on there, and in particular, `fetchMissingData` can be
be very involved. But it is possible to do this in a generic way so we only have to write this code once. Perhaps that's worth a post on it's own at some point.

But we don't give away all the secret sauce here - there's too much going on for that!

To summarise -here are the key points:

1. Never throw away any data in the Redux store unless we know exactly what to replace it with. Instead we flag it as invalid.
2. Strive for a flat, normalised data graphical state store data structure, with all data uniquely represented
3. Recognise that there are essentially 2 projections of the redux store, the valid data, needed for the data fetcher, and everything else. Let the view be eventually consistent
4. Never throw away or replace any data in the state store that hasn't changed. Preserve reference equality and make your components pure

I hope these ideas are somewhat helpful, and at least help get some thought and planning going into how to simplify data flows in React applications.
