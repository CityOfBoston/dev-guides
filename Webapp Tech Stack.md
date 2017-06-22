# Webapp Tech Stack

### Introduction

This document describes the frameworks and tools that we chose to develop
webapps for the City of Boston.

### Principles

We tried to adhere to the following principles when choosing technologies.

**Good and easy beats perfect and hard.** It’s much better to have a solution
that’s 85% what we want but is maintained by someone else than to build a 100%
solution that we have to maintain ourselves.

**Bias towards familiarity.** We want to use technologies and techniques that
are likely familiar to interns, contractors, and casual frontend developers.

**Allow for consistency.** Our development model is to have a portfolio of many
different apps, rather than a few monolithic ones. Therefore we want tech that
we can apply consistently across many different apps so that moving among them
will be straightforward.

**Support infrequent maintenance.** While we want to adhere to practices of
iterative development, the nature of having several municipal apps means that
many will not be actively developed at any one time. We want tools that support
coming back to work on an app after a long time, when the developer has forgotten
many of the details.

**Take away unimportant decisions.** A lot of development effort can be wasted
on architectural bikeshedding, particularly around how to structure CSS or REST
APIs or similar. We want solutions that sidestep having to make those choices.


## Technologies, Libraries, and Frameworks

#### [React](https://facebook.github.io/react/)

For a view library, we wanted something that supported declarative UIs, as that
is far-and-away the most pleasant way to write frontend apps. We chose React
because it’s a very commonly-used frontend library and has a large community
that supports an ecosystem of components that we can integrate.

We recently used Vue for the public notices project, and it was in the running,
but the ecosystem and familiarity arguments for React tipped the scales in its
favor. Preact was a tempting alternative that could leverage some React
components, but we worried that debugging subtle behavior differences between it
and React would be a time sink later on.

Ember is too much of a whole-app framework that we worried about overall
flexibility in making our own architectural choices.

#### [GraphQL](http://graphql.org/)

For new APIs, GraphQL is a much more plesant choice than a REST architecture. A
client-driven API structure is more maintainable and scales well to multiple
clients (such as a native app). This model also works well for connecting to the
variety of data sources within the government while presenting a unified
interface to the client app.

#### [Node](https://nodejs.org/)

Given our choices of React and GraphQL, Node was an obvious one for the server.
This would let us do the server-side rendering that is good both for perceived
speed and searchability.

Ruby on Rails was briefly considered, but due to our intention to build the UI
in React and run GraphQL for the API it didn’t have anything to recommend it.

#### [Next](https://github.com/zeit/next.js/)

Choosing Next was a prime example of “good and easy beats perfect and hard.”
Next is a works-out-of-the-box framework for running React apps on the client
and server. Its built-in Webpack configuration that automatically handles code
splitting across pages and hot reloading in development was a huge reason we
chose this rather than rolling our own.

While Next’s strategy for hydrating the browser client app’s data is not as
sophisticated as other approaches (such as Apollo’s, or anything we might build
ourselves), it works and is straightforward to understand. Next definitely feels
like something we *could* cobble together ourselves out of Webpack, React
Router, and some SSR glue code (minus some details like JS preloading support),
but that effort and maintenance is completely not worth it just to get something
a little more tuned to our needs and tastes.

Rakt is on our radar, but is not necessarily a good fit given we’re already
splitting data access out to our GraphQL API. It’s also pre-alpha right now.

#### [styled-jsx](https://github.com/zeit/styled-jsx/)

The bulk of our styling comes from Crispus, our common CSS library for
Boston.gov. Having a common, external library gives us good consitency across
apps at the smallish cost of downloading library CSS that is not used on the
current page or app.

To augment Crispus, we wanted a CSS-in-JS solution for custom components or tiny
layout needs. CSS-in-JS lets us keep styling in the same file as the relevant
component, leverage JS abstractions, and avoids questions about how to write
well-scoped CSS.

We chose styled-jsx because it’s built in to Next.

The 311 app currently uses Glamor, which we actually prefer to styled-jsx, as
its JS object–based styles are easier to work with than the awkward embedded
`<style>` tags, but since we aim to have Crispus handle the heavy listing of
styling, we decided styled-jsx was good enough and didn’t need any separate
setup.

#### [fetch](https://github.com/matthew-andrews/isomorphic-fetch)

Since we’re using GraphQL, the question of a GraphQL client came up. We chose to
POST queries with fetch because it’s lightweight and integrates well with being
called from Next’s `getInitialProps` on the client and the server, owing to
Hapi’s ability to inject requests on the server side.

While this means that we need to roll our own caching and pagination in some
cases, so far those cases are rare enough that we can write our own simple
solutions that meet the needs of the particular use case.

We considered Relay, but its server-side story is not strong, it has a
relatively large code size, and it ends up intertwined with all of your
components. Using apollo-codegen and Flow we can also still solve the “make sure
components have the data they want” problem that React optimizes for.

Apollo has a lighter-weight feel, but for our apps is still overkill.

#### [Hapi](https://hapijs.com/)

Fin heard someone at a hackathon talking good things about Hapi, so we’re trying
it out. It has some good built-in features, but our server needs are light
enough that the choice here is not significat. Between Next and GraphQL, we have
very few endpoints that a server needs to manage.

Hapi does have the `inject` method, which we use quite effectively to do local,
server-side GraphQL requests during server-side rendering.

Express is the obvious alternative and probably wins out over Hapi on the “bias
towards familiarity” criteria, but because we do so little server-side the
choice hasn’t mattered much.

#### [MobX](https://mobx.js.org/)

For managing shared state across components in an app it’s good to have a tool
to help, as passing props and mutation functions up and down the component
hierarchy can require a substantial amount of discipline to keep straight, and
often requires making changes to intermediate components to pass new values back
and forth.

