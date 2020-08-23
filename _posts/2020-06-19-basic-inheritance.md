---
layout: post
title:  "Back to basics: C++ Inheritance in a nutshell"
thumbnail: assets/images/cpp-logo.svg
categories: [cpp, design patterns]
---

C++ is an object-oriented language and inheritance can be used to define relationships
between objects in the form of class hierarchies. It provides a mean to structure
and organize your code. Assuming that you already know the basics of inheritance,
 we will look at how and when to best use it. Especially we look at how to use it 
for runtime and compiletime polymorphism.

Before you start using inheritance to describe your object's relationships, consider
also other alternatives. Often composition is a better approach to structure
your code. A simple guideline is to use composition when the relationship between
your objects can be described with a `has-a` relationship e.g. the `Robot` class
has a `Leg`). And use inheritance when it is better described with an `is-a` relationship
e.g. a `Dog` is an `Animal`. However that does not completely work in some cases. It
is better to follow the [Liskov Substitution Principle](https://stackoverflow.com/a/584732).
This principle basically states, that if you chose inheritance, then the `Derived` class should
be able to be used anywhere, where the `Base` class is used i.e. you could substitute
all your `Base`s with `Derived`s.

To give a simple example:
At first it might seem like a good idea to make your `Penguin` class inherit from
the `Bird` class because a pengin *is a* bird. However this does not work well in code if
the `Bird` class has a `fly()` method in its interface. So following Liskov,
we should not use this inheritance, because we could not use `Penguin` everywhere
we used `Bird`.
 
![Liskov Meme](http://web.archive.org/web/20160505182607/https://lostechies.com/derickbailey/files/2011/03/LiskovSubtitutionPrinciple_52BB5162.jpg){: .center-image }

When is it then a good idea to use inheritance? One often encounter design, is to use it together
with virtual functions for runtime polymorphism. 
Following the [don't repeat yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle
we want to write code that does the same thing for similar objects, but we do not want
to write the same code for every single class again.

## Runtime Polymorphism
So let's assume we have a `PositionSensor` and a `AccelerationSensor` class. However
our program should detect during runtime how many of each are present and should
store them in a container. So a possible solution is to introduce a `Sensor` base
class and use this base class interface in the part of the code that deals with sensors
in a generic way.

```
struct Sensor {
  virtual double measure_value() = 0;
  virtual ~Sensor() = default;
};

struct AccelerationSensor : Sensor {
  double measure_value() override { ... }
};

struct PositionSensor : Sensor {
  double measure_value() override { ... }
};

std::vector<std::unique_ptr<Sensor>> detect(Hardware* hw){
  std::vector<std::unique_ptr<Sensor>> sensors;
  for (size_t i = 0; i < hw->num_sensors(); ++i) {
    auto type = hw->get_next_sensor_type();
    switch (type):
    case SensorType::Position:
      sensors.emplace_back(std::make_unique<PositionSensor>());
      break;
    case SensorType::Acceleration:
      sensors.emplace_back(std::make_unique<AccelerationSensor>());
      break;
    default:
      throw std::runtime_error("Sensor type not supported");
  }
  return sensors;
}

...
GUI::update_sensor_values() {
  for (size_t i = 0; i < sensors_.size(); ++i) {
    display_values_.at(i) = sensors_.at(i)->measure_value();
  }
}

```

The example above demonstrates a very basic use of inheritance to achieve runtime polymorphism through
virtual function calls. 

