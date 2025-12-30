# LessML
A proposal for a no-nonsense, editable markup language. Like XML or JSON, but less.

#### Motivation
I saw the proposal for [HUML](https://huml.io/) on HN and thought to myself:
1. This wouldn’t be more editable than JSON for our support staff
1. Why do these kinds of markup languages always prioritize readability over editability?
1.Why on earth is YAML considered human?

That night I couldn’t sleep (not because of HUML), and in my head I started to design my own markup language. One that our support staff could intuitively understand, without any gotchas or more concepts than necessary - a boring, stable, editable markup language. So here it is: LessML.

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

More examples and explanations below.

## Philosophy
### Goals
LessML aims to be a markup language that, in order of importance:
 1. Is easy for laypeople to learn and understand, so they can edit it
 1. Is easy to read
 1. Builds on familiar concepts
 1. Is parsable in an efficient manner and keeps character count low

This makes LessML suitable for:
 - Configuration files edited by non- or semi-technical people
 - Transporting these files over the network without too much overhead
 - In theory, a simple markup language like HTML - not that HTML needs replacing

 ##### Not goals
 Replace XML, JSON, or TOML - maybe YAML though ;). 

 #### Advantages over XML
 - Less repetition
 - No DTD, schemas, namespaces, etc. Keep it simple and stupid.
 - No header needed (I wanted it to be 'typeable' without having to look things up)
 
 #### Advantages over JSON
 - Less error-prone. It’s very hard for humans to spot a missing or superfluous comma.
 - Closing tags on large sections improve readability; no need to guess which node a } closes.
 - Comments for all versions.
 
 #### Advantages over YAML
 - Whitespace is readable, but small mistakes break the schema. Those mistakes are invisible, because they’re whitespace,  - which leads to sadness.
 - Whitespace also makes it not minifiable, which LessML is.

 #### Advantages over TOML / Ini
 - TOML is great.
 - But lists with complex objects are still hard.
 - Especially for non-technical people.

#### Disadvantages
 - Yes, I am aware of [xkcd #927](https://xkcd.com/927/).
 - It looks like XML, which might lead to confusion.
 - It looks like XML; many programmers will dislike it just because of that.
 - It’s not as powerful as XML, so it has limited use.
 - Probably not as efficient to parse as JSON.
 - It can’t be minified as much as JSON, especially not JSON5.

```
JSON5 vs JSON4 vs LessML
endpoints:[{path:"ideas",description:"New ideas"}]
{"endpoints":[{"path":"ideas","description":"New ideas"}]}
<endpoints><endpoint="/ideas"description="New ideas"></endpoints>
```

## Details

### Simple objects

Simple objects are basically the properties/fields of objects in JS (or other languages like C#).

```
<name="Sword of a Thousand Truths">
<price="130.50" currency="GLD">
<owner=null>
```

Simple objects have a **primary value**, which when serialized I would recommend to be the first property defined. For example:

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

In case of `<name="Sword of Truth">`, the property would just be a string.

### Complex objects
Sometimes objects are big, and to avoid an endless string of attributes, we can display them as complex objects.

```xml
<item>
    <name="Sword of All Truths">
    <type="weapon">
    <price="130.50" currency="usd">
    <owner=null>
</item>
```

I chose to keep the `</item>` closing tag because in larger files it helped me orient myself. It’s a sacrifice of compactness for readability.

You might ask yourself what the rules are for when an object should be serialized as a simple versus a complex object. The answer is: there are none. It’s completely up to the serializer. It could depend on property count, something like value type vs. reference type in C#, be controlled by attributes/decorators, etc. It probably should be an option in the serializer, since the “correct” answer depends on the use case.

Also, you can't use attributes in complex types. Something like the following is invalid:

```xml
<item="name">
    <type="weapon">
    <price="130.50" currency="usd">
    <owner=null>
</item>
```

### Lists
Lists look like complex objects, but have the constraint that they can only contain the same tag as its children:

```xml
<inventory>
    <item>
        <name="Sword of All Truths">
        <type="weapon">
        <price="130.50" currency="usd">
        <owner=null>
    </item>
    <item>
        <name="Sword of Some Truths">
        <type="weapon">
        <price="0" currency=null>
        <owner="me">
    </item>
</inventory>

# or

<inventory>
    <item="Sword of All Truths" type="weapon">
    <item="Sword of Some Truths" type="weapon">
<inventory/>

</inventory/>
```

As you can see, lists can have simple or complex types. The only constraint here is that if the first child is an `item`, all children must be an `item`. Otherwise the serializer should throw an exception.

`<inventory/>` is an empty list. The `/>` notation is only used for empty lists or maps.

A problem here is that 

```xml
<inventory>
    <item="Sword of All Truths" type="weapon">
<inventory/>
```

is ambiguous. It could be a list or a complex object. In strongly typed languages, this is not a problem, since the deserializer has type information to know what to do with it. Loosely or untyped language people: Please let me know if this even poses a problem in your world.


### Maps / Dictionaries
Maps are lists, but with key-value pairs as items. Since they’re special in almost all languages, I decided to support them as a first-class citizen. I'm still debating with myself, which of the following syntaxes I prefer: 

```xml
<attributes>
    <_entry key="type" value="weapon">
    <_entry key="slot" value="two-handed">
    <_entry key="damage" value="100">
</inventory>

<attributes>
    <_entry="type" value="weapon">
    <_entry="slot" value="two-handed">
    <_entry="damage" value="100">
</inventory>

<attributes>
    <attribute="type" value="weapon">
    <attribute="slot" value="two-handed">
    <attribute="damage" value="100">
</inventory>
```

The first two makes use of a **reserved word** which would be fixed for maps. Other such words could be `_type` for type information or `_ref` for object references. The last one basically lets the user chose what to name the objects. The problem here is though, that a serializer wouldn't know what to do with that and just always pick `entry` or something. At the moment, my favourite is #2.

### Comments
Comments can be one line or multi line. When minimizing, comments are purged.

```
# Single line comment

###
Sometimes one line is not
enough.
###
```


## Feedback
Maybe I’ll post this on HN one day. Until then, you can direct any abuse toward [@luetm](https://x.com/luetm) on Twitter, or create an issue.