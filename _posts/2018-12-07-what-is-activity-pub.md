---
title: "What is ActivityPub?"
description: >
  In MoodleNet project, we are implementing a generic ActivityPub protocol.
  One of our goals is to make the world freer and less dependent of large corporates
tags: [MoodleNet, ActivityPub]
header:
  teaser: /assets/images/what_is_activity_pub/activitypub.png
  image: /assets/images/what_is_activity_pub/activitypub.png
  overlay_image: /assets/images/what_is_activity_pub/activitypub.png
  overlay_filter: 0.5
last_modified_at: 7/12/2018

---

This was my first question when I joined to MoodleNet team. Now, after studying it deeply for a few weeks, I feel able to give a proper answer. So I thought of writing a blog post, with my goal being to offer a **nice introduction to ActivityPub for technical people**. If you're interested in my more general impressions on the *fediverse* and our approach to building a federated app, I recommend you to read [my previous blog post about our goals](/moodlenet-more-than-a-technical-project/).

{% include figure image_path="/assets/images/what_is_activity_pub/activitypub.png" alt="ActivityPub" %}

*[ActivityPub](https://www.w3.org/TR/activitypub/) is a protocol for decentralized social networks, which can also be extended to create all kinds of federated apps.* So if you are thinking of building one it is very interesting to know how this standard works. It is important to notice it has two different parts:

* Client to Server API
* Server to Server API

A service can decide if it wants to implement only one or both of them. This decision just depends on the needs of the project. If you're a frontend developer, you might use the Client to Server API. You will be focused on the Server to Server API when you only care about the federation part though.

## ActivityStreams 2.0

The ActivityPub message format is JSON-LD, which can be used just like regular JSON and we'll do this for this blog post. The vocabulary used in this protocol is defined by [ActivityStreams 2.0](https://www.w3.org/TR/activitystreams-core/) and ActivityPub just adds a couple of requirements more on top of it. In this vocabulary only exists two kind of entities: `Link` and `Object`. Here we will focus on `Object` and its subtypes. **An `Object` has several fields and only two of them are required by ActivityPub:** `id` and `type`. The most important properties are:

* `type`: which defines the type of object. It can be a string or an array if the object implements more than a type. This is mandatory in ActivityPub.
* `id`: a unique identifier for the object. For ActivityPub this field should be a URL. The interesting part is that you can retrieve the full object information making a `GET` request to this URL with the ActivityStreams accept header. This allows us to send just the `id` instead the full object information in many cases.
* `content`: The content or textual representation of the Object, usually an HTML string.
* `name`: A simple, human-readable, plain-text name for the object.
* `summary`: A natural language summary of the object encoded as HTML.
* `image`: A link to an image that represents an object.
* `@context`: This is the only necessary property to accomplish with JSON-LD specification. In our examples we just set to `"https://www.w3.org/ns/activitystreams"`.

## Natural Language Values

So any object can have those fields. It is interesting that any field likely to be translated, like `content`, `name` or `summary`, can be renamed to a "map variant" to include different languages, ie:

```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Note",
  "id": "https://mycoolserver.com/notes/1",
  "content": "This is a translated note"
}

{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Note",
  "id": "https://mycoolserver.com/notes/1",
  "contentMap": {
     "es": "Esto es una nota traducida",
     "en": "This is a translated note"
  }
}
```

## "Data Container" Objects

You may have noticed that the type of the previous object is `"Note"` instead of `"Object"`. A `Note` is a subtype of object defined by ActivityStreams too, and it is used to represent any short written work. A subtype means that share the same fields, but it may define more if needed.

There are some more defined types like `Article`, `Image`, `Page`, `Document`, `Video`, and more. While it may feel strange to not see something like comment or similar, ActivityStreams is very generic and we have to keep this in mind. The recommendation is to use `Note` because it ["Represents a short written work"](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-note).
Those objects may be considerated "data container" objects. They don’t define more fields and they are used to transmit information.

## Activity

In addition to those "data container" objects, ActivityStream defines `Activity` objects. The type of those objects may be more intuitive:

* Create
* Delete
* Update
* Add
* Remove
* Join
* Follow
* etc

They are the type of actions that you can expect from a social network. Special attention should be paid to  the `Announce` activity, as it used to call attention to particular objects or notifications. It can be understood like a "retweet" on Twitter, or a simple way to share an already created object in your social network.

Activities may have 6 additional properties compared to a simple object:

* `object`
* `actor`
* `target`
* `origin`
* `result`
* `instrument`

Of course, not all of them make sense in every type of activities, so they are optional. So imagine you want to represent that Luke created the previous note, you can write:

```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "id": "https://mycoolserver.com/activity/1",
  "actor": "https://mycoolserver.com/actor/luke",
  "object": "https://mycoolserver.com/note/1"
}
```

Here, you only used `actor` and `object`. The more common ones. You don't need to use the rest of them if they do not make sense in your activity.

One important thing to keep in mind is that the ids of the objects will not always appear, and the complete specification of them may appear instead. The previous example could be written in the following way, both being completely valid:

```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "id": "https://mycoolserver.com/activity/1",
  "actor": {
     "type": "Person",
     "id": "https://mycoolserver.com/actor/luke",
     "name": "Luke"
  }
  "object": {
    "type": "Note",
    "id": "https://mycoolserver.com/notes/1",
    "contentMap": {
       "es": "Esto es una nota traducida",
     "en": "This is a translated note"
  }
}
```

## Actor

If you take a close look at the previous example you will see that Luke is a `Person`, this is a type of Object we haven’t yet mentioned. `Person` is one of the 5 types of `actors` that ActivityStreams defines:

* `Person`
* `Group`
* `Organization`
* `Application`
* `Service`

Actors are objects that may execute the different activities, this means they can be used as the `actor` field on an Activity. An actor has two mandatory properties as well:

* `inbox`
* `outbox`

The previous two properties are mandatory for any actor. The following properties are not but they are recommended to be implemented:

* `following`
* `follower`
* `liked`

## Collection

All these five properties are from a core type we haven't talked about yet: `Collections`. As you’d expect, a `Collection` is simply a group of objects. They also have additional properties, ie: `totalItems` with the number of elements in the `Collection` or `items` where you’ll find the objects in the collection.

So this means that the `inbox` of an actor is just a `Collection` containing all the activities received by the actor. The `outbox` contains all the activities sent by the actor. Any time an actor likes an object, this is added to their `liked` collection.

Obviously, after a while, those collections cannot be managed with just an array. It will be impractical to manage all the followers of a popular person in a simple array. For this reason, we have the last core type of ActivityStreams: the `CollectionPage` which should be used to paginate a full `Collection`.

## Core types

So those are the 5 core object types that ActivityStreams defines:

* `Object`
* `Actor`
* `Activity`
* `Collection`
* `CollectionPage`

All of them are also an `Object`, so they share most of the fields. Once you understand this vocabulary it becomes easier to work with the ActivityPub protocol.


## Targeting

A big difference from working with other private social networks is the "Audience Targeting". All objects can use 5 optional fields for this:

* `to`
* `bto`
* `cc`
* `bcc`
* `audience`

{% include figure image_path="/assets/images/what_is_activity_pub/mail_boxes.jpg" alt="Mail Boxes" %}

Not only that, but it is the client application that fills in these fields. It can be just an actor, but maybe is your followers collection or it is just "public". This can be better understood with the following examples.

## Examples

### Sending a private message

First of all, messages are always activities in ActivityPub. So if you want to send a private message to Ben, you have to build a `Create` activity with a `Note`. **It is equally important to set the field `to` to only Ben’s id, because is a private message between Alyssa and Ben.** You don't want to make it public or to share with your followers. The JSON could be something like the following:


```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "id": "https://social.example/alyssa/posts/a29a6843-9feb-4c74-a7f7-081b9c9201d3",
  "to": [
    "https://chatty.example/ben/"
  ],
  "actor": "https://social.example/alyssa/",
  "object": {
    "type": "Note",
    "id": "https://social.example/alyssa/posts/49e2d03d-b53a-4c4c-a95c-94a6abf45a19",
    "attributedTo": "https://social.example/alyssa/",
    "to": [
      "https://chatty.example/ben/"
    ],
    "content": "Say, did you finish reading that book I lent you?"
  }
}
```

This is a `Create` activity because, of course, the message didn’t already exist. As you can see, the actor property is just a Link with Alyssa's ID. Not need to send the full object. Similarly with the property `to` with Ben's ID, but in this case in an array.

ActivityPub defines that a client should always create new activities talking to the server where the user’s account resides. In this example, Alyssa account resides at "https://social.example/". Once the message is received by the server it is persisted so it can be requested at any time, but also, it detects that the recipient of the message does not belong to this server. This means that it’s now the server which should communicate this message to Ben's server, in this case: "https://chatty.example/ben/". It will send a post request to Ben's inbox. *It should then be Ben's server that notifies Ben that he has a new message*.

### Add an image to an album

In this example we are working with images. The user Alyssa just move an image from her "Camera Roll" album to "My Cat Pictures". The client application generates the following activity to represent the action that just happened:

```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "id": "https://social.example/alyssa/activities/10239"
  "summary": "Alyssa added a picture of her cat to her cat picture collection",
  "type": "Add",
  "actor": "https://social.example/alyssa",
  "to": "https://social.example/alyssa/followers",
  "object": {
    "type": "Image",
    "name": "A picture of my cat",
    "id": "http://example.org/img/cat.png"
  },
  "origin": {
    "type": "Collection",
    "name": "Camera Roll"
    "id": "https://social.example/alyssa/collection/1"
  },
  "target": {
    "type": "Collection",
    "name": "My Cat Pictures",
    "id": "https://social.example/alyssa/collection/278"
  }
}
```

This activity is public only for the Alyssa’s followers. The client application send this activity to the Alyssa’s server. Now it is the server which has to locate all her followers and make sure that they receive this activity in their `inbox`. Maybe they are all locals, they resides on the same server, or maybe it has to send this activity to many external servers.

### Follow request

In this scenario, Sally asks John to be friends in our application:

```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "id": "http://example.org/connection-requests/123",
  "summary": "Sally requested to be a friend of John",
  "type": "Offer",
  "actor": "https://sally.example.org",
  "object": {
    "summary": "Sally and John's friendship",
    "id": "http://example.org/connections/123",
    "type": "Relationship",
    "subject": "https://sally.example.org",
    "relationship": "http://purl.org/vocab/relationship/friendOf",
    "object": "https://john.example.org"
  },
  "target": "https://john.example.org",
  "to": "https://john.example.org",
  "cc": "https://www.w3.org/ns/activitystreams#Public"
}
```

We make this request using the `Offer` activity type with a `Relationship` object and with the target John. Now John can accept the offering with:

```json
{
     "@context": "https://www.w3.org/ns/activitystreams",
      "summary": "John accepted Sally's friend request",
      "id": "http://example.org/activities/122",
      "type": "Accept",
      "actor": "https://john.example.org",
      "object": "http://example.org/connection-requests/123",
      "inReplyTo": "http://example.org/connection-requests/123",
      "context": "http://example.org/connections/123",
      "to": "https://sally.example.org/",
      "cc": "https://www.w3.org/ns/activitystreams#Public"
}

```
It is interesting to see that in this application, the creator decided to make public this kind of interactions. To make public an object we have to add to any "targeting property" the value `"https://www.w3.org/ns/activitystreams#Public"`, in this case in the `cc` property.

## Conclusions

It is important to remember that ActivityPub is a protocol. It defines the expected behaviour but never the implementation details behind it. The client sends activities to the server. The server federates the same activities to other servers. The client can fetch for activities and the server should provide them. The way you do it is completely up to you.

There are a lot more things around ActivityPub, here we only scratched the surface. There are more types in the vocabulary, the possibility of extending it with your own types and properties, security recommendations, side effects, client and server responsibilities, etc. The goal of this article was to give a small introduction without getting lost in technical formalities. I hope you enjoyed.

---

This post was also published in [MoodleNet blog](https://blog.moodle.net/2018/what-is-activitypub/).

Photo by [Marius Christensen](https://unsplash.com/photos/UXfi8LyqGDk)
