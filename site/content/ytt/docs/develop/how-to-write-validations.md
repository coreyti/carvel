---
title: Writing Validations
---

## Overview

In the same way one can write constraints on columns/fields/documents, so too with `ytt` Data Values Schema one can declare constraints on Data Values.

One might do this for a number of reasons:
- **catch configuration errors early** — help the user of your `ytt` library from wasting time _discovering_ errors in their configuration when they use it... by catching and _reporting_ those errors right away
  - e.g. in Kubernetes, instead of sifting through statuses and logs, troubleshooting a failed deploy, present the user with an error message _before_ the deployment begins.
- **avoid impractical configuration** — guide them away from setting values that won't work, in practice (e.g. too few or too many `replicas:` in a Kubernetes Deployment);
- **make a Data Values "required"** — force the user to supply values for Data Values that you — as the author — can't possibly know (e.g. credentials, connection info to services, etc.).

This guide is an introduction to writing Validations.

_(Looking for a quick start? see the [Validations Cheat Sheet](quick-ref-validations.md))_

## What a Validations Look Like

A validation is an annotation on a Data Value in a schema file:

```yaml
#@data/values-schema
---
dex:
  #@schema/validation min_len=1
  namespace: ""
```

Here:
- the Data Value `dex.namespace` is a string
- to be valid, that string must be set to a value at least one character long.

One can specify multiple rules to refine what a valid value looks like:

```yaml
#@data/values-schema
---
dex:
  #@schema/validation min_len=1, max_len=63
  namespace: ""
```

Here:
- `dex.namespace` must ultimately be at least 1 character _and_ no more than 63 characters in length.

And one can declare validations for _each_ Data Value; `ytt` will ensure _all_ Data Values are valid.

```yaml
#@data/values-schema
---
dex:
  #@schema/validation min_len=1, max_len=63
  namespace: ""
  #@schema/validation min_len=1
  username: ""
```

Here:
- `dex.username` is also a string; and to be valid must be at least one character in length.

Finally, one can write their own custom rules, as well:

```yaml
#@data/values-schema
---
dex:
  #@schema/validation ("not 'default'", lambda v: v != "default"), min_len=1, max_len=63
  namespace: ""
  #@schema/validation min_len=1
  username: ""
```

Here:
- For `dex.namespace` to be valid it must:
  - _not_ be the string "default", _and_
  - be at least one character long, _and_
  - be no more than 63 characters in length.

## How Validations Work

At a high level, validations fit into the flow like so:

