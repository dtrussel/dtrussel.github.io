---
layout: post
title:  "C++ Design Patterns: Template Method - no templates involved"
thumbnail: assets/images/cpp-logo.svg
categories: [cpp, design patterns]
---
Assume you have something that has an overall structure, but some parts of it need to
be customized depending on the use case. The idea of the Template Method pattern is
to define the overall structure in a base class and let the derived classes override
the specific behavior.

Going back to the sensor example from my last post:
```cpp
struct Sensor {
  double measure(){ return do_measure(); }
  virtual ~Sensor() = default;
private:
  virtual double do_measure() = 0;
};

struct AccelerationSensor : Sensor {
private:
  double do_measure() override { ... }
};

struct PositionSensor : Sensor {
private:
  double do_measure() override { ... }
};

```

This is great because the interface `measure()` is separated from the implementation `do_measure()`.
If we would simply make the `measure()` method virtual and override it in the base class, the
implementation and interface would be coupled and it would make it harder to maintain such classes.
If we want later to add some functionality it will be much easier with the template method / a non-virtual
interface. E.g. we want to filter each sensor value.
Then we can just modify the base class like this:
```cpp
struct Sensor {
  double measure(){
    const auto val = do_measure();
    return filter(val);
  }
  virtual ~Sensor() = default;
private:
  virtual double do_measure() = 0;
  double filter(double value) { /* some filter implementation */ }
};
```
The interface `measure()` stays exactly the same and none of the client code has to be modified.
This would not have been possible, had we worked with a overloading a virtual interface.

So make your interface non-virtual and public in the base class and separate the implementation into private/protected
virtual functions which the derived classes can override.

## References 
* Hands-On Design Patterns with C++ by Fedor G. Pikus (ISBN 978-1-78883-256-4)
* [Virtuality - Herb Sutter](http://www.gotw.ca/publications/mill18.htm)



