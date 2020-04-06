## Why TypePoint?

If you're developing a client & server app both in TypeScript, you can leverage TypePoint to **_DRYly_** define your RESTful API endpoints to have strongly typed requests and responses, without duplicating or generating code.

## Features

- Strongly typed requests and responses, both on client and server!
- Endpoints are defined only once
- No code generation!
- Validation of request params and body
- Promise based request handlers & middleware
- Support for React Hooks

## Usage

### Install

First we need to install the following packages:

```shell
npm add @typepoint/client @typepoint/server @typepoint/shared @typepoint/express
```

### Endpoint Definitions

We start by defining an endpoint in a shared folder. Our example will define an endpoint to return a `Todo` with a given `id` passed as a path param.

<p class="filename" data-filename="./shared/endpoints/getTodoEndpoint.ts"></p>

```typescript
import { defineEndpoint, Empty } from '@typepoint/shared';

export interface Todo {
  id: string;
  title: string;
  isCompleted: boolean;
}

export interface GetTodoRequestParams {
  id: string;
}

export const getTodoEndpoint = defineEndpoint<GetTodoRequestParams, Empty, Todo[]>((path) =>
  path.literal('/api/todos/').param('id'),
);
```

Our `getTodoEndpoint` variable defines our endpoint, including:

- The HTTP method (defaults to `'GET'`)
- The path to the endpoint: `'/api/todos/{id}'`
- The request params type: `GetTodoRequestParams`
- The request body type: `Empty` (no body, because its a GET method)
- The response body: `Todo`

### Server

Next, let's create a handler for our endpoint in our server.

<p class="filename" data-filename="./server/todos/getTodoHandler.ts"></p>

```typescript
import { createHandler } from '@typepoint/server';
import { getTodoEndpoint } from '../../shared/endpoints/getTodoHandler';
import { getTodoById } from './todoService';

export const getTodoHandler = createHandler(getTodoEndpoint, async ({ request, response }) => {
  // Get todo from our async todo service and put it in our response body
  response.body = await getTodoById(request.params.id);
});
```

Next we need to create a router in order to route requests to our handlers.

<p class="filename" data-filename="./server/router.ts"></p>

```typescript
import { Router } from '@typepoint/server';
import { getTodoHandler } from './todos/getTodoHandler';

export const router = new Router({
  // We just have one handler for now
  handlers: [getTodoHandler],
});
```

Finally, we need to connect our router to our web server.

<p class="filename" data-filename="./server/index.ts"></p>

```typescript
import express = require('express');
import * as bodyParser from 'body-parser';

import { toMiddleware } from '@typepoint/express';
import { router } from './router';

async function run() {
  const app = express();
  const port = 3001;

  app.use(bodyParser.json());

  const handlerMiddleware = toMiddleware(router);
  app.use(handlerMiddleware);

  app.listen(port, () => {
    console.log(`API Server running at http://localhost:${port}`);
  });
}

run().catch(console.error);
```

Now if we run `ts-node ./server` we'll have a working server that will handle getting a todo.

### Client

On the front-end, we need to create a `Client` in order to make requests.

<p class="filename" data-filename="./client/typepoint.ts"></p>

```typescript
import { TypePointClient } from '@typepoint/client';

export const client = new Client({
  server: 'http://localhost:3001',
});
```

Now we have our `TypePointClient` created, we can use it anywhere in our front-end.

<p class="filename" data-filename="./client/index.ts"></p>

```typescript
import { client } from './typepoint';
import { getTodoEndpoint } from '../shared/endpoints/getTodoEndpoint';

async function showTodo() {
  const response = await client.fetch(getTodoEndpoint, {
    params: {
      id: '1',
    },
  });

  const todo = response.body;

  alert(todo.title);
}
```

The above client side code is completely typed. Trying to fetch from the getTodoEndpoint without passing the a string id as a param will cause a design/compile-time error. The response is also completely typed.

## React Hooks

The `@typepoint/react` library provides react hooks to call your endpoints.

### Install

```shell
npm add @typepoint/react
```

### TypePoint Provider

Before you can start using TypePoint React hooks, you'll need to wrap your root component with a `TypePointProvider`. This provides the `client` that the hooks will use.

<p class="filename" data-filename="./index.tsx"></p>

```tsx
import * as React from 'react';
import * as ReactDOM from 'react-dom';
import { TypePointClient } from '@typepoint/client';
import { TypePointProvider } from '@typepoint/react';
import { App } from './app';

const client = new TypePointClient({
  // Path to API server, normally this will be an environment variable
  server: 'http://localhost:3001',
});

export const AppWithProviders = React.memo(() => (
  <TypePointProvider client={client}>
    <App />
  </TypePointProvider>
));

const root = window.document.getElementById('root');

ReactDOM.render(<AppWithProviders />, root);
```

### useEndpoint hook

`useEndpoint` is a hook that immediately fetches a given endpoint with the given `params` and or `body`.

<p class="filename" data-filename="./app.tsx"></p>

```tsx
import React from 'react';
import { useEndpoint } from '@typepoint/react';
import { getTodosEndpoint } from '../shared/endpoints/getTodosEndpoint';

const TodoApp = () => {
  const { response } = useEndpoint(getTodosEndpoint, {});
  const todos = response?.body ?? [];

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  );
};

export default TodoApp;
```

The endpoint will not be fetched unnecessarily, only when the `params` change. In the above example no params are required so it will only be fetched once.

`useEndpoint` also returns a `refetch` function which can be called imperatively to refetch the endpoint with the last used params and body. This is useful for refreshing data.

```typescript
const { response, refetch } = useEndpoint(getTodosEndpoint, {});
```

`useEndpoint` also returns the loading state of the request, as well as an `error` if there is one.

```tsx
const TodoApp = () => {
  const { response } = useEndpoint(getTodosEndpoint, {});
  const { response, loading, error } = useEndpoint(getTodosEndpoint, {});

  if (loading) {
    return <Spinner />;
  }

  if (error) {
    return <div>Something went wrong!</div>;
  }

  const todos = response?.body ?? [];

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  );
};
```

### useEndpointLazily hook

`useEndpointLazily` is just like `useEndpoint` except that it does not immediately fetch the endpoint, instead it provides a `fetch` function to let you fetch the given endpoint on demand.

<p class="filename" data-filename="./newTodoInput.tsx"></p>

```tsx
import React from 'react';
import { useEndpointLazily } from '@typepoint/react';
import { addTodoEndpoint } from '../shared/endpoints/addTodoEndpoint';

/**
 * Component which can create a new todo.
 */
const NewTodoInput = () => {
  const [title, setTitle] = useState('');

  const { fetch: addTodo } = useEndpointLazily(addTodoEndpoint);

  const addTodoWithTitle = useCallback((title: string) => {
    addTodo({ body: { title } });
  }, [addTodo]);

  return (
    <div>
      <input type="text" value=>
      <button type="button" onClick={addTodoWithTitle}>Add</button>
    </div>
  );
};

export default TodoApp;
```

## Examples

Check out the [examples repository](https://github.com/typepoint/examples) for example apps that use TypePoint.

- [Todo app using Express & React](https://github.com/typepoint/examples/tree/master/todos-react-hooks-express)
- [Todo app using Next.js & React](https://github.com/typepoint/examples/tree/master/todos-nextjs)

## Contributing

Got a problem or suggestion? Submit an [issue](https://github.com/typepoint/typepoint/issues)!

Want to contribute? Fork the [repository](https://github.com/typepoint/typepoint) and submit a pull request! 🌩
