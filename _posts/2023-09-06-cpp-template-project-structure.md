---
title: Serialization and Separating Declaration and Definition in Header Only Libraries
categories: [programming]
tags: [cpp]
last_modified_at: 2023-09-06-T12:00:00-05:00
---

In my personal projects, and at work, I have been venturing very heavily into templated libraries.
Sometimes the project was completely header-only, other times it was a hybrid. In my most recent endeavor,
I was challenged with creating a templated library that was "closed" off from additional implementations.
These challenges pressured me into understanding the purpose of certain design patterns and how
larger projects, like Boost, utilized them to build incredible libraries. In this post, I am going
to walkthrough three design-ideas to approaching header-only libraries and discuss their pros and cons.

## 1. Header Only (Literally)

The first C++ structured project I learned was separation of declaration and definition. That is,
use the `.hpp` files to store declarations only. A statement saying "I promise this definition will
be available, eventually, when compiling." In the following example, I am going to walk through a design
decision for serializing and deserializing JSON into a C++ object. When first addressing this problem,
I decided to write code similar to the following:

### IParsable.hpp

```cpp
#ifndef I_PARSABLE_HPP
#define I_PARSABLE_HPP

#include <string>

class IParsable {

  virtual void serialize(std::ostream& os) = 0;
  virtual void deserialize(std::istream& is) = 0;

}; // class IParsable

#endif // I_PARSABLE_HPP
```

### User.hpp

```cpp
#ifndef USER_HPP
#define USER_HPP

#include <IParsable.hpp>

class User : public IParsable {
public:
  std::string UserId;
  std::string FirstName;
  std::string LastName;
  int Age;

  void serialize(std::ostream& os) const override {
    // TODO: Add serialize() functionality here
  }

  void deserialize(std::istream& is) override {
    // TODO: Add deserialize() functionality here
  }

}; // class User

#endif // USER_HPP
```

This implementation works and is a viable solution and takes on a very object oriented approach. It is simple
and modular. There is a way we handle it though that I find not ideal. We have to create an instance of the object
before we can use the serialize and deserialize methods. This means we have to make an empty object initially and
then load in data into it. Additionally, the functionality of serialize and deserialize never change between
objects, or are not instance specific.

```cpp
#include <memory>
#include <iostream>

const std::string& input = read_json_from_file("User.json");
auto user = std::make_unique<User>();
user->deserialize(input);

std::cout << "UserID: " << user->UserId << "\nAge: " << user->Age << '\n';

std::istream is;
user->serialize(is);
```

One solution to keep this approach is to simply create a wrapper around the serialization and deserialization methods.

### ParsableFactory.hpp

```cpp
template <typename Parsable>
class ParsableFactory {
public:
  static_assert(std::is_base_of<IParsable, Parsable>::value, "Parsable must be derived from IParsable");

  static std::unique_ptr<Parsable> deserialize(std::istream& is) {
    std::unique_ptr<Parsable> result = std::make_unique<Parsable>();
    result->deserialize(is);
    return result;
  }

  static void serialize(const Parsable& parsable, std::ostream& os) {
    parsable.serialize(os);
  }

}; // class ParsableFactory
```

It is a great way to help scale the program for if there are many classes that inherit and implement
the `IParsable` interface. All interactions can come directly from the `ParsableFactory`, and if there
is a requirement to enforce this behavior, it is possible to protect the serialize and deserialize
functions in the class and create a `friend` relationship to the factory.

```cpp
std::istream is(/* serializable string*/);
const User user = ParsableFactory<User>::deserialize(is); // OK

/* ... */

User user;
user.serialize(); // Error: protected member
```

There is one particular aspect I do not like about this implementation. The primary one is that
the serialize and deserialize are not reliant on a particular instance to be used. And in my head,
we should be using deserialize as a means to generate a new `IParsable` object, not to fill an
already existing object. Due to this, there is another way we can enforce having this functionality
while also keeping the function members static.

```cpp
template <typename Object>
class IParsable {

  static std::string serialize(const Object& object) {
    return Object::serialize_internal(object);
  }

  static Object deserialize(const std::string& input) {
    return Object::deserialize_internal(input);
  }
}; // class IParsable
```

The new `IParsable` is acting like the `ParsableFactory` while also enforcing that a static
implementation of `deserialize_internal` and `serialize_internal` be implemented in the
inheriting class. The new `User` class would then look like:

```cpp
class User : public IParsable<User> {
public:
  std::string UserId;
  std::string FirstName;
  std::string LastName;
  int Age;

protected:
  friend IParsable;

  static std::string serialize_internal(const User& object) {
    // Add serialize here
    return {};
  }

  static User deserialize_internal(const std::string& input) {
    // Add deserialize here
    return {};
  }

}; // class User
```

Although it is almost identical, I prefer this one a lot more. The
way you utilize any of the serialize or deserialize functions is a bit different, not
requiring the user to understand which classes support the `IParsable` interface, which they
would typically find out during compile time from the assert statements. This way,
we can get function suggestions straight from the class we want to parse:

```cpp
const std::string input = /* read in content */;
const User user = User::deserialize(input);
```

For the rest of this post, I am going to be using the static member function approach.
Throughout this, I was writing assuming everything was being placed in the header files.
Which leads me to a second and my preferred way to writing quality C++ code with template and
non-template functions.

## 2. Separate Declaration and Definition

When it comes to templated classes and functions, we really do not have a choice but to
put them in the header. But that doesn't mean the definition has to be baked in
with the header declarations. Taking a look at the [Boost Library](https://github.com/boostorg/boost),
they separate their implementation by placing the definitions in an `.ipp` file usually in
a `impl` folder. The i stands for inline, as all definitions in it need to be inlined. We can
utilize this approach to separate the implementation in the `IParsable` file:

### IParsable.hpp

```cpp
#ifndef IPARSABLE_HPP
#define IPARSABLE_HPP

#include <string>

template <typename Object>
class IParsable {

  static std::string serialize(const Object& object);

  static Object deserialize(const std::string& input);
}; // class IParsable

#include <ProjectName/impl/IParsable.ipp>

#endif // IPARSABLE_HPP
```

### impl/IParsable.ipp

```cpp
#ifndef IPARSABLE_IPP
#define IPARSABLE_IPP

#include <string>
#include <ProjectName/IParsable.hpp>

template <typename Object>
std::string IParsable<Object>::serialize(const Object& object) {
  return {};
}

template <typename Object>
Object IParsable<Object>::deserialize(const std::string& input) {
  return {};
}

#endif // IPARSABLE_IPP
```

The goal is to create a `impl` folder and place the `IParsable.ipp` inside of it.
Then, at the very bottom of the header file, include `IParsable.ipp`. As long as they
are templated or inlined, then this separation will work just fine. I tend to
only use the `.ipp` if there are templated functions being used. Anything that doesn't have
to be used in the header can go into the `.cpp` files as usual.

## 3. "Closing" off the templated functions

One concern that many people have with templated functions is that we leave any user open to
pass anything and implement anything for the template. There are ways to help avoid this if
this is not wanted. One method that I found was to instead of include the `.ipp` file at the bottom
of the header, to include it in the `.cpp` file and then specialize the template for each type
you want supported.

### IParsable.cpp

```cpp
#include <ProjectName/impl/IParsable.ipp>
#include <ProjectName/User.hpp>

IParsable<User>;
```

Now there are only specializations for the `User` class.
