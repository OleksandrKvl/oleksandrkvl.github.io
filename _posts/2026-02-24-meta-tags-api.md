---
layout: post
title:  "Making generic wrapper for a niche binary format using meta tags"
date:   2026-02-24 15:50:00 +0300
post_url: 2024-02-24-meta-tags-api
---

In this article I'll show how meta tags turned out to be a surprisingly powerful
feature and how I used them to implement core parts of a polymorphic wrapper
around a binary message format.

---

## Problem overview

I work for a company that makes various software for financial markets. Some of
those markets use
[FIX SBE format](https://github.com/FIXTradingCommunity/fix-simple-binary-encoding)
(or simply SBE) for message encoding. Details
of this format don't really matter here but small example will make overall
picture clearer. In SBE message layout is specified using XML schema:

```xml
<!-- user-defined types... -->
<type name="uint32_t" primitiveType="uint32"/>

<!-- user-defined messages... -->
<sbe:message name="msg_a" id="1">
    <field name="field_a" type="uint32_t" id="1"/>
    <field name="field_b" type="uint32_t" id="2"/>
</sbe:message>
```

In memory this message looks roughly like:

```cpp
struct msg_a_layout{
    // common for all messages from the same schema, provides message ID
    message_header header;
    uint32_t field_a;
    uint32_t field_b;
};
```

Thanks to its binary nature, the format is very fast to work with, much faster
than its ancestor, text-based
[FIX](https://en.wikipedia.org/wiki/Financial_Information_eXchange).

Typical way of working with SBE is to generate language-specific code from XML
schema (so the generated code is schema-specific) and include it into the
final app. This workflow is very similar to `protobuf` or `flatbuffers`.

At some point, being unhappy with our proprietary implementation, I decided to
write my own one - [sbepp](https://github.com/OleksandrKvl/sbepp). The main
goals I had in mind for that project were:
- zero overhead, in financial apps we care about performance
- provide only the basic functionality, building blocks that allow users to
    create their own efficient solutions to their specific problems
- provide all information from SBE schema, high-level things like field
    description and low-level details like memory offsets and sizes

As a result, generated code never allocates or does more work than one would do
by hand.

`sbepp` compiles XML schema into C++ header-only library, for most SBE entities
it generates a class with field-named accessors:

```cpp
// SBE-specific wrapper around `std::uint32_t`
class uint32_t{
    std::uint32_t value() const;
    // ...
};

class msg_a{
    uint32_t field_a();      // getter
    void field_a(uint32_t);  // setter

    uint32_t field_b();
    void field_b(uint32_t);
};

// other messages, types, etc.
```

Since SBE is a format from financial world, its primary use-case is encoding and
decoding market messages. It's quite natural to generate and include
schema-specific headers in market-specific component. We began to use `sbepp`
for this purpose and everything was quite good.

However, we also have a project that can work with multiple markets and SBE is
used internally as their common message format. The main market logic is still
located in market-specific components but there's also a core component that has
to be able to work with *any* SBE message in a generic way. In particular, it
should be able to:
- convert to/from JSON
- create and fill message given its schema/message/field name/value

And here's the problem, how to make a generic, runtime polymorphic wrapper
around a bunch of unrelated classes, each with its own set of field names and
types? There's no way to convert `msg->get_by_name("field_a")` into
`msg.field_a()` in a generic fashion. Even the return type for such a function
is unclear.

In theory, `sbepp` knows everything during code generation but again, it's not
possible to generate interface/implementation that will satisfy all users. Not
mentioning that most users don't even need this polymorphic wrapper, it's a very
ad-hoc problem.

So the goal of this task became to provide a minimal mechanism that allows users
to do literally anything they want without overhead. No ad-hoc solutions, only
building blocks for them.

However, there was a glimpse of light. If you remember, one of the initial
`sbepp` goals was providing all information from SBE schema. That means field
names, descriptions, offsets, basically every attribute that SBE schema defines.
The way it's been done is a standalone set of meta tags that represent SBE
schema structure and various traits to access properties via those tags:

```cpp
// tags
struct schema_name::schema{
    struct types{
        struct uint32_t{};
        // other types...
    };

    struct messages{
        struct msg_a{
            struct field_a{};
            // other fields...
        };
        // other messages...
    };
};

// traits
struct type_traits<schema_name::schema::types::uint32_t>{/*...*/};
struct message_traits<schema_name::schema::messages::msg_a>{/*...*/};
struct field_traits<schema_name::schema::messages::msg_a::field_a>{/*...*/};

// representation classes
class schema_name::messages::msg_a{
    uint32_t field_a();
    void field_a(uint32_t);
    // other fields...
};

// a mapping between them is available too
// representation type to meta tag
static_assert(std::same_as<
    sbepp::traits_tag_t<msg_a>,
    schema_name::schema::messages::msg_a>);
// meta tag to representation type
static_assert(std::same_as<
    message_traits<schema_name::schema::messages::msg_a>::value_type,
    schema_name::messages::msg_a>);
```

---

## Solution

Turns out that both, a generic de/serialization and by-name accessors, need
similar functionality - ability to navigate over the fields at compile time to
access their meta information (e.g. name) and, at the same time, to access field
value at run-time if needed.

From the previous section we know that `sbepp` provides a way to access both
kinds of information via two separate API which, however, is impossible to use
generically:

```cpp
// access runtime value
auto field_value = msg.field_a();
// access meta-information
constexpr auto field_name = field_traits<message_a::field_a>::name();
// but no way to combine them
```

To solve the first part of the problem, compile-time navigation, we can use a
very simple tool - a type list.

```cpp
template<typename... Types>
struct type_list{};
```

Existing meta-tags and their traits already provide all possible information, we
only need to put them inside a list and provide via trait:

```cpp
// concrete message traits
template<>
struct message_traits<msg_a>{
    using field_tags = type_list<
    schema_name::schema::messages::msg_a::field_a,
    schema_name::schema::messages::msg_a::field_b>;
};
```

Same approach is used to represent all children tags within their parents
traits, not just message fields. For more convenience, let's add `is_ABC_tag`
trait so user can figure out tag kind and use proper trait to access its
properties, e.g. `is_enum_tag<T>` followed by `enum_traits<T>`.

The next step is a runtime access by compile time tag. For that we add a couple
of functions: `get/set_by_tag<Tag>`. Every generated class that has children to
access by tag implements it by simply forwarding to a proper name-based
accessor:

```cpp
class msg_a{
    // normal accessors
    uint32_t field_a();
    void field_a(uint32_t);

    // for exposition only
    auto get_by_tag(message_a::field_a){
        return field_a();
    }

    void set_by_tag(message_a::field_a, uint32_t value){
        return field_a(value);
    }
};

// public methods forward to internal ones above
sbepp::get_by_tag<message_a::field_a>(msg);
sbepp::set_by_tag<message_a::field_a>(msg, 123);
```

Now let's see how these two seemingly minor additions allowed a very rich set of
functionality. Actually, they allow nearly everything one can imagine about
working with SBE data.

---

## Examples

*Disclaimer. Examples are intentionally simplified and are not intended for
direct use as-is. The main goal is to demonstrate overall approach, interested
reader can look near the bottom of the
[Examples](https://oleksandrkvl.github.io/sbepp/1.7.0/examples.html) page for
examples that do compile.*

Common technique for most examples is iteration over a type list that contains
children tags of various kind. Its implementation is not important here, just
assume it has this signature:

```cpp
// For each tag `T` in `TypeList`, calls `cb(T{})` until it returns `true`.
// Returns `true` if `cb` returned `true`, `false` otherwise.
template<typename TypeList>
auto for_each_tag_until(auto cb);
```

Also, I won't specify `sbepp` namespace to make the code more compact.

---

### Enum to string

`sbepp` represents SBE enums as scoped C++ enums for which there's a classic
problem: you have an enum and want to log its name. To solve it we can iterate
over all known enumerator values to find a match with the current one:

```cpp
// returns `nullptr` if value is unknown
template<typename Enum>
const char* enum_to_string(Enum e){
    const char* res{};

    for_each_tag_until<enum_traits<traits_tag_t<Enum>>::value_tags>(
        [e, &res]<typename Tag>(Tag){
            if(enum_value_traits<Tag>::value() == e){
                res = enum_value_traits<Tag>::name();
                return true;
            }
            return false;
    });

    return res;
}
```

Btw, here's how to do the opposite conversion:

```cpp
template<typename Enum>
Enum string_to_enum(std::string_view name){
    Enum res{};

    for_each_tag_until<enum_traits<traits_tag_t<Enum>>::value_tags>(
        [name, &res]<typename Tag>(Tag){
        if(enum_value_traits<Tag>::name() == name){
            res = enum_value_traits<Tag>::value();
            return true;
        }
        return false;
    });

    return res;
}
```

Note that it's not the only possible implementation. While for enum-to-string
conversion and relatively small enums we can expect compiler to optimize into a
lookup table, for the opposite conversion a compile time hash table can be used
to improve the performance in complex cases (again, can be decided at compile
time based on the number of enumerators).

---

### Handle all schema messages

Another common use-case is when you receive SBE message from network and need to
handle it. To do that, you first need to figure out message type by looking at
message header and then create corresponding message wrapper. Instead of writing
a bunch of `if`-s or `case`-s for each message type manually, we can make a
generic helper that will do that for us:

```cpp
// calls `cb(Message)` when a buffer given by `data`/`size` pair represents
// any `Message` from schema represented by `SchemaTag`.
// returns `false` if message is unknown
template<typename SchemaTag>
bool handle_schema_message(const char* data, const size_t size, auto cb){
    // implementation of this function is not important here
    const auto msg_id = get_message_id_from_header<SchemaTag>(data, size);

    return for_each_tag_until<schema_traits<SchemaTag>::message_tags>(
        [data, size, msg_id, &cb]<typename Tag>(Tag){
            if(message_traits<Tag>::id() == msg_id){
                cb(message_traits<Tag>::value_type{data, size});
                return true;
            }
            return false;
    });
}

// usage:
handle_schema_message<MySchema::schema>(data, size, overloaded{
    [](MySchema::messages::message_a msg){},
    [](MySchema::messages::message_b msg){},
    // other messages...
});
```

Note that this helper allows to either ensure that all messages are handled or
to handle only some target messages by adding `[](auto){}` at the end of the
overload set. Optimizer will see it and eliminate branches for ignored messages.

---

### Access by name

So far we've seen only the usage of tags iteration, let's add `get/set_by_tag`
into the mix. Here's simplified by-name getter:

```cpp
template<typename Message>
uint64_t get_by_name(Message m, std::string_view field_name){
    uint64_t res{};

    for_each_tag_until<message_traits<traits_tag<Message>>::field_tags>(
        [field_name, m, &res]<typename Tag>(Tag){
        if(field_traits<Tag>::name() == field_name){
            res = get_by_tag<Tag>(m).value();
            return true;
        }
        return false;
    });

    return res;
}
```

It assumes all message fields type can be represented by `uint64_t`. The full
implementation of this method can return a variant of basic types or even some
extra info, for example enumerator name along with its value. It also should
handle complex field paths instead of just a single field name, e.g.
`field_a.sub_field_b`. That's all doable using the same technique. Note that
this approach relies on string comparison, it compares given field name to all
known field names at runtime. This might be suboptimal for some applications.
Further optimization is quite trivial, we can compare numbers instead of strings
by relying on some compile time hash function:

```cpp
// constexpr-friendly hash implementation
constexpr uint64_t get_hash(const std::string_view str);

const auto field_name_hash = get_hash(field_name);

for_each_tag_until<message_traits<traits_tag<Message>>::field_tags>(
    [field_name, field_name_hash, m, &res]<typename Tag>(Tag){
        if((get_hash(field_traits<Tag>::name()) == field_name_hash)
            && (field_traits<Tag>::name() == field_name)){
                res = get_by_tag<Tag>(m).value();
                return true;
        }
        return false;
});
```

Because field names are available at compile time, their hash can be computed at
compile time too so at runtime this will mostly compare numbers. One string
comparison is unavoidable if we want to protect against collisions. To give you
an idea how to extend it to support, for example, SBE enums, we can leverage
`if constexpr` to distinguish field kind:

```cpp
for_each_tag_until<...>(
    [...]<typename Tag>(Tag){
    // as before
    if(names_match){
        // field tag != its value type tag
        using value_type_tag = field_traits<Tag>::value_type_tag;
        if constexpr(is_enum_tag_v<field_type_tag>){
            res = to_underlying(get_by_tag<Tag>(m));
        }
        else{
            res = get_by_tag<Tag>(m).value();
        }
        return true;
    }
    return false;
});
```

Since SBE enums are scoped C++ `enum`-s, we use `to_underlying()` to get their
underlying value instead of `value()` for numeric fields.

OK, let's see the setter:

```cpp
template<typename Message>
bool set_by_name(Message m, std::string_view field_name, uint64_t value){
    return for_each_tag_until<
        message_traits<traits_tag_t<Message>>::field_tags>(
        [m, value]<typename Tag>(Tag){
            if(field_traits<Tag>::name() == field_name){
                set_by_tag<Tag>(m, value);
                return true;
            }
            return false;
    });
}
```

Again, the real implementation will need to handle various field types and check
that given `value` can be properly converted to the underlying field type.

---

### De/serialization

Conversion to/from any other format is not a big deal now as we can freely
iterate over the message structure and get any representation of its fields,
let's imagine we want JSON:

```cpp
template<typename Message>
std::string to_json(Message m){
    json j;

    for_each_tag_until<message_traits<traits_tag_t<Message>>::field_tags>(
        [m, &j]<typename Tag>(Tag){
            j[field_traits<Tag>::name()] = get_by_tag<Tag>(m).value();
            return false;
        });

    return j.to_string();
}

template<typename Message>
Message from_json(const json& j){
    Message m;

    for_each_tag_until<message_traits<traits_tag_t<Message>>::field_tags>(
        [&m, &j]<typename Tag>(Tag){
            set_by_tag<Tag>(m, j[field_traits<Tag>::name()]);
            return false;
        });

    return m;
}
```

Of course, the real implementation is a bit harder than that because message can
have nested structure and many other field types. But what's important is that
it's possible to tailor the code to specific needs, for example, in our internal
implementation enums are represented as JSON strings or numbers for unknown
values, optional numeric fields are numbers or JSON `null`, arrays are JSON
strings or JSON arrays of numbers and so on.

---

## Final thoughts

Being able to navigate over the message structure at compile time and have
access to both its compile and run-time properties has become a surprisingly
powerful mechanism. I already used it in `sbepp` to replace some generated code
with a single generic implementation that works for all. When I initially wrote
this project, I was sure that "normal" accessors are a key part and meta tags
just provide an extra info that is useful in some relatively rare cases. But
with this new mechanism it's kinda vice versa. Meta tags and their traits
actually contain *all* information about the schema. Instead of forwarding
`get/set_by_tag` to normal accessors that contain hardcoded values and types,
it's possible to reverse it and let `get/set_by_tag` to actually calculate them
at compile time. In fact, it's possible to generate SBE implementation for any
other language now without touching `sbepp`'s own schema compiler. It
effectively becomes a project-specific reflection mechanism and it works even in
C++11.