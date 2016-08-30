# redux-fractal
Local component state &amp; actions in Redux.

Provides the means to hold up local component state in redux state,  to dispatch locally scoped actions and to react to global ones.

What Redux fractal offers is a Redux private store for each component with the notable difference that the component state is actually help up in your
app's state atom, so all global and components ui state live together.

The unique and powerful approach consists in the fact that it allows you to
use the built-in Redux createStore, combineReducers and others for defining the shape and managing the UI ,state with great benefits:
1. All state ( either application state or UI state) is help up in your app single state atom
2. Component state updates are managed via reducers just like your global app state which means predictability and easy testability
3. Reducers used for local state aren't aware that they are used for managing  state for a component instance. As such they can be easily re-used across components or even used for managing global state.
4. You can have per component middleware. This opens up interesting possibilities like locally scoped sagas(eg redux-saga), intercepting and handling of all actions generated by a component before they reach the component's UI store.
5. By default a component intercepts in reducer only actions generated by itself but it;s easy ato enable intercepting global actions or actions generated by other components.

It's easy to get started using redux-fractal

1. Add the local reducer to the redux store under the key 'local'
```js
    import { localReducer } from 'redux-fractal';
    const store = createStore(combineReducers({
        local: localReducer,
        myotherReducer: myotherReducer
    }))
```
2. Decorate the components that hold ui state( transient state, scoped to that very specific component ) with the 'local' higher order component and provide a mandatory, globally unique key for your component.
The key can be generated based on props or a static string but it must be unique and be stable between re-renders. Basically it should follow exactly
the same rules as the React component 'key' but be globally unique.
```js
    import { local } from 'redux-fractal';
    import { createStore } from 'redux';

    const CompToRender = local({
        key: 'myDumbComp',
        createStore: (props) => {
            return createStore(rootReducer, { filter: true, sort: props.sortOrder })
        },
        mapDispatchToProps:(dispatch) => ({
            onFilter: (filter) => dispatch({ type: 'SET_FILTER', payload: filter  }),
            onSort: (sort) => dispatch({ type: 'SET_SORT', payload: sort }),
        })
    })(Table);
```
3. Define a root reducer that will intercept own dispatched functions( by default, but the 'local' higher order component can be easily configured to  intercept actions from global redux instance or generated by other components):
```js
const rootReducer = (state = { filter: null, sort: null, trigger: '', current: '' }, action) => {
     switch(action.type) {
         case 'SET_FILTER':
            return Object.assign({}, state, { filter: action.payload });
         case 'SET_SORT':
            return Object.assign({}, state,
                { sort: action.payload });
         case 'GLOBAL_ACTION':
            return Object.assign({}, state, { filter: 'globalFilter' });
        case 'RESET_DEFAULT':
           return Object.assign({}, state, { sort: state.sort+'_globalSort' });
         default:
            return state;
     }
};
```
Note that the reducer is like any other ordinary reducer used in a redux app. The difference is  that it manages and controls the state transitions for a certain component state.

In fact, you can use whatever method of combining reducers you use for your app with no exceptions, also for the individual components:
```js
import { combineReducers, createStore } from 'redux';
const rootReducer = combineReducers({
    filter: filterReducer,
    sort: sortReducer
});
local({
    id: (props) => props.tableID,
    createStore: (props) => {
        return createStore(rootReducer, componentInitialState);
    }
})
```

The well know 'mapStateToProps' and 'mapDispatchToProps' familiar from react-redux 'connect' are available having the very same signatures.
In fact , internally, redux-fractal uses the connect function from 'react-redux' to connect the component to it's private store.

The difference is, that you get only the component's state in mapStateToProps as opposed to the entire app state and the 'dispatch'
function in mapDispatchToProps dispatches an action tagged with the component key as specified in the HOC config.
These actions can be caught in any global reducer and by default only in the originating component reducer.
One can inspect the originating component by looking at action.meta.triggerComponentKey to get the component's key that dispatched the action.

```js
import { combineReducers, createStore } from 'redux';
const rootReducer = combineReducers({
    filter: filterReducer,
    sort: sortReducer
});
local({
    id: "mygreattable",
    createStore: (props) => {
        return createStore(rootReducer, componentInitialState);
    },
    mapStateToProps: (componentState, ownProps) => ({
        filter:  getTableFilter(componentState)
    }),
    mapDispatchToProps: (localDispatch) =>({
        onFilter: (term) => localDispatch(updateSearchTerm(term))
    })
})
```

By default your component will not react to when other components dispatch
local actions or when something is being dispatched in the global store.
You can change that using 'filterGlobalActions':

```js
local({
    id:  (props) => props.itemID,
    filterGlobalActions: (action) => {
        // Any logic to determine if the actions should be forwarded
        // to the component's reducer. By default none is except those
        // originated by component itself
        const allowedActions = ['RESET_FILTERS', 'CLEAR_SORTING'];
        return allowedActions.indexOf(action.type) !== -1;
    }
})
```
Now any RESET_FILTERS or CLEAR_SORTING global actions or originated by other components will be allowed.
You have lots of flexibility with this method to react when a component updates it's UI state.
Crazy example: when the sorting from one component changes all dropdowns from another component should close:
 ```js
 local({
     id:  'dropdownsContainer',
     createStore: (props) => {
        return createStore((state = {isClosed: false}, action) => {
            switch(action.type) {
                case SET_SORT:
                    return Object.assign({}, state, { isClosed: true });
                break;
                default:
                    return state;
            }
        });
     },
     filterGlobalActions: (action) => {
         // This component is interested in updating it's state
         // when things happen in the 'mygreattable' component, in this
         // case when sorting changes
         const allowedActions = ['SET_SORT'];
         return allowedActions.indexOf(action.type) !== -1 && actions.meta.triggerComponentKey === 'mygreattable';
     }
 })
 ```

TODO
1. Write additional tests
2. Verify behaviour with custom middleware
