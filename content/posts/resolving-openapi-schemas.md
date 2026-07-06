+++
date = '2026-07-06T11:19:04-07:00'
draft = false
title = 'Resolving OpenAPI schemas with TypeScript'
summary = ""
+++

Deep dive into parsing a spec, resolving cross-referenced schemas, guarding against circular references, and turning raw JSON into something a UI can safely render. This post is a full walkthrough of one piece of that pipeline, a TypeScript module that loads Stripe's OpenAPI spec and resolves its schemas into a renderable tree. It goes line by line through the actual code, covering the TypeScript patterns that make this kind of tool possible: derived types, discriminated unions, recursive data structures, and the defensive checks needed when working with data you don't fully control.

This article covers the details of the Stripe OpenAPI Demo for generating API documentation. For the original code, see [https://github.com/kapunahelewong/stripe-openapi-demo/blob/main/lib/spec.ts](https://github.com/kapunahelewong/stripe-openapi-demo/blob/main/lib/spec.ts).


```typescript
let name: string = "Alex";
name = 42; // TypeScript error: Type 'number' is not assignable to type 'string'
```

At runtime, this is just `let name = "Alex"`. The `: string` part gets deleted during compilation.

## Defining the known values: `as const`

```typescript
export const RESOURCES = [
  "customers",
  "payment_intents",
  "charges",
  "invoices",
  "products",
] as const;
```

Normally, writing `const nums = [1, 2, 3]` gives TypeScript the type `number[]`: an array of numbers, could be any numbers, could change later. That's because `const` in JavaScript only prevents reassigning the variable itself. You can still `.push()` onto the array, so TypeScript keeps the type loose.

`as const` overrides that. It tells TypeScript to treat the array as a fixed, readonly tuple of exact values instead. Applied to `RESOURCES`, the type becomes:

```typescript
readonly[("customers", "payment_intents", "charges", "invoices", "products")];
```

`readonly` means there are five exact string slots. Calling `.push()` on this would now be a compile error.

## Deriving a type from a value

```typescript
export type Resource = (typeof RESOURCES)[number];
```

This line does two things.

`typeof RESOURCES` looks like the runtime `typeof` operator from JavaScript (`typeof "hello" === "string"`), but here it's used in a type position, which changes its meaning. It means "give me the TypeScript type of this value," not the runtime string it produces.

`[number]` is called an indexed access type. Given an array or tuple type, adding `[number]` means "give me the type you'd get from indexing into this with any number." For a tuple of literal strings, that produces the union of every value in it:

```typescript
"customers" | "payment_intents" | "charges" | "invoices" | "products";
```

A union type, written with `|`, means a value must be exactly one of the listed options. So `Resource` now only accepts those five strings. Try passing anything else to a function expecting `Resource` and TypeScript will catch the typo before you ever run the code.

The reason to build the type this way, instead of writing the union out by hand, is that it creates a single source of truth. Add a new entry to the `RESOURCES` array later and `Resource` picks it up automatically. There's no second list to remember to update.

## Enforcing complete coverage with `Record`

```typescript
export const RESOURCE_LABELS: Record<Resource, string> = {
  customers: "Customers",
  payment_intents: "Payment Intents",
  charges: "Charges",
  invoices: "Invoices",
  products: "Products",
};
```

`Record<K, V>` is a built-in generic utility type. Think of it as a function, but one that operates on types instead of values: feed it two types and it produces an object type. `Record<K, V>` means "an object where every key is of type `K` and every value is of type `V`."

`Record<Resource, string>` means this object must have an entry for every single value in the `Resource` union, and every value must be a string. Delete the `products` line and TypeScript will flag a missing property. This turns a mistake that would otherwise show up as a blank label in the UI into a compile-time error instead.

## Reading `interface` blocks field by field

```typescript
interface RawOperation {
  operationId: string;
  summary?: string;
  description?: string;
  parameters?: RawParameter[];
  requestBody?: {
    content?: Record<string, { schema?: unknown }>;
  };
  responses?: Record<
    string,
    { description?: string; content?: Record<string, { schema?: unknown }> }
  >;
}
```

An `interface` describes the shape an object must have. Reading this one field by field:

`operationId: string` has no question mark, so it's required.

`summary?: string` has a question mark, which marks the field as optional. Its value is either a string or `undefined`. Not every operation in Stripe's spec includes a summary, so the type honestly reflects that instead of pretending the field always exists.

`requestBody?: { content?: Record<string, { schema?: unknown }> }` is an inline object type. Rather than naming this shape as its own interface, it's written directly inside the field's type position. Reading it from the inside out: `{ schema?: unknown }` is an object that might have a schema field, and we don't yet know its shape, so it's typed `unknown`. `Record<string, { schema?: unknown }>` is a dictionary where the keys aren't known in advance, since they're content types like `"application/json"`, but every value shares that shape. Wrapping that in `{ content?: ... }` makes the whole thing optional too.

This mirrors Stripe's actual JSON structure:

```json
"requestBody": {
  "content": {
    "application/json": { "schema": { } }
  }
}
```

Elsewhere in the file:

```typescript
paths: Record<string, Partial<Record<HttpMethod, RawOperation>>>;
```

`Partial<T>` is another built-in utility type that makes every field in `T` optional. Without it, `Record<HttpMethod, RawOperation>` would require every path to define all five HTTP methods, get, post, put, patch, and delete. But real API path might only support get and post. Wrapping it in `Partial<...>` fixes this: the object may contain any of those five keys, but doesn't have to contain all of them.

## `unknown` versus `any`

You might notice schema fields are typed `unknown`, not `any`. Both mean "could be anything," but they behave very differently. `any` turns off type checking entirely for that value. `unknown` forces you to check or narrow the value before you're allowed to use it. This is why code working with schema data does things like:

```typescript
if (!schema || typeof schema !== "object") {
  return { kind: "primitive", type: "unknown" };
}
```

before touching `schema` as an object. TypeScript won't let you skip that check.

## Modeling a tree with a discriminated union

```typescript
export type SchemaNode =
  | {
      kind: "object";
      description?: string;
      refName?: string;
      properties: { name: string; required: boolean; node: SchemaNode }[];
    }
  | { kind: "array"; description?: string; items: SchemaNode }
  | {
      kind: "union";
      description?: string;
      unionType: "anyOf" | "oneOf";
      variants: SchemaNode[];
    }
  | {
      kind: "primitive";
      description?: string;
      type: string;
      format?: string;
      enumValues?: string[];
    }
  | { kind: "ref-limit"; description?: string; refName: string };
```

This is one of the more useful patterns in TypeScript: a discriminated union. It's five different object shapes joined with `|`, each sharing one field with a distinct literal value, in this case `kind`. That shared field is the discriminant, the thing you check to know which shape you're dealing with.

The payoff shows up wherever this type gets used:

```typescript
if (node.kind === "array") {
  console.log(node.items); // TypeScript knows this exists here
}
```

Inside that `if` block, TypeScript narrows the type automatically. It knows that if `kind` is `"array"`, `node` must be the array variant, so `.items` is safe to access. Trying to access `.properties`, which only exists on the object variant, would be an error inside that same block. One variable, several possible shapes, and the compiler tracks which shape applies.

Also worth noticing: `SchemaNode` appears inside its own definition, in `items: SchemaNode` and `variants: SchemaNode[]`. This is a recursive type. It mirrors the actual data: schemas can nest arbitrarily deep, an object containing an array of objects containing more objects, and the type needs to allow for that.

## Reading a function signature with defaults

```typescript
export function resolveSchema(schema: unknown, refDepth = 0, visited: Set<string> = new Set()): SchemaNode {
```

`schema: unknown` accepts anything, since it comes from parsed JSON with no guaranteed shape.

`refDepth = 0` is a default parameter value. Call `resolveSchema(someSchema)` with just one argument and `refDepth` defaults to `0`. Callers outside this function don't need to think about depth tracking at all. Only the function's own recursive calls pass a specific value in.

`visited: Set<string> = new Set()` follows the same idea, defaulting to a fresh, empty `Set` per top-level call. A `Set` stores unique values with fast lookups, and here it tracks which schema references have already been followed on the current path, which is how the function protects against infinite loops.

The return type `SchemaNode` means the function always produces one of the five shapes from the union above.

## Following a reference without looping forever

Stripe's schemas reference each other by name instead of repeating themselves. A Customer schema might point to a Card schema, which points back to Customer. Here's how that gets handled safely:

```typescript
if (typeof s.$ref === "string") {
  const name = refName(s.$ref);
  if (visited.has(name) || refDepth >= MAX_REF_DEPTH) {
    return { kind: "ref-limit", refName: name };
  }
  const spec = loadSpec();
  const target = spec.components.schemas[name];
  const node = resolveSchema(target, refDepth + 1, new Set(visited).add(name));
  if (node.kind === "object") node.refName = name;
  return node;
}
```

`typeof s.$ref === "string"` checks whether this schema is a reference rather than an inline definition.

`refName(s.$ref)` extracts a clean name like `"Customer"` out of a value like `"#/components/schemas/Customer"`, using a small helper that splits on `/` and takes the last piece.

```typescript
if (visited.has(name) || refDepth >= MAX_REF_DEPTH) {
  return { kind: "ref-limit", refName: name };
}
```

This is the guard that keeps recursion from running forever. `visited.has(name)` asks whether this exact reference has already shown up earlier in this same chain, which is what catches a true cycle like Customer pointing to Card pointing back to Customer. `refDepth >= MAX_REF_DEPTH` caps how many reference hops are followed regardless of cycles, since even a non-circular spec can nest deeply enough to produce an unreasonably large tree. If either condition is true, the function stops and returns a placeholder node instead, so the UI can say "there's more here" without the app hanging or crashing.

```typescript
const node = resolveSchema(target, refDepth + 1, new Set(visited).add(name));
```

This is the recursive call, and the third argument deserves a close look. `new Set(visited)` creates a brand new copy of the current visited set. `.add(name)` then adds the current reference's name to that copy. The reason this matters: the same `visited` set gets reused across sibling branches of the tree, for instance when resolving multiple properties on the same object. If the function mutated the original set directly instead of copying it, resolving one property could incorrectly affect the cycle tracking for a completely unrelated sibling property. Copying keeps each branch's tracking isolated to its own path through the tree.

## Handling unions, arrays, and objects

```typescript
if (Array.isArray(s.anyOf) || Array.isArray(s.oneOf)) {
  const unionType = Array.isArray(s.anyOf) ? "anyOf" : "oneOf";
  const rawVariants = (s.anyOf ?? s.oneOf) as unknown[];
  return {
    kind: "union",
    description: typeof s.description === "string" ? s.description : undefined,
    unionType,
    variants: rawVariants
      .slice(0, MAX_UNION_VARIANTS)
      .map((v) => resolveSchema(v, refDepth, visited)),
  };
}
```

OpenAPI lets a schema say a value could be any of several types (`anyOf`) or exactly one of several types (`oneOf`), each represented as an array of alternative schemas.

`Array.isArray(s.anyOf) ? "anyOf" : "oneOf"` is a ternary expression, shorthand for an if/else that produces a value: if `anyOf` exists, label this `"anyOf"`, otherwise it must be `"oneOf"`.

`(s.anyOf ?? s.oneOf) as unknown[]` uses the nullish coalescing operator, `??`, which means "if the left side is null or undefined, use the right side instead." It's different from `||` because it doesn't trigger on falsy values like `0` or an empty string, only on `null` or `undefined`. Here it picks whichever of the two actually exists.

`rawVariants.slice(0, MAX_UNION_VARIANTS).map((v) => resolveSchema(v, refDepth, visited))` caps the list at 6 variants and resolves each one recursively. Notice `refDepth` isn't incremented here. Moving into a union variant isn't the same as following a reference, so depth only increases in the reference-handling branch above.

Arrays are simpler:

```typescript
if (s.type === "array") {
  return {
    kind: "array",
    description: typeof s.description === "string" ? s.description : undefined,
    items: resolveSchema(s.items, refDepth, visited),
  };
}
```

If the schema is explicitly typed as an array, recursively resolve whatever's inside `s.items`, which describes the type of each element, and wrap the result in an array node.

Objects pull several patterns together at once:

```typescript
if (s.type === "object" || s.properties) {
  const props = (s.properties ?? {}) as Record<string, unknown>;
  const required = new Set((s.required as string[] | undefined) ?? []);
  return {
    kind: "object",
    description: typeof s.description === "string" ? s.description : undefined,
    properties: Object.entries(props)
      .slice(0, MAX_PROPERTIES)
      .map(([name, propSchema]) => ({
        name,
        required: required.has(name),
        node: resolveSchema(propSchema, refDepth, visited),
      })),
  };
}
```

`s.type === "object" || s.properties` treats a schema as an object either when it's explicitly typed that way, or when it simply has a `properties` field, since real-world specs aren't always consistent about declaring the type explicitly.

`required` collects the list of required field names into a `Set`, giving fast lookup later with `.has()`.

`Object.entries(props)` converts the properties object into an array of `[key, value]` pairs, which can then be looped over with array methods. `.slice(0, MAX_PROPERTIES)` caps the count at 30.

```typescript
.map(([name, propSchema]) => ({
```

The parameter here uses array destructuring. Each entry from `Object.entries` is a two-item array, `[name, propSchema]`. Destructuring pulls both values out directly in the parameter list, instead of writing something like `(entry) => { const name = entry[0]; const propSchema = entry[1]; }`.

Inside, `name` is shorthand property syntax, used because the variable name already matches the property name being set. `required: required.has(name)` checks whether this property's name appears in the required set built earlier. `node: resolveSchema(propSchema, refDepth, visited)` recursively resolves this property's own schema, again without incrementing depth, since descending into a property isn't following a reference either.

Finally, if none of the earlier checks matched, the function falls through to a default case for primitive values like strings, numbers, and booleans:

```typescript
return {
  kind: "primitive",
  description: typeof s.description === "string" ? s.description : undefined,
  type: typeof s.type === "string" ? s.type : "any",
  format: typeof s.format === "string" ? s.format : undefined,
  enumValues: Array.isArray(s.enum) ? (s.enum as string[]) : undefined,
};
```

Same defensive pattern as before: check the actual runtime type of each field before trusting it, and fall back to a safe default if the raw JSON doesn't match expectations.

## The patterns worth remembering

A few ideas repeat throughout this file and are worth carrying into your own TypeScript code:

Deriving a type from a value with `as const` plus `(typeof X)[number]` keeps a list of allowed values and its corresponding type in sync automatically, rather than maintaining both by hand.

`Record` and `Partial` are utility types that let you describe object shapes precisely, whether that means requiring every key to be present or allowing all of them to be optional.

Discriminated unions, built around a shared field like `kind`, let TypeScript narrow a value's type automatically based on a single check, which is one of the most common patterns you'll see in real applications.

Optional chaining (`?.`) and nullish coalescing (`??`) exist for the same reason: real data is full of missing or absent fields, and these operators let you handle that without writing defensive `if` checks everywhere.

`unknown` is the safer alternative to `any` when you genuinely don't know a value's shape yet, since it forces you to verify before using it.

Recursive functions that walk untrusted or self-referencing data, like this schema resolver, need explicit guards against infinite loops. Tracking visited state with a copied `Set` per branch, rather than a single mutated set, is a reliable way to keep each recursive path isolated from its siblings.

You'll come across these same patterns when working with OpenAPI specs, especially anywhere you're parsing external data with an uncertain shape and need to build something safe and predictable out of it.
