---
layout: post
title:  "Back to basics: Simple polymorphism in C"
thumbnail: assets/images/cpp-logo.svg
categories: [c, design patterns]
---
TODO stuck with C, virtual function calls...

# The "base class"

```c
#include <stddef.h>
#include <stdint.h>
#include <string.h>

typedef enum {
  FAILURE = -1,
  SUCCESS = 0,
} Return_codes ;

typedef struct _Memory_device Memory_device;
typedef struct _Memory_device_ops Memory_device_ops;

#define MEMORY_DEVICE(obj) ((Memory_device *)obj)

struct _Memory_device {
  Memory_device_ops* ops;
  char name[32];
};

struct _Memory_device_ops {
  int (*read)(Memory_device* impl, uint32_t address, uint8_t* buffer, size_t size);
  int (*write)(Memory_device* impl, uint32_t address, const uint8_t* buffer, size_t size);
};
```

# Base class methods aka. your generic interface
```
// Base class implementations

inline const char* memory_device_get_name(Memory_device* self) {
  return self->name;
}

inline void memory_device_set_name(Memory_device* self, const char* name) {
  strncpy(self->name, name, 32);
}

// Base class "virtual" functions implemented by "derived" classes

inline int memory_device_read(Memory_device* self, uint32_t address, uint8_t* buffer, size_t size) {
  return self->ops->read(self, address, buffer, size);
}

inline int memory_device_write(Memory_device* self, uint32_t address, const uint8_t *buffer, size_t size) {
  return self->ops->write(self, address, buffer, size);
}
```

# A "derived class"
```

typedef struct _Flash_Memory_Device_AB123 Flash_Memory_Device_AB123;

struct _Flash_Memory_Device_AB123 {
  Memory_device parent;
  ABC123Hardware hardware;
}

static int AB123_read(Memory_device* baseclassptr, uint32_t address, uint8_t* buffer, size_t size) {
  int ret;
  Flash_Memory_Device_AB123* self = (Flash_Memory_Device_AB123*) baseclassptr;
  self->hardware.some_magic(address, buffer, size);
  ... /// do actual work
  return ret;
}

static int AB123_write(Memory_device* baseclassptr, uint32_t address, const uint8_t* buffer, size_t size) {
  int ret;
  Flash_Memory_Device_AB123* self = (Flash_Memory_Device_AB123*) baseclassptr;
  self->hardware.some_magic(address, buffer, size);
  ... /// do actual work
  return ret;
}

static Memory_device_ops AB123_ops = {
    .read = AB123_read,
    .write = AB123_write,
};

// Derived class ctor
static void AB123_init(Flash_Memory_Device_AB123* self, const char* name) {
  self->parent.ops = &AB123_ops;
  memory_device_set_name(&self->parent, name);
  ABC123HardwareInit(self->hardware);
}

```
Why? -> generic function

Usage:
```

// some generic function you want to implement for several memory devices
uint16_t load_configuration(Memory_Device* device) {
  uint16_t ret = 0;
  uint8_t buffer[2];
  if (SUCCESS == device->read(0xA000, buffer, 2)) {
    ret = buffer[1];
    ret |= buffer[0] << 1;
  }
}

int main() {
  Flash_Memory_Device_AB123 persitent_settings;
  AB123_init(&permanent_settings, "Persistant_Flash");
  uint16_t configuration = load_configuration(MEMORY_DEVICE(&permanent_settings));
  return 0;
}

```

Bla bla
