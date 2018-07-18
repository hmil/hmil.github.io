## Implement test app with 3 - 4 major front end frameworks

### Things to test:

- If possible, create a component test to match all implementations against a spec


User features:
- Result page with paging and reload-as-you-type search
- User session: Login without page reload updates content
- Post content with markdown transformation, no page reload
- Interface with restyped/axios
- server-side rendering

Dev features:
- Code splitting
- TypeScript integration
- Unit testing

Dev setup:
- Live reload capabilities
- Ease of debugging


Practical setup:
- An API server with restyped/express or something like that
- A front-end server (maybe as a loadable module from the API server) to test SSR and serve the actual front
- A front end app
