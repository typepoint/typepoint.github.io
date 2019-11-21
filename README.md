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
npm add @typepoint/shared
npm add @typepoint/server
npm add @typepoint/client
```

### Endpoint Definitions

We start by defining an endpoint in a shared folder. Our example will define an endpoint to return a `Todo` with a given `id` passed as a path param.

<p class="filename" data-filename="./shared/endpoints/getTodoEndpoint.ts"></p>

```typescript
import { defineEndpoint } from '@typepoint/shared';

export interface Todo {
  id: string;
  title: string;
  isCompleted: boolean;
}

export interface GetTodoRequestParams {
  id: string;
}

export const getTodoEndpoint = defineEndpoint<GetTodoRequestParams, Empty, Todo[]>(path =>
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
import { defineHandler } from '@typepoint/server';
import { getTodoEndpoint } from '../../shared/endpoints/getTodoHandler';
import { getTodoById } from './todoService';

export const getTodoHandler = defineHandler(getTodoEndpoint, async ({ request, response }) => {
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

const todo = await client.fetch(getTodoEndpoint {
  params: {
    id: '1'
  }
});

alert(todo.title);
```

## Examples

Check out the [examples repository](https://github.com/typepoint/examples) for examples on using TypePoint.

## Contributing

Got an problem or suggestion? Submit an [issue](https://github.com/typepoint/typepoint/issues)!

Want to contribute? Fork the [repository](https://github.com/typepoint/typepoint) and submit a pull request! 😸