We chose MobX because it gives us reactivity in a fairly straightforward way.
Its `@computed` feature is a nice way to define values that a frontend needs in
terms of a small state footprint. Its `reaction`/`autorun` capability has also
been good in bridging between things like map events and the queries they
trigger.

Redux is obvious alternative. It has a better server-to-client hydration story
and its use of functions and immutable state is appealing, but its separation of
state transformations, while it makes some testing easier, tends to complicate
tracing how components interact with state. We tried Redux initially for 311 but
switched to MobX because it seemed to make our logic easier to follow, which
should aid future maintenance. For example, a MobX `@computed` method is often
easier to reason about than having a Redux reducer that pre-computes a similar
value.

Nevertheless, we subscribe to only using a state management tool when it’s
really needed to coordinate across components, and try to use `setState` and
props when possible.

## Tools

#### [ES2017](https://babeljs.io/learn-es2015/)

Object spread operators, argument destructuring, arrow functions, and
`async`/`await` make coding in JavaScript light years more plesant than ES5 syntax.

#### [Flow](https://flowtype.org/)

Static type checking is like hundreds of tiny unit tests sprinkled through your
code that your editor can check for automatically. Flow has been a huge help in
eliminating tiny bugs from null dereferences or getting object keys wrong. We
expect it will pay off in spades when we start into “infrequent maintenance” of
our apps.

Writing Flow definitions for third party libraries is also a useful exercise to
learn about those libraries in-depth before adopting them in code.

Flow is also a good tool in applying some checking to the API responses we get
from 3rd party services in our GraphQL endpoint.

Typescript is also a consideration, though our impression was that Flow is
easier to incrementally adopt and lets you stay more “JavaScripty.” We are not
strongly tied to one over the other, compared with not having static type
checking at all.

#### [apollo-codegen](https://github.com/apollographql/apollo-codegen)

This tool automatically generates Flow (and other languages) from a GraphQL
schema and a client’s GraphQL queries. This has been a huge win since it means
that our app code is statically type checked against the data we will be getting
from our GraphQL server. This gives us one of the big wins of Relay, that
components have the data they expect, without any runtime/bundle size cost.

#### [Jest](https://facebook.github.io/jest/)

Our current feeling on browser testing is that trying to use real browsers in an
automated way is flakier than the bugs you’ll catch anyway, which are typically
either visual or obscure. Jest gives us a good enough fake DOM with jsdom, runs
things in parallel which is probably faster, and works out of the box with no
setup.

Additionally, Jest’s snapshot testing is a great approach to solving the problem
of “this component should render” tests that makes it less of a pain to update
the test for intentional changes.

Karma as a test runner does have a “real browser” story, but we haven’t felt the
need for that at this point.

#### [ESLint](http://eslint.org/)

Similar to static type checking, we want to enforce some things about code
around React props, unused values, *&c*. On the basis of not making 
decisions, we go with the default ESLint recommendations.

We also exclusively use errors over warnings. If something is important it
should be fixed, otherwise it can be ignored. ESLint also has good inline
comments for disabiling certain checks when we need them to be.

#### [Prettier](https://github.com/prettier/prettier)

What code style you choose is far less important than having one applied
consistently. Prettier lets you write JavaScript without particuarly caring
about how it looks, knowing that the tool will sort that all out for you. We’ve
tried to go with as vanilla a Prettier config as possible, though we have added
`trailingComma` because it reduces commit diffs and makes in easier to add or
rearrange object literals.

It’s possible to just use ESLint for style enforcement, and we do that on some
older projects, but Prettier does a more in-depth fix that takes into account
line length and such.

More than anything, it’s important to set up style fixes to run automatically on
save. We have plugins to run Prettier through ESLint and then you can `eslint
--fix` via your editor.

#### [Storybook](https://storybook.js.org/)

Storybook is a tool that lets you stand up individual React components in a
browser, outside of the rest of your app. It’s useful for showing off the
various states of components that can be hard to easily reproduce in a running
app (such as error conditions).

Developing components in Storybook can be a kind of TDD for UIs, though it does
impose an overhead when you change component props to update the stories as
well.

#### [GitHub](https://github.com/)

Obvs.

#### [Travis](https://travis-ci.org/)

Our CI needs are basically to run `npm test` and report to GitHub, and Travis
does that well and is free for our open source repositories.

The city already used Travis for other things so we did not investigate other
options, given that Travis meets our needs.

#### [Heroku](https://heroku.com/)

Heroku lets us easily and automatically run staging servers for continuous
deployment, and its push-button release from GitHub is a big win. This is a
perfect example of applying our “good and easy” principle vs. running on EC2 or
in the city’s datacenter.

#### [Codecov](https://codecov.io/)

While we’re not dogmatic about coverage amounts, we find it at least valuable to
visualize the coverage in an app, and Codecov does a great job at that,
integrating into GitHub to deliver coverage change reports on each PR.

#### [Opbeat](https://opbeat.com/)

We needed a way to get frontend / backend exceptions and also record timings
from our GraphQL server to our other backends. Opbeat is free to start with and
supports these things (though we’ll see how this changes with its recent sale to
Elastic).

New Relic, Sentry, and Rollbar were all considered, but either were overkill,
costly, or did not support measuring timing. This choice may change as we move
apps from staging to production and our needs / budget change.

#### [Greenkeeper](https://greenkeeper.io/)

Chasing tiny patch upgrades to libraries can be a risky time sink, especially
across multiple app that are working fine, but Greenkeeper at least keeps us
abreast of changes and will be useful to monitor for any security updates that
we should prioritize.
