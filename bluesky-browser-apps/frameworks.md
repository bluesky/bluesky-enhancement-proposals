# Common Frameworks for Browser Based Applications
A wide variety of user interface frameworks are in use across the various facilities that particiapte in the Bluesky environment. These include native desktop frameworks like `Qt` and `napari` and web frameworks like `Flask`, `React`, and `Dash` and even `Jupyter Hub` and the various Jupyter additions like `Jupyter Widgets`, `Voila`, etc.. 

There is a lof momentum for building browser-based applications at facilities. Several known projects exist in the Bluesky world like:
 - [Austrialian Synchrotron's React App for SAXS](https://github.com/AustralianSynchrotron/saxs_beamline_library_react)
 - [Bluesky Web Client](https://github.com/bluesky/bluesky-webclient)
 - [Databroker Client](https://github.com/bluesky/databroker-client)
 - [Tiled Web Front End](https://github.com/bluesky/tiled/tree/main/web-frontend)

Additionally, the MLExchange project has a number of tools built in Dash.

Developing browser-based applications involves making many choices for tools and APIs. While there is no single framework that will cover all use cases, if multiple teams are contributing browser-based applications, it makes sense to come to have community recommendations in order to increase compatibility and code reuse. This design document makes some suggestions for the community to consider.

## Assumptions
For the purpose of explanation, and to attempt to limit the scope of the document, we highlight a few assumptions:
 - A full web applicaiton can involve a lot of layers: user interface code, server code, databases, etc. We only outline the browser-based interface portion of Web Applications. We assume that other projects (e.g. `Bluesky Queue Server`) provide web-friendly services for persistence or communication with devices.
 - Many use cases that a coalition of Bluesky developers might fulfill with web applications involve communication with controls APIs. For these use cases, we see value in Single Page Applications (SPAs), where user interface logic is transplied into javascript run entirely in the web browser process. Other alternatives where the user interface  (HTML, javsacipt, css) is fully or partially constructed in a server applciation present more latency and potential failure modes. It is likely that this would end up being an unnecessary middle process which would turn around and issue requests to existing process like Queue Server HTTP process.

# Recommendations

## Framework: React
We call out React as the browser based framework of choice. React is mature and widely known. One downside of React is that it is old enough to have undergone a number of significant hanges over the years. Below we generally recommend the more modern alternatives available. 

### Component Type: functional
Functional Components over Class Components. Along with this comes the ability to use the various hooks like `useState`, `useEffect`, and `useContext` (which we describe in more detail below).

### Global State: Context API and useContext / useReducer
For small React applications, state management can often be accomplished on a per-component basis. For example, the current entry for this text box is a state, but other components in the app have no access to that, and cross-component communication about state changes must be handled with events.) But for larger applications an app-wide state management mechanism is useful. For example, if different components need to know the identity of a logged-in user, and is useful to have a state manager outside of each component.

For global state, two very popular alternatives include using the [Redux javascript library](https://react-redux.js.org/introduction/getting-started) and the Context API that is now built into React, along with the new `useContext` and `useReducer` hooks. We recommend using the built-in Context API for managing global state. A nice comparison of these techniques can found [here](https://blog.logrocket.com/react-hooks-context-redux-state-management/)

## CSS / Component Library: MUI
Few things will raise as much discussion as this. User interface libraries add a great deal of pleasing user interface sugar to applications. Some libraries like Tailwind add many useful CSS classes. MUI and React Bootstrap add CSS as well as React components. While it's not necessary for the community to settle on a single look and feel, if we are sharing components across projects, not using similar tools will have signicant impact on code reuse.

Whether we know it or not, we are very used to interacting with Bootsrap applications...they are everywhere. MUI (the newer version of the material-ui package) is very popular. Both are great.There is some momentum towards MUI as several Bluesky projects have already started using it.

## Language: Typescript
React components can be built using JavaScript or Typescript. While transpiling from Typescript to JavaScript adds to the tooling burdon, we recommend Typescript for its readability and the general momentum that Typescript has.

## Unit Testing: Jest
Libraries available for Javascript unit testing include Jest, Mocha, Enzyme and Jasmine. Jest appears to the default for React applicatoins.

## Package Management: NPM
Yarn and NPM are two popular package management solutions. We've never used yarn, so, we recommend NPM.


