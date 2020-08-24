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
for runtime and compile time polymorphism.

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

When is it then a good idea to use inheritance? Often it is to used together
with virtual functions for runtime polymorphism. 
Following the [don't repeat yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle
we want to write code that does the same thing for similar objects, but we do not want
to write the same code for every single class again.

## Runtime Polymorphism
So let's assume we have a `PositionSensor` and a `AccelerationSensor` class. However
our program should detect during runtime how many of each are present and should
store them in a container. So a possible solution is to introduce a `Sensor` base
class and use this base class' interface in the part of the code that deals with sensors
in a generic way.

```cpp
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

GUI::update_sensor_values(std::vector<std::unique_ptr<Sensor>>& sensors) {
  for (size_t i = 0; i < sensors.size(); ++i) {
    display_values_.at(i) = sensors.at(i)->measure_value();
  }
}

```
And then use it like this:
```cpp
Hardware hw = load_HW_from_configuration_file();
auto sensors = detect(hw);
GUI gui;
gui.update_sensor_values(sensors);

```

The example above demonstrates a very basic use of inheritance to achieve runtime
polymorphism through virtual function calls. 

## Compile time Polymorphism

In some cases you want to write generic code, but you already know your polymorphic
types at compile time and you don't want to pay the runtime cost of virtual
function calls. One way of achieving this in C++ is called the Curriously Recurring
Template Pattern (CRTP) idiom. 
It's main idea is to use the derived class as a template parameter of the base
class.

```cpp
template <typename T>
struct Base {
...
};


struct Derived : Base<Derived> {
...
};

```
If you see this for the first time it might look a bit weird. The `Derived` class
inherits from the `Base` class with itself as a template parameter. But this way
we have the possiblity to access the methods of the derived class in the base class.
And if we name the method of the derived class the same we can overwrite the base
class method.
But let's just change above example to compile time polymorphism to better see how
this works:

```cpp
template <typename T>
struct Sensor {
  double measure_value() {
    T& derived = static_cast<T&>(*this);
    return derived.measure_value();
  } 
};

struct AccelerationSensor : Sensor<AccelerationSensor> {
  double measure_value() { ... }
};

struct PositionSensor : Sensor<PositionSensor> {
  double measure_value() { ... }
};

...

template <typename Derived>
GUI::update_sensor_value(int i, Sensor<Derived>& sensor) {
  display_values_.at(i) = sensor.measure_value();
}

```

And then use it like this:
```cpp
AccelerationSensor sensor0{};
PositionSensor sensor1{};
GUI gui;
gui.update_sensor_value(0, sensor0);
gui.update_sensor_value(1, sensor1);

```

Now this example is of course a bit oversimplified and you could achieve the same,
with a simple templated `update_sensor_value` function. But as soon as you have more
logic built into your classes you will see that the CRTP is a good way to write
generic code for types you already know at compile time.

As a final note: Remember that inheritance is not the only way to achieve
polymorphism and consider other choices as well e.g. using `std::variant`.

## References 
* Hands-On Design Patterns with C++ by Fedor G. Pikus (ISBN 978-1-78883-256-4)
* [Fluent C++](https://www.fluentcpp.com/2017/05/12/curiously-recurring-template-pattern/)







