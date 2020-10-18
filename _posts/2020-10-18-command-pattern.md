---
layout: post
title:  "C++ Design Patterns: A Modern Command Pattern"
thumbnail: assets/images/cpp-logo.svg
categories: [cpp, design patterns]
---
Don't worry. This is not yet another take at the classic Gang of Four
[Command Pattern](https://en.wikipedia.org/wiki/Command_pattern#C++).
Instead we look at how we can use modern C++ features to solve
the same problem in a different way. Namely we want to send commands to a (possibly)
remote application. Whilst choosing a testable and maintainable design.

Let's best have a look at an example to illustrate the task at hand.
We will control a remote light bulb.

```cpp
// our mock hardware
struct Lightbulb{
  Lightbulb(std::string name) : name_(std::move(name)){}

  void set_brightness(unsigned val) {
    std::cout << "Lightbulb(" << name_ << ") set brightness to " << val << '\n';
  }

  void set_color(unsigned r, unsigned g, unsigned b) {
    std::cout << "Lightbulb(" << name_ << ") set color to RGB("
              << r << ',' << g << ',' << b << ")\n";
  }
  std::string name_ = "";
};
```

From the perspective of the software which will control our light bulb,
we need to do the following things:
1. Receive the command from a communication interface
2. Deserialize the command
3. Pass the command on to the hardware (i.e. call a function of our hardware class)

One could of course just hardwire the commands to directly call the `Lightbulb` methods when
deserializing them. However this would introduce a very strong coupling between the communication
and the "business" logic. That would neither be easily testable nor very maintainable.

# The Commands

Since C++17 we have the visitor pattern built into the STL with `std::variant` and `std::visit`.
Which in my opinion is a great way how to address the above problem.

So we will define structs/classes for the commands we want to send.
```cpp

namespace cmd {

struct SetBrightness {
  unsigned val = 0;
};

struct SetColor {
  unsigned r = 0;
  unsigned g = 0;
  unsigned b = 0;
};

using Command = std::variant<SetBrightness, SetColor>;

} // namespace cmd

```

An instance of `std::variant` holds one of its template types. So it is a great
way to store unrelated types like our command structs. 

We then only need a visitable object (we need an action for every type the variant can hold):
```cpp

namespace cmd {

struct CommandExecutor{
  CommandExecutor(Lightbulb& bulb) : bulb_(bulb){};

  void operator()(const SetBrightness& cmd){
    bulb_.set_brightness(cmd.val);
  }

  void operator()(const SetColor& cmd){
    bulb_.set_color(cmd.r, cmd.g, cmd.b);
  }

private:
  Lightbulb& bulb_;
};

} // namespace cmd

```

And that is basically it.

* Define simple POD structs that represent your commands and use `std::variant` to pass them around.
* Define a visitor object that performs different actions based on the actual value of the `std::variant`

To give a more complete example we will also look at the communication and serialization steps mentioned above.

# Deserialize it!

On the communication interface (which we will define bellow) we will receive the commands in a certain format / protocol and we need
to parse these messages into our command representation above. For this example I will use JSON. JSON is in my opinion a very
good starting point for machine to machine communication, because it is human readable and hence easy to debug. Most of the time its performance is also good enough for sending small data like the here mentioned commands.

I decided to use [nlohmann/json](https://github.com/nlohmann/json) for this example:

{% raw %}
```cpp
#include <nlohmann/json.hpp>

namespace cmd {

inline
void to_json(json& j, const SetBrightness& cmd) {
  j = json{{"brightness", cmd.val}};
}

inline
void from_json(const json& j, SetBrightness& cmd) {
  j.at("brightness").get_to(cmd.val);
}

inline
void to_json(json& j, const SetColor& cmd) {
  j = json{{"red", cmd.r},{"green", cmd.g},{"blue", cmd.b}};
}

inline
void from_json(const json& j, SetColor& cmd) {
  j.at("red").get_to(cmd.r);
  j.at("green").get_to(cmd.g);
  j.at("blue").get_to(cmd.b);
}

inline
Command deserialize(const json& j) {
  Command ret;
  const auto type = j.at("command_type").get<std::string>();
  if (type == "set_brightness") {
    ret = j.at("command_arguments").get<SetBrightness>();
  } else if (type == "set_color") {
    ret = j.at("command_arguments").get<SetColor>();
  } else {
    throw std::runtime_error("Could not deserialize json command " + j.dump());
  }
  return ret;
}

} // namespace cmd

```
{% endraw %}
 
# Communication

Like for the serialization part above, there exist many possibilities how to perform communication between your applications. For the application here I chose to use websockets, since they are easy to use for both local and remote communication.

Here we will be using [boost's websocket](https://www.boost.org/doc/libs/1_70_0/libs/beast/doc/html/beast/using_websocket.html) implementation, which is maybe a bit verbose, but available for almost any decent OS.

At some point we will also need some thread safe mechanism to pass the commands from the communication thread to our main thread.
For simplicity's sake I will just use a `boost::lockfree::queue`, since anyway we are using boost. A thread safe queue is a great way to deal with communication between threads, because from the point of the main thread there will only be a single source of commands. This will make this part of the system easier to test.

```cpp
#include <boost/lockfree/queue.hpp>

namespace cmd {

# for convenience
using CommandQueue = boost::lockfree::queue<cmd::Command>;

} // namespace cmd
```

Our light bulb application will be a websocket server, where clients can connect to. So we define a listener class, which listens for new connections and starts a new session for each (we will define later what a session does).
```cpp

#include "commands.hpp"

#include <boost/asio/bind_executor.hpp>
#include <boost/asio/ip/tcp.hpp>
#include <boost/asio/strand.hpp>
#include <boost/beast/core.hpp>
#include <boost/beast/websocket.hpp>

namespace beast = boost::beast;         // from <boost/beast.hpp>
namespace websocket = beast::websocket; // from <boost/beast/websocket.hpp>
namespace net = boost::asio;            // from <boost/asio.hpp>
using tcp = net::ip::tcp;               // from <boost/asio/ip/tcp.hpp>

// Report a failure
void fail(beast::error_code ec, char const *what) {
  std::cout << what << ": " << ec.message() << "\n";
}

// Accepts incoming connections and launches the Sessions
class Listener : public std::enable_shared_from_this<Listener> {
  net::io_context &ioc_;
  tcp::acceptor acceptor_;
  cmd::CommandQueue& cmd_queue_;

public:
  Listener(net::io_context &ioc, tcp::endpoint endpoint, cmd::CommandQueue& cmd_queue)
      : ioc_(ioc), acceptor_(ioc), cmd_queue_(cmd_queue) {
    beast::error_code ec;

    // Open the acceptor
    acceptor_.open(endpoint.protocol(), ec);
    if (ec) {
      fail(ec, "open");
      return;
    }

    // Allow address reuse
    acceptor_.set_option(net::socket_base::reuse_address(true), ec);
    if (ec) {
      fail(ec, "set_option");
      return;
    }

    // Bind to the server address
    acceptor_.bind(endpoint, ec);
    if (ec) {
      fail(ec, "bind");
      return;
    }

    // Start listening for connections
    acceptor_.listen(net::socket_base::max_listen_connections, ec);
    if (ec) {
      fail(ec, "listen");
      return;
    }
  }

  // Start accepting incoming connections
  void run() { do_accept(); }

private:
  void do_accept() {
    // The new connection gets its own strand
    acceptor_.async_accept(
        net::make_strand(ioc_),
        beast::bind_front_handler(&Listener::on_accept, shared_from_this()));
  }

  void on_accept(beast::error_code ec, tcp::socket socket) {
    if (ec) {
      fail(ec, "accept");
    } else {
      // Create the Session and run it
      std::make_shared<Session>(std::move(socket), cmd_queue_)->run();
    }

    // Accept another connection
    do_accept();
  }
};

```

With a `Session` which just waits for new messages, deserializes them, puts them on our queue and resumes waiting for new messages.
```cpp
class Session : public std::enable_shared_from_this<Session> {
  websocket::stream<beast::tcp_stream> ws_;
  beast::flat_buffer buffer_;
  cmd::CommandQueue& cmd_queue_;

public:
  // Take ownership of the socket
  explicit Session(tcp::socket &&socket, cmd::CommandQueue& cmd_queue) : ws_(std::move(socket)), cmd_queue_(cmd_queue) {}

  // Start the asynchronous operation
  void run() {
    // Set suggested timeout settings for the websocket
    ws_.set_option(
        websocket::stream_base::timeout::suggested(beast::role_type::server));

    // Accept the websocket handshake
    ws_.async_accept(
        beast::bind_front_handler(&Session::on_accept, shared_from_this()));
  }

  void on_accept(beast::error_code ec) {
    if (ec)
      return fail(ec, "accept");

    // Read a message
    do_read();
  }

  void do_read() {
    // Read a message into our buffer
    ws_.async_read(buffer_, beast::bind_front_handler(&Session::on_read,
                                                      shared_from_this()));
  }

  void on_read(beast::error_code ec, std::size_t bytes_transferred) {
    boost::ignore_unused(bytes_transferred);

    // This indicates that the Session was closed
    if (ec == websocket::error::closed)
      return;

    if (ec)
      fail(ec, "read");

    auto data = reinterpret_cast<char*>(buffer_.data().data());
    const auto json_cmd = json::parse(data, data + buffer_.data().size());
    buffer_.consume(buffer_.size());
    const auto command = cmd::deserialize(json_cmd);
    cmd_queue_.push(command);

    do_read();
  }

  void on_write(beast::error_code ec, std::size_t bytes_transferred) {
    boost::ignore_unused(bytes_transferred);

    if (ec)
      return fail(ec, "write");

    // Clear the buffer
    buffer_.consume(buffer_.size());

    do_read();
  }
};
```

# Putting it all together
Now we got all the building blocks for our application. So let's write `main()`.

```cpp
#include "commands.hpp"
#include "websocket.hpp"

#include <chrono>
#include <thread>

#include <csignal>

void process_commands(cmd::CommandExecutor& executor, cmd::CommandQueue& cmd_queue){
  cmd::Command command;
  while (cmd_queue.pop(command)) {
    std::visit(executor, command);
  }
}

sig_atomic_t signaled = false;

void signal_handler(int signal){
  if ((SIGTERM == signal) or (SIGINT == signal)) {
    signaled = true;
  }
}

int main(){
  std::signal(SIGTERM, signal_handler);
  std::signal(SIGINT, signal_handler);

  Lightbulb bulb("LED");
  cmd::CommandExecutor executor(bulb);
  cmd::CommandQueue cmd_queue(100);

  net::io_context io_context(1);
  
  // listen on all IPv4 interfaces on port 8888
  std::make_shared<Listener>(io_context, tcp::endpoint{tcp::v4(), 8888}, cmd_queue)->run();
  std::thread io_task([&io_context](){ io_context.run(); });
	
  // our main loop
  while (not signaled) {
    const auto now = std::chrono::steady_clock::now();
    process_commands(executor, cmd_queue);
    // some other tasks...
    std::this_thread::sleep_until(now + std::chrono::milliseconds(500));
  }

  std::cout << "=== THE END ===\n";
  io_context.stop();
  if (io_task.joinable()) {
    io_task.join();
  }
}

```

And that's it. From the websocket client we can then send our commands as json strings e.g.:
```
{
  "command_type": "set_color",
  "command_arguments": { "red": 11, "green": 22, "blue": 33 }
}
```

Some take away points:
* `std::variant` and `std::visit` are great alternatives to an inheritance based command design.
* Separating the communication from the hardware control makes it easy to maintain e.g. replacing the communication interface or protocol.
* Having a single source of commands (here our command queue) makes it very suitable for testing
