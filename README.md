[![npm version](https://img.shields.io/npm/v/@itrocks/delete?logo=npm)](https://www.npmjs.org/package/@itrocks/delete)
[![npm downloads](https://img.shields.io/npm/dm/@itrocks/delete)](https://www.npmjs.org/package/@itrocks/delete)
[![GitHub](https://img.shields.io/github/last-commit/itrocks-ts/delete?color=2dba4e&label=commit&logo=github)](https://github.com/itrocks-ts/delete)
[![issues](https://img.shields.io/github/issues/itrocks-ts/delete)](https://github.com/itrocks-ts/delete/issues)
[![discord](https://img.shields.io/discord/1314141024020467782?color=7289da&label=discord&logo=discord&logoColor=white)](https://25.re/ditr)

# delete

Deletion action handling the button, user confirmation, data source deletion, and visual feedback.

*This documentation was written by an artificial intelligence and may contain errors or approximations.
It has not yet been fully reviewed by a human. If anything seems unclear or incomplete,
please feel free to contact the author of this package.*

## Installation

```bash
npm i @itrocks/delete
```

Adding this dependency gives you a ready‑to‑use `Delete` action that fits
into the it.rocks action stack (`@itrocks/action`, `@itrocks/route`,
`@itrocks/action-request`, …).

## Usage

`@itrocks/delete` provides a generic `Delete<T>` action that:

- asks the user for confirmation before deleting (HTML variant),
- deletes one or more objects from the configured data source,
- returns a small HTML page with a notification and an automatic
  redirect back to the listing, or a JSON payload describing the
  deleted objects.

It is typically used together with:

- `@itrocks/route` to expose a conventional `/delete` route,
- `@itrocks/action-request` to create `Request<T>` objects from HTTP
  requests,
- `@itrocks/storage` as the persistence layer,
- `@itrocks/confirm` to handle the confirmation flow,
- `@itrocks/auto-redirect` for post‑deletion navigation.

### Minimal example

```ts
import { Delete }       from '@itrocks/delete'
import { Route }        from '@itrocks/route'
import type { Request } from '@itrocks/action-request'

class User {
	id   = 0
	name = ''
}

// Conventional configuration: a delete action for User
@Route('/users/:id/delete')
export class DeleteUser extends Delete<User> {}

// Somewhere in your HTTP layer
const deleteUser = new DeleteUser()

async function deleteUserHtml (request: Request<User>) {
	// The first call shows a confirmation page.
	// Once confirmed, it deletes the object and shows a success message
	// with an automatic redirect back to the list.
	return deleteUser.html(request)
}
```

`Request<User>` is usually created by
`@itrocks/action-request` from the incoming HTTP request.

### Complete example: HTML + JSON delete endpoints

The same `Delete<T>` action can serve both HTML and JSON use cases.

```ts
import { Delete }              from '@itrocks/delete'
import type { Request }        from '@itrocks/action-request'
import type { FastifyInstance } from 'fastify'

import { User }                from '../domain/user.js'

// A simple delete action for User
class DeleteUser extends Delete<User> {}

export function registerUserDeleteRoutes (fastify: FastifyInstance) {
	const deleteUser = new DeleteUser()

	// HTML endpoint: confirmation + redirecting feedback page
	fastify.get('/users/:id/delete', async (req, reply) => {
		const request  = toActionRequest<User>(req)
		const response = await deleteUser.html(request)
		reply
			.status(response.status)
			.headers(response.headers)
			.type('text/html')
			.send(response.body)
	})

	// JSON endpoint: directly delete and return deleted objects as JSON
	fastify.delete('/api/users/:id', async (req, reply) => {
		const request  = toActionRequest<User>(req)
		const response = await deleteUser.json(request)
		reply
			.status(response.status)
			.headers(response.headers)
			.send(response.body)
	})
}
```

In the HTML flow, the template bundled with the package shows a
notification such as "`John Doe` deleted." and a link that automatically
redirects the user back to the list of objects.

## API

### `class Delete<T extends object = object> extends Action<T>`

`Delete<T>` is a concrete action that removes objects of type `T` from
the configured data source.

From the implementation you can see that it is decorated with:

- `@Need('object')` – the action requires an existing object (or
  objects) to be present in the request.
- `@Route('/delete')` – declares a conventional `/delete` route when
  used with `@itrocks/route`. You can also add your own route decorators
  in your project (for example `/users/:id/delete`).

Typical call sites receive a `Request<T>` (for example built by
`@itrocks/action-request` from an HTTP request) and invoke either the
`html` or `json` method.

#### Type parameter

- `T` – The domain object type you want to delete (for example `User`,
  `Product`, `Order`, …).

#### Methods

##### `html(request: Request<T>): Promise<HtmlResponse>`

HTML‑oriented delete flow:

1. Uses `@itrocks/confirm` to ask the user for a confirmation (if not
   yet confirmed).
2. Once confirmed, loads the objects from the request via
   `request.getObjects()`.
3. Deletes each object with `dataSource().delete(object)`.
4. Returns an HTML response built from the provided `delete.html`
   template, including a success message and a link that uses
   `@itrocks/auto-redirect` to go back to a sensible screen (typically
   the list of objects).

Use this method for classic web flows where the browser follows
redirects and renders HTML.

##### `json(request: Request<T>): Promise<JsonResponse>`

JSON‑oriented delete flow:

1. Loads the objects from the request via `request.getObjects()`.
2. Deletes each object using the configured data source.
3. Returns a JSON response containing the deleted objects.

Use this method from APIs consumed by SPAs, mobile applications, or
other back‑end services that want to manage their own UI after the
deletion.

## Typical use cases

- **Back‑office delete action** – provide a consistent way to delete
  entities (users, products, orders, …) from an admin UI, with
  confirmation and a redirect back to the list.
- **REST/JSON delete endpoint** – expose `DELETE /api/...` routes that
  delete one or more entities and return the deleted records as JSON.
- **Reusable domain‑specific CRUD packs** – compose `Delete<T>` with
  `New<T>`, `List<T>` and `Edit<T>` to publish a complete CRUD module
  for a given business entity.
- **Safety wrapper around destructive operations** – centralise
  confirmation, error handling and messaging for destructive actions
  instead of re‑implementing them in each controller.
