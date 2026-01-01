# LessML
A proposal for a no-nonsense, editable markup language. Like XML or JSON, but less.

#### Motivation
I saw the proposal for [HUML](https://huml.io/) on HN and thought to myself:
1. This wouldn’t be more editable than JSON for our support staff.
1. Why do these kinds of markup languages always prioritize readability over editability?
1. Why on earth is YAML considered human-readable?

That night I couldn’t sleep (not because of HUML), and in my head I started designing my own markup language. One that our support staff could intuitively understand, without any gotchas or more concepts than necessary - a boring, stable, editable markup language. So here it is: LessML.

## Example

```xml

<client="api-client">
  <host="api.lessml.io">
  <port="443">
  <schema="https">
  
  <clientCertificate>
    <pkey="pkey.pem">
    <cert="client.pem">
    <algorithm="sha256">
    <store=null>
  </clientCertificate>

  <endpoints>
    <endpoint="/ideas" description="New ideas">
    <endpoint="/implementations" description="All implementations">
    <endpoint="/issues" description="Issue list">
  </endpoints>

  <headers>
      <_entry key="Accept" value="application/lessml">
      <_entry key="Authorization" value="{{API_AUTH}}">
      <_entry key="User-Agent" value="Mozilla/5.0">
  </headers>

  # Comments

  ###
  Multi 
  line 
  comments
  ###
</client>

```

## Why
### Goals
LessML aims to be a markup language that, in order of importance:
1. Is easy for laypeople to learn and understand, so they can edit it
1. Is easy to read
1. Builds on familiar concepts
1. Is parsable in an efficient manner and keeps character count low

This makes LessML suitable for:
- Configuration files edited by nontechnical or semi-technical people
- Transporting these files over the network without too much overhead

##### Not goals
Replace XML, JSON, or TOML - maybe YAML though ;). 

#### Advantages over XML
- Less repetition
- No DTDs, schemas, namespaces, etc. Keep it simple and stupid.
- No header needed (I wanted it to be typeable without having to look things up)

#### Advantages over JSON
- Less error-prone. It is very hard for humans to spot a missing or superfluous comma.
- Closing tags on large sections improve readability; no need to guess which node a } closes.
- Comments in all versions.

#### Advantages over YAML
- Whitespace is readable, but not editable. Small mistakes break the schema. Those mistakes are invisible because they are whitespace - which leads to sadness.
- Whitespace also makes it not minifiable, which LessML is.

#### Advantages over TOML / INI
- TOML is great.
- But lists with complex objects are still hard.
- Especially for non-technical people.

