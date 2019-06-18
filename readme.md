# package.json Entries Proposal

## Background

Node.js has traditionally used the `"main"` field as a string to indicate the entry point to use for any given package
when imported as a bare specifier (`import 'pkg'`).

When support for browsers was needed the [`"browser"` field](https://github.com/defunctzombie/package-browser-field-spec) was created to support different mains for the browser, allowing universal npm packages to coexist with standard Node.js packages.

With the introduction of ES modules in browsers, the `"module"` field has been adopted to mean `"module"` AND `"browser"`.

In addition, many other environments exist with their own version of the main field in the same way, such as `"react-native"` and `"electron"` to just name a couple.

There has also been talk of introducing new environment targets to allow moving to no longer referencing transpiled code with special names such as a `"syntax"` field supporting `"current"` or `"esmodule"` main entry point targets.

## Problem Statement

The above is just a short summary of some of the fields that have been used, but there are dozens more custom solutions in place.

The issue is that this proliferation of main entry point mechanisms does not scale, and it does not allow them to compose.

For example, it is impossible for Node.js to implement the `"module"` field, since it would break ecosystem packages in Node.js
which assume `"module"` is a browser module field with certain tooling-defined semantics which it does not support. Does that mean Node.js would need to create a special `"nodemodule"` field? The potential combinations like this grow as we add any further differentiation of the main in future.

## Proposal

The proposal is for a new field which captures multiple custom environment conditions under a single object, where the environment
condition names can be extended and host-defined in both runtimes and tools such as bundlers.

```json
{
  "entries": {
    "browser": "./main-browser.js",
    "node": "./main-node.js",
    "default": "./main-misc.js"
  }
}
```

The standard environment condition names implemented would include `"node"` (Node.js), `"browser"` (any browser runtime), `"module"` (any environment or runtime with support for ES modules) and `"default"` which would be true in any environment as a fallback main.

The `"main"` field effectively desugars to `"entries": { "default": "..." }`.

The entry point for a given resolve operation is taken based on the first match from the `"entries"` field, in object property order.

Unknown names are ignored, and names which are associated with the environment are matched as soon as they are seen.

### Composition

In addition to direct environment conditional names, conditional names can also be composed using the `|` operator:

```json
{
  "entries": {
    "module|browser": "./x.browser.module.js",
    "module|node": "./x.node.module.js",
    "default": "./x.cjs.js",
  }
}
```

In the above, we are able to provide different entry points for the ES module version in the browser and Node.js, while falling back to the CommonJS version.

### Extending Environment Entry Names

Hosts can define custom conditional names for themselves and support them. For example, React Native can define `"react-native"` and Electron can define `"electron"` just fine.

If hosts want to register the name centrally, we could allow names to be reserved as well provided they have minimal chances of conflict with existing names in use with other hosts.

## FAQ

### Why not just allow conventions to continue as they are, with different types of main fields?

The problem is that as we get more and more fields, we have no ability to compose combinations or subsets of their conditions.

In addition, support for different fields may be limited between different tools and runtimes. By having a single field that can track
the different types of conditions being detected, we can better ensure semantic-equivalence between resolver implementations as well.

### Does this replace the "module" field?

The `"module"` field could continue to be supported just fine, and could even be desugared into this field as:

```json
{
  "entries": {
    "module|browser": "..."
  }
}
```

since the `"module"` field has pretty much assumed the convention of meaning the modular browser environment.

If we were to specify such a backwards compatibility, this again would help different tools to be able to agree on the semantics at play here.

### Why should this be implemented in Node.js?

It could be possible for this field not to be implemented in Node.js itself, and we could just disallow the `"node"` environment conditional and go along with the other types of conditionals in tools.

But we may well want to add conditionals into Node.js in future that would be useful to support, for example supporting non-transpiled entry points in Node.js would need new custom conditionals directly supported in Node.js, which could be a useful feature to add.

In addition in future with new types of entry points like binary AST support this could be a useful thing to select on as well.

There is no rush to implement this in Node.js though certainly.

### Who will decide on what condition names are accepted as standard?

I have no idea. Let's discuss this!

## License

MIT