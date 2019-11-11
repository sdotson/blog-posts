# Clean Architecture blog post draft

This is a cautionary tale, but also a tale full of redemption, one that will re-invigorate your faith in software engineers and their ability to solve complex problems. I will describe the factors that led us down the road to perdition, the twinkle of inspiration that led us to adopting the clean architecture pattern, and finally some of our reflections looking back on the experience during a recent retrospective.

## Initial motivations
Early on we decided to use [SuperTest](https://github.com/visionmedia/supertest), [nock](https://github.com/nock/nock), and fixture files for the integration tests in our various microservices. SuperTest is a library that facilitates the testing of Node HTTP servers. Nock is a way of intercepting various HTTP requests and returning mock data, represented by our fixture files. We felt good about testing various scenarios for all of our endpoints.

Those were simpler times. Our product was simpler. Our concerns were simpler. The sun even seemed simpler back then. Although if we looked a little closer at that great fiery orb in the sky, we'd see hidden complexity at unimaginable scale, including a solar dynamo generating a magnetic field, solar flares leaping majestically into the cosmos, and hydrogen fusion generating the heat and light that sustains us all.

Intoxicated by our own optimism, we gleefullly added tests and fixtures, and bowed before the golden idol of our own creation. Off in the distance, old man fate was already descending the mountain, and some time later, would catch us in the midst of our idolatrous revelry. The moment of reckoning awaited us all.

The complexity of our microservices grew, and eventually, some of the endpoints were orchestrating a dozen or more requests. Each possible request and response needed a fixture file. Every new scenario needed a new fixture file. New version of the endpoint? Most of the time this meant a new fixture file. Sometimes the fixture files needed fixture files. These fixture files slowly grew out of date, as our services continued to evolve, much like old yearbook photos look less and less like their subjects as time marches on.

Soon we were drowning in fixture files. Simple one line changes in domain logic would lead to hours of tedious fixture updates or creating new fixtures altogether. In an effort to save time, sometimes fixtures would be re-used across different versions of the endpoints, creating a complicated web of dependencies. On one particular occasion the very [twinkle from my eye was snuffed out](https://medium.com/upside-team-blog/chilling-thrilling-stories-of-the-upside-engineer-1dbfdcc0228).

We tried to mitigate this with a handy helper we called a golden files loader. If we were confident about the logic, we could flip a config var to update many of the fixture files automatically, but this was only a bandaid for a situation that had gotten out of hand.

We realized that the current situation encouraged shortcuts to thwart the tests, discouraged the creation of new tests, and imperiled our confidence in our services. Something had to be done.

## Inspiration
Many of us actively researched some way out of this problem. Many books were read, but the most influential was Clean Architecture by Robert Martin (Uncle Bob).  His other books, including Clean Code and Clean Coder are also very influential. 

Clean Architecture advocates for a pattern inspired by hexagonal architecture, which was eventually renamed the "ports and adaptors" pattern. The hexagonal architecture was initially proposed by [Alistair Cockburn in 2005](https://alistair.cockburn.us/hexagonal-architecture/). Other sources of inspiration include the [onion pattern](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/) and [DCI](https://www.artima.com/articles/dci_vision.html).

Martin's pattern is guided by the principle that architecture should be indepedendant from the UI, database, frameworks, and any other external agency. We acheive this by defining clear boundaries between components and separating stable business policy from technical details.

In addition, he proposed six principles for components:
- **Reuse/release equivalence** - Components reused together should be released together.
- **Common closure** - A component level restatement of the single responsibility principle. "Gather together things that change at the same time for the same reasons. Separate those things that change at different times or for differnt reasons.", in the words of Robert Martin.
- **Common reuse** - No unneccesary dependencies.
- **Acyclic dependencies** - Avoid cycles by using the dependency inversion principle or creating a new component with shared functionality that other components depend upon.
- **Stable dependencies** - Less stable components should depend on more stable components.
- **Stable abstractions** - Stable components should be abstract and and less table components whould be more concrete.

More concretely in terms of organization, clean architecture advocates for domain logic, interface adaptors, interfaces, entities, and use cases.

## Our implementation
We decided to create a layered architecture organized by function. These functions are domain, adaptors, glue, entities, and interfaces. To make it clear what interfaces are implemented and to express our entities more elegantly we also started using typescript in our microservices.

### Folder structure
- domain
  - interfaces
  - entities
- adaptors
- routers
- glue

### Entities
The entities are respresentations of the different data structures used in the system. 

```
export interface Talent {
  name: string;
  skillLevel: number;
}
```

### Interfaces
The interfaces describe the various inputs and outputs expected by the domain functions, including the higher order domain functions. We could have used typescript interfaces but instead opted for types, which we thought were simpler to use and understand.

```
export type FunMaker = (
  relationships: Relationship[],
  environment: Environment,
  money?: number
) => Fun;
```

### Domain
Domain functions contain the business logic, of which we had two types: higher order functions and simple functions.

#### Simple functions
The simple functions are functions, usually pure, that take an input and produce an output.

```
const calculateTotal = ({ shipping: number, taxes: number, total: number }) => {
  return shipping + taxes + total;
};
```

#### Higher order functions
For the functions that include orchestration logic implementing different adaptors, we use higher order functions that accept all the necessary dependencies and return a function that implement those dependencies.

```
const buildHappinessGenerator = (
 makeMoney: MoneyMaker,
 makeRelationships: RelationshipMaker,
 makeFun: FunMaker,
 determinePurpose: PurposeDeterminer,
 rageAgainstNihilism: NihilismRager,
 seekHelp: HelpSeeker
) => {
  return (
    values: Value[], 
    talents: Talent[],
    weaknesses: Weakness[],
    environment: Environment
  ): Happiness => {
    try {
      const purpose = determinePurpose(values, talents, weaknesses, environment);
      const relationships = makeRelationships(values, environment);
      const money = makeMoney(talents, environment, purpose);
      const fun = makeFun(relationships, environment, money);

      return rageAgainstNihilism(purpose, relationships, money, fun);
    } catch (err) {
      seekHelp(relationships);
    }
  };
}
```

These are all expected to have tests and this is where the real power is revealed. Whereas in the past we’d have to use nock to intercept HTTP requests and create a whole bunch of fixture files, now we could create mock functions representing the injected dependencies, focusing our testing efforts on the logic and orchestration.

### Adaptors
Adaptors are functions that encapsulate I/O, whether that is reading/writing from a data store, calling an API, shooting a firework into the air, or whatever silly example you can think of. These functions are meant to be very simple, with no business logic, and as a result, don’t need tests and should be excluded from code coverage.

A edge case scenario we encountered was when we wanted to transform the data returned from an adaptor. Although this is business logic, we decided to group it with the adpator logic because it was tightly coupled to that concept. We tested adaptor transform functions and they were included in code coverage.

### Glue
Glue “glues” our different injected dependencies together to create a specific implementation strategy with a domain higher order function.

Glue is where we use our factory functions to inject our dependencies and build the specific implementation strategies we want to use. The dependencies can be adaptors or domain functions. The thought is to isolate and decouple concerns so that the business logic does not have to care about the specific implementation details, but rather, the interface for that implementation.

### Routers
The routers folder just includes the routes as well as controllers and request validation. These should all be very thin.

### Things we’ve learned

At Upside we strive to learn and grow everyday, which means we frequntly get together for retrospectives to talk about how things went. The passion channeled into these discussions frequently resemble Thanksgiving dinners, except with a strong undercurrent of mutual respect and nobody's in tears at the end.

Here are a few takeways:

#### Good
- Tests are less brittle.
- Interfaces and entities explicitly describing what things do and what they are.
- Easy to add tests and alternative implementations

#### Things we can improve
- **Might not be the solution for everything** - Several folks mentioned that clean architecture might be overkill or even unnatural for smaller services, GraphQL services, or services written in other languages where another pattern is more idiomatic. Further, we should weigh its advantages against other patterns on a case by case basis.
- **Be more thoughtful about dependency injection** - One particularly powerful point raised was that we might not need the flexibility afforded by the higher order functions in the glue file. If there is only one implementation of a function, do we need the additional complexity added by having a higher order function to create it? Further, some argued that the use of dependency injection implies that a function is interfacing directly with something in the outer ring of the dependency circle, i.e., a database connection, HTTP request, or some other side effect.

### Links for more information
- https://www.freecodecamp.org/news/a-quick-introduction-to-clean-architecture-990c014448d2/