#### Disadvantages
- Yes, I am aware of [xkcd #927](https://xkcd.com/927/).
- It looks like XML, which might lead to confusion.
- It looks like XML; many programmers will dislike it just because of that.
- It is not as powerful as XML, so it has limited use.
- Probably not as efficient to parse as JSON.
- It cannot be minified as much as JSON, especially not JSON5.

```
JSON5 vs JSON4 vs LessML
endpoints:[{path:"ideas",description:"New ideas"}]
{"endpoints":[{"path":"ideas","description":"New ideas"}]}
<endpoints><endpoint="/ideas"description="New ideas"></endpoints>
```

## Details

### Simple objects

Simple objects are basically the properties/fields of objects in JS (or other languages like Java, C#, ...).

```xml
<name="Sword of a Thousand Truths">
<owner=null>
```

They can be a single key-value pair as above, or multiple key-value pairs on one line:

```xml
<price="130.50" currency="gld">
```

In that case, they can have a **primary value**, which, when serialized, I would recommend be the first property defined. For example:

```ts
// Type Script
type Price = {
  price: number
  currency: string
}
```

```csharp
// C#
record Price(decimal Price, string Currency);
```

The `price` property would be the primary value in `<price="130.50" currency="gld">`. Usually, the first property of an object defines the object (`id`, `name`, `key`, ...). This helps reduce markup a little without hurting readability too badly.


### Complex objects
Complex objects are for objects with many fields/properties:

```xml
<item>
    <name="Sword of All Truths">
    <type="weapon">
    <price="130.50" currency="usd">
    <owner=null>
</item>
```

I chose to keep the </item> closing tag (instead of using something like </>) because in larger files it helps me orient myself. It is a sacrifice of compactness in favor of readability.

Complex objects can have primary values too, so the following is valid.

```xml
<item="name">
    <type="weapon">
    <price="130.50" currency="usd">
    <owner=null>
</item>
```
```ts
type Item = {
  name: string
  type: WeaponType
  price: Price
  owner: Player
}
```

Complex objects can also have **primary content**, which is explained later.

### Lists
Lists look like complex objects, but with the constraint that they can only contain the same tag as their children:

```xml
<inventory>
    <item="Sword of All Truths">
        <type="weapon">
        <price="130.50" currency="usd">
        <owner=null>
    </item>
    <item="Sword of Some Truths">
        <type="weapon">
        <price="0" currency=null>
        <owner="luetm">
    </item>
</inventory>
```

Simple objects are allowed too

```xml
<inventory>
    <item="Sword of All Truths" type="weapon">
    <item="Sword of Some Truths" type="weapon">
<inventory/>
```

As you can see, lists can have simple or complex types. The only constraint here is that if the first child is an `item`, all children must be `item`. Otherwise, the serializer should throw an exception.

`<inventory/>` is an empty list. The `<tag/>` notation is only used for empty lists or maps. `null` values for properties should be represented as `<lastSeen=null>`.

You might notice that


```xml
<inventory>
    <item="Sword of All Truths" type="weapon">
<inventory/>
```

is ambiguous. It could be a list or a complex object. In strongly typed languages, this is not a problem, since the deserializer has type information to know what to do with it. In untyped languages, this might pose a problem. However, I lack experience with those languages to know whether this is actually an issue. I am curious to learn more here.

### Maps / Dictionaries
Maps are lists, but with key-value pairs as items. Since they’re special in almost all languages, I decided to support them as a first-class concept:

```xml
<attributes>
    <_entry="type" value="weapon">
    <_entry="slot" value="two-handed">
    <_entry="damage" value="100">
</inventory>
```

All tags beginning with `_` are **reserved words**. Other such words could be `_type` for type information or `_ref` for object references.

### Primary content
Complex objects can have **primary content**, which compacts the markup a little.

```xml
<player name="luetm">
    <inventory>
        <item="Sword of some Truths" amount="1">
        <item="Light Leather" amount="1340">
    </inventory>
</player>
```

Can be written as

```xml
<player name="luetm" alive="true">
    <item="Sword of some Truths" amount="1">
    <item="Light Leather" amount="1340">
</player>
```

for an object like this

```ts
type Player = {
  name: string
  alive: bool
  inventory: Item[]
}
```

Since `name` and `alive` have already been defined, and `inventory` is a list of items, its tag `inventory` can be omitted, because it wouldn’t really add any information. I chose to add this because, in config files, I ran into this situation a lot.


### Comments
Comments can be one line or multi line. When minimizing, comments are purged.

```
# Single line comment

###
Sometimes one line is not
enough.
###
```

## Open Questions
#### Serializing as simple vs complex objects
You might have noticed that there is no clear rule for when to serialize an object as a simple or a complex object. For the moment, my opinion is that this should be left to the serializer or the serializer configuration.

For example, in C#, it would make sense to serialize value type objects, and reference type objects with fewer than four (or so) properties, as simple objects.


```xml
<string="abc">
<int="2">
<float="1.5">
<dateOnly="2026-01-01">
<dateTime="2026-01-01" time="13:44:01" kind="local">
<dateTimeOffset="2026-01-01" time="13:44:01" offset="+1" timezone="Europe/Zurich">

<patient="019b7997-6303-78ac-ab03-dc2f2cb36e74">
    <firstName="John">
    <middleName=null>
    <lastName="Doe">
    <sex="m">
    <dob="1970-01-01">
    <dod="2024-12-31">
    # ...
</patient>
```

I assume that, for most languages, simple rules like this would exist.

#### Ambiguousness of lists / complex objects
You might notice that

```xml
<inventory>
    <item="Sword of All Truths" type="weapon">
<inventory/>
```

is ambiguous. It could be a list or a complex object. In strongly typed languages, this is not a problem, since the deserializer has type information to know what to do with it. In untyped languages, this might pose a problem. However, I lack experience with those languages to know whether this is actually an issue. I am curious to learn more here.

#### Primary Content
Primary content seems nice in some circumstances; however, I fear it might be "too clever", especially if non-technical people edit these files. I'm still on the fence about whether to include it.

#### Map syntax

```xml
<attributes>
    <_entry="type" value="weapon">
    <_entry="slot" value="two-handed">
    <_entry="damage" value="100">
</inventory>
```

Is not too bad. But I like this too:

```xml
<attributes>
    <attribute="type" value="weapon">
    <attribute="slot" value="two-handed">
    <attribute="damage" value="100">
</inventory>
```

`attribute` would be an invented tag and could be anything. The advantage of the first syntax is that it tells an untyped language to deserialize the tags as a map, not a list.

## Feedback
Maybe I’ll post this on HN one day. Until then, you can direct any abuse toward [@luetm](https://x.com/luetm) on Twitter, or create an issue.