1. In schema, the Author declares validations on Data Values (described [above](#what-a-validations-look-like))
2. The Consumer configures data values (described in [How To Use Data Values](how-to-use-data-values.md#configuring-data-values))
3. All of those values are merged into a single set (the first step of the [ytt Pipeline](how-it-works.md#step-1-calculate-data-values))
4. All validations are run on that final Data Values, collecting any violations
5. If there were violations, processing stops and those are reported; \
   ... otherwise, processing continues normally.

For example, given this schema:

```yaml
#@data/values-schema
---
dex:
  #@schema/validation min_len=1, max_len=63
  namespace: ""
  #@schema/validation min_len=1
  username: ""
```

If the Consumer supplies no data values, then the final Data Values are the defaults from the schema.
When the validations are run, instead of rendering templates, `ytt` reports the violations:

```console
$ ytt -f schema.yaml
ytt: Error: One or more data values were invalid:
- "namespace" (schema.yaml:5) requires "length greater or equal to 1"; fail: length of 0 is less than 1 (by schema.yaml:4)
- "username" (schema.yaml:7) requires "length greater or equal to 1"; fail: length of 0 is less than 1 (by schema.yaml:6)
```

And if the Consumer supplies data values:

```yaml
dex:
  #!                  1         2         3         4         5         6
  #!         1234567890123456789012345678901234567890123456789012345678901234
  namespace: the-longest-namespace-you-ever-did-see-in-fact-probably-too-long
  username: alice
```

```console
$ ytt -f schema.yaml --data-values-file values.yaml
ytt: Error: One or more data values were invalid:
- "namespace" (values.yaml:4) requires "length less than or equal to 63"; fail: length of 64 is more than 63 (by schema.yaml:4)
```

## Common Use Cases

There are a variety of ways you can put validations to use:

- ["Required" Data Values](#required-data-values) — to ensure the user supplies their own value
- [Enumerations](#enumerations) — to limit to a finite, specific set.
- [Mutually Exclusive Sections](#mutually-exclusive-sections) — when you want the Consumer to supply complex configuration for exactly one (1) of a number of options.
- [Conditional Validations](#conditional-validations) — to trigger validations only in certain situations.

### "Required" Data Values

Sometimes, there are configuration values that you — as the Author — either can't possibly know (e.g. IP addresses, domain names) or do not want to supply (e.g. passwords, tokens, certificates, other credentials). Instead, you want to force the Consumer to supply these values.

> _The way to mark a Data Value as "required" is by declaring a validation rule that is not satisfied by that Data Value's default._

There are three general strategies:

- ideally, [using natural constraints](#using-natural-constraints)
- otherwise, [using the empty/zero value](#using-the-emptyzero-value)
- if all else fails, [mark as 'nullable' and 'not_null'](#mark-as-nullable-and-not_null)

#### Using Natural Constraints

The most concise (and maintainable) way to make a data value "required" is to lean on the natural constraints on it.

For example, if `port:`, means any "registered" (ports 1024 - 49151) or "dynamic" (49152 - 65535) port, then

```yaml
#@data/values-schema
---
#@schema/validation min=1024
port: 0
```

out of the box, the Consumer receives this message:

```console
$ ytt -f schema.yaml
ytt: Error: One or more data values were invalid:
- "port" (schema.yaml:4) requires "a value greater or equal to 1024"; fail: 0 is less than 1024 (by schema.yaml:3)
```

Where there are not "natural" limits, one might be able to use the zero or empty value...

#### Using the empty/zero value

For strings, an empty value is often not valid. One can specifically require a non-zero length:

```yaml
#@schema/validation min_len=1
username: ""
```

For integers and floating-point values, non-positive numbers are often not valid. One can require a non-negative number

```yaml
#@schema/validation min=1
replicas: 0
```

For array values, note that [the default value is always an empty list](how-to-write-schema.md#setting-a-default-value-for-an-array). One can require that the array not be empty:

```yaml
#@data/values-schema
---
dex:
  oauth2:
    #@schema/validation min_len=1
    responseTypes:
    - ""
```
Here, 
- `dex.oauth2.responseTypes` is an array of strings. 
- by default no response types are configured.
- however, the rule requires that _at least_ one be specified.

#### Mark as 'nullable' and 'not_null'

In some cases, there simply is no invalid value and/or there is no zero value (e.g. maps).

What's left is to specify no value at all (i.e. `null`) and then require a non-null value.

```yaml
#@data/values-schema
---
#@schema/nullable
#@schema/validation not_null=True
tlsCertificate:
  tls.crt: ""
  tls.key: ""
  ca.crt: ""
```
Here:
- `tlsCertificate:` is a map, containing three items.
- `@schema/nullable` changes `tlsCertificate:` in two ways (details at [`@schema/nullable`](lang-ref-ytt-schema.md#schemanullable))
  - now, `tlsCertificate` can be set to `null`
  - and, `tlsCertificate` is `null` by default.
- the `not_null=` rule requires that `tlsCertificate:` _not_ be `null`

out of the box, the Consumer receives this message:

```console
$ ytt -f schema.yaml
ytt: Error: One or more data values were invalid:
- "tlsCertificate" (schema.yaml:5) requires "not null"; fail: value is null (by schema.yaml:4)
```

### Enumerations

Some values must be from a discrete and specific set.

```yaml
#@data/values-schema
---
#@schema/validation one_of=["aws", "azure", "vsphere"]
provider: vsphere
```


### Mutually Exclusive Sections 

### Conditional Validations

#### Conditionals

## About Rules

- The two formats of rules
  - kwargs
  - (description, assertion) tuple
  - in the reference we show you both
### Using "Named" Rules

### Writing Your Own Custom Rules
