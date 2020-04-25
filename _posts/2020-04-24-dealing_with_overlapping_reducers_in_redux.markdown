---
layout: post
title:      "Dealing with overlapping reducers in Redux"
date:       2020-04-24 22:56:14 -0400
permalink:  dealing_with_overlapping_reducers_in_redux
---


My final project for Flatiron got a lot more complicated than I was expecting it to with reducers, and there were many little finnicky things here that tripped me up, even into the last hours of coding! This made me think it might be a good thing to write a blog about. 

To get briefly into the context, my project is a novel tracker app where users can keep track of their writing progress when participating in a novel writing contest. The website's functionality depends on having access to: 
* a current user 
* their current novel (from this year, as the contest is meant to run every year)
* all the novels in this year's contest 
* (it also includes a few other things, but we don't need to worry about them here!)

With Redux, I could keep all of these things in store, which can be accessed from any component in the app--super handy!--and with Redux's `combineReducers` function, I could make a store with different reducers for each key I wanted in store. 

```
import { combineReducers } from 'redux'
import allCurrentNovels from './allCurrentNovels'
import currentUser from './currentUser'
import currentNovel from './currentNovel'

const rootReducer = combineReducers({
  currentUser,
  allCurrentNovels,
  currentNovel
})

export default rootReducer;
```

The fun part came when I decided I wanted the `currentNovel` to also be part of `allCurrentNovels`, so that I could easily access the main novel in question for most of the app but also have that novel be visible to the user on the main page along with all the other novels in the contest. 

Basically: when using `rootReducer`, each reducer going into it is responsible for its own piece of the store. `currentNovel` is either set to `null` or a user's novel. `allCurrentNovels` includes everything. What this means practically is that every change to `currentNovel` in its reducer must also find and change that novel in the `allCurrentNovels` array, because the novel is also in there, and how it appears there influences how it looks in the app somewhere else. 

Here are a couple of excerpts from switch statements in each reducer: 

```
// /reducers/currentNovel.js
...
    case 'UPDATE_NOVEL':
      return action.novel
    case 'ADD_BADGE':
      return {
        ...state,
        badges: [...state.badges, action.badge]
      }
			...
			
// /reducers/allCurrentNovels.js
... 
    case 'ADD_NOVEL':
      return [...state, action.novel]
    case 'ADD_BADGE':
      return state.map(novel => {
        if (novel.id === action.badge.novel_id) {
          return {
            ...novel,
            badges: [...novel.badges, action.badge]
          };
        } else {
          return novel;
        }
      })
...
```

The `currentNovel` reducer is only responsible for the key of `currentNovel` in the Redux store, which is either null or an object, so it only has to return the payload of the action dispatched to it. The `allCurrentNovels` reducer is responsible for an array of novels, so adding a `currentNovel` needs to affect it too--but because it needs to hold all of the other novels too, the new state it returns spreads the old state into a new array which _also_ includes the new novel object. 

Similarly, with adding a badge to the `currentNovel`, the `currentNovel` reducer only has to return an object that spreads the rest of the pre-existing state into it, and then spreads the pre-existing badges state into the novel's badges array while also adding the new badge. The `allCurrentNovels` reducer needs to deal with this action in a similar way, but it needs to find the right novel before modifying it. 

While my project set up might be a bit of a fringe case for this, how an action might need to go through multiple reducers is a good thing to keep in mind for a variety of different scenarios. Another one from my project that comes to mind is that when an action with the type `"CLEAR_CURRENT_USER"` fires, it needs to be in the `currentUser` reducer to set the `currentUser` object to `null`, but also in the `currentNovel` reducer to set that object to null--and in whatever other reducers deal with pieces of the store that should be influenced by a user logging out. 

I hope this is helpful to anyone else struggling to get their head around multiple reducers like I was! 

