# Types-4-Strapi

Typescript interface generator for Strapi 4 models.

## Install locally

```bash
npm i --save-dev types-4-strapi
```

Add t4s to your scripts:

```json
"scripts": {
  "develop": "strapi develop",
  "start": "strapi start",
  "build": "strapi build",
  "strapi": "strapi",
  "t4s": "t4s"
}
```

Then run with:

```bash
npm run t4s
```

## Install globally

```bash
npm i -g types-4-strapi
```

Then run with:

```bash
t4s
```

## Attributes

For some inscrutable reason, Strapi 4 returns objects where all the properties (aside from `id`) are wrapped into an `attributes` object. The resulting interfaces will look like this:

```
{
  id: number;
  attributes: {
    username: string;
    email: string;
    provider: string;
    confirmed: boolean;
    blocked: boolean;
    createdAt: Date;
    updatedAt: Date;
  }
}
```

However, for some even more inscrutable reason, sometimes the same object is returned "flattened", without an `attributes` object. This is the case, for instance, for the `/api/users` endpoint, which returns an array of Users with the following structure:

```
{
  id: number;
  username: string;
  email: string;
  provider: string;
  confirmed: boolean;
  blocked: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

The same "flat" structure is also required when submitting the body of `POST` and `PUT` requests. Here is an example using `fetch`.

```ts
// correct
await fetch('https://project.com/api/users', {
  method: 'POST',
  body: JSON.stringify({
    username: 'Jon Snow',
    email: 'jon.snow@housestark.com',
  }),
});

// incorrect
await fetch('https://project.com/api/users', {
  method: 'POST',
  body: JSON.stringify({
    attributes: {
      username: 'Jon Snow',
      email: 'jon.snow@housestark.com',
    },
  }),
});
```

In these cases, rather than creating completely new types, we recommend that you simply 'extract' the type of the `attribute` object from the entity's interface using **indexed access types**.

```ts
type UserAttributes = User['attributes'];

await fetch('https://project.com/api/users', {
  method: 'POST',
  body: {
    username: 'Jon Snow',
    email: 'jon.snow@housestark.co.uk',
  } as UserAttributes
});
```

If you are using [strapi-plugin-transformer](https://market.strapi.io/plugins/strapi-plugin-transformer) to remove the `attributes` key from all responses, use the following generic transformation type to be able to utilise the interfaces generated by `types-4-strapi`:

```ts
type Transformed<A extends { attributes: any }> = A['attributes'] & {
  id: number;
};
```

Usage:

```ts
const response = await fetch('https://project.com/api/users');

const json = await response.json();

const users = json.data as Transformed<User>[];
```

# Even Better:
Instead of the `Transformed` type, in concert with the `strapi-plugin-transformer` plugin, you can tell us to omit `attributes` and `data` structures from the output:
```
npm run t4s -- --noattributes --nodata
```
or
```
t4s --noattributes --nodata
```

An output directory option can also be specified to output the types to a different directory:
```
npm run t4s -- --output <path>
```

To install using this github repo:
```
npm install --save-dev git+https://github.com/netfiredotnet/types-4-strapi.git
```

# TODO
* Generate User type based on actual schema (possible?)
* Modify Payload types:
    ```
    interface Meta {
      pagination?: {
        page: number;
        pageSize: number;
        pageCount: number;
        total: number;
      }
    }

    export interface FindOneRequestPayload<T> {
      data: T;
      meta: Meta;
    }
    export interface FindManyRequestPayload<T> {
      data: Array<T>;
      meta: Meta;
    }
    ```
* Support type-only imports
* Support self-references:
    ```
    export interface SalesOrder {
      id: number;
      tenant?: Tenant;
      lines: LineItem[];
      number?: number;
      date?: Date;
      vendor_number?: string;
      po_number?: string;
      is_template?: boolean;
      type?: 'estimate' | 'invoice';
      estimate?: SalesOrder;
      invoice?: SalesOrder;
      message?: string;
    }
    ```
    to this:
    ```
    export interface SalesOrder {
      id: number;
      tenant?: Tenant;
      lines: LineItem[];
      number?: number;
      date?: Date;
      vendor_number?: string;
      po_number?: string;
      is_template?: boolean;
      type?: 'estimate' | 'invoice';
      estimate?: this;
      invoice?: this;
      message?: string;
    }
    ```
* Generate variants of types for requests and responses. Requests differ in that relations are simply `Array<number>`s that refer to other objects, whereas the responses will contain the related object's contents if populate is set to true.