---
layout: post
title:  "C++ Design Patterns: Low effort observers"
thumbnail: assets/images/cpp-logo.svg
categories: [cpp, design patterns]
---
Another classic Gang of Four Pattern is the 
[Observer Pattern](https://en.wikipedia.org/wiki/Observer_pattern). In this
pattern, *observers* want to be notified about state changes of a *subject*.
In this post we will look how to easily implement this with `std::function`.

As before I will stick to a sensor example. Let's assume we have a sensor
that changes its state in unpredictable intervals and different parts of
your system need to know about these changes. Of course you could pull
the sensor state form each part. However this is not very elegant and might
lead to several unneeded busy loops.
In the classic pattern the *subject* would keep a list of the *observers*
but since C++11 we can register callbacks very easily with `std::function`.
For simplicity we will assume the sensor state is represented by an `int`.

# Low effort observers

```cpp
#include <functional>
#include <list>

class Sensor {

std::list<std::function<void(int)>> callbacks_{};

public:

void attach(std::function<void(int)> callback) {
  callbacks_.emplace_back(callback);
}

void measure() {
  // some complex measurment logic which is waiting for hardware state changes...
  const int new_sensor_state = 42;
  notify(new_sensor_state);
}

private:

void notify(int state) {
  for (const auto& callback : callbacks_) {
      callback(state);
  }
}

};

```

The usage would then be like this:

```cpp
Sensor sensor;
sensor.attach([](int state) {
  std::cout << "New sensor state: " << state << std::endl;
});

```

That's it. Super simple and implemented within minutes. Most interestingly
there is no observer class in this observer pattern.

But you might have realized that there is one small catch. There is no `detach`
method. Once we registered an observer with `attach` there is no way to deregister.
That is due to the fact, that `std::function` is not comparable (or only against `nullptr`).
In many cases this is fine and you want to observer a state during the whole runtime.
However if you need to be able to unregister there is an easy fix for this.

# Observers with handles
When attaching a callback to our sensor we can just return a handle. When we then
want to unregister from notifications about the state changes of the sensor we 
pass this handle to the `detach` method.

Change the `Sensor::attach` method to return a handle to the inserted callback:
```cpp

std::list<std::function<void(int)>>::iterator attach(std::function<void(int)> callback) {
  callbacks_.emplace_back(callback);
  return --callbacks_.end();
}

```

And add a `Sensor::detach` method:
```cpp

std::list<std::function<void(int)>>::iterator attach(std::function<void(int)> callback) {
  callbacks_.emplace_back(callback);
  return --callbacks_.end();
}

```

And use it like this:
```cpp
Sensor sensor;
auto handle = sensor.attach([](int state) {
  std::cout << "New sensor state: " << state << std::endl;
});

/// Do some stuff...

sensor.detach(handle);

```

Unfortunately there is no free lunch. We now placed a burden on the user to
keep track of the handles. But as long as `std::function` is not comparable
there is no easy workaround for this. If keeping track of handles is not
acceptable you might at this point be better off not reinventing the wheel
and use an existing library e.g.
[boost::signals2](https://www.boost.org/doc/libs/1_74_0/doc/html/signals2.html#id-1.3.36.3.5)
or [Qt signals](https://doc.qt.io/qt-5/signalsandslots.html)

## References 
* [Generalizing Observer - Herb Sutter](https://www.drdobbs.com/cpp/generalizing-observer/184403873)

