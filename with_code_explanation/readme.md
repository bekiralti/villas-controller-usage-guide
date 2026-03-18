# Summary

This guide will try to show how to use VILLAScontroller and also try to explain it by using according Code Snippets. Let's start with the installation first.

# Installation

**Assuming** we are on a Linux Distribution (specifically Arch Linux, but hopefully this should also work on other Linux Distributions, if not feel free to write an Issue or if you have the solution create a Pull Request). 

This section will focus on installing `villas-controller` and `villas-node`.

## VILLAScontroller

1. Clone the VILLAScontroller repository:
   ```bash
   git clone https://github.com/VILLASframework/controller
   ```
2. Change into the VILLAScontroller directory:
   ```bash
   cd controller
   ```
3. Create a `venv`:
   ```bash
   python -m venv .venv
   ```
4. *Activate* the `venv`:
   ```bash
   source .venv/bin/activate
   ``` 
4. Install `villas-controller`:
   ```bash
   pip install .
   ```
   Or if you want to edit the source code and see the changes on the fly:
   ```bash
   pip install -e .
   ```
5. Check if we were successful:
   ```bash
   villas-controller --version
   ```
   The output should be something like:
   ```
   0.4.1
   ```

Next, we will install `villas-node`.

## VILLASnode

1. Clone the VILLASnode repository:
   ```bash
   git clone https://github.com/VILLASframework/node
   ```
2. Change into the VILLASnode directory:
   ```bash
   cd node
   ```
3. **WORK IN PROGRESS** At this moment in time I would redirect you to the [VILLASnode repository](https://github.com/VILLASframework/node/tree/master/packaging/nix), specifically using the `nix` packet manager to install VILLASnode on your system. I will try to write out this step in future, also describing how to actually compile VILLASnode (not using any packet managers whatsoever).
4. Check if installation was successful:
   ```bash
   villas-node -V
   ```
   The output should be something like:
   ```bash
   09:06:06 info node             This is VILLASnode 1.0.1 (built on Jan  1 1980, 00:00:00)
   09:06:06 info signals          Initialize subsystem
   1.0.1
   ```

# Explanation and Usage

Let's start cooking.

## RabbitMQ

VILLASController uses RabbitMQ to communicate. Pretty much one of the first things the VILLASController does (besides parsing the arguments yada yada) is to connect to a RabbitMQ broker.
```python
# villas/controller/main.py

with kombu.Connection(broker_url, connect_timeout=3) as c:
```

This is why a RabbitMQ service needs to be already running. We can simply use the [VILLAScontroller documentation](https://villas.fein-aachen.org/docs/controller/usage) and run RabbitMQ inside a Docker Container.
```bash
docker run -p 5672:5672 -p 15672:15672 -d rabbitmq:management
```

In order to give `villas-controller` the `broker_url` we can use two methods:
> [!TIP]
> We will always run VILLAScontroller as a daemon, hence we will be using the `deamon` SUBCOMMAND (check `villas-controller -h`) through this entire guide.

1. Use a config file. E.g., simply create a new file called `config.json`:
   ```json
   {
       "broker": {
           "url" : "amqp://guest:guest@localhost/%2F"
       }
   }
   ```
   Run VILLAScontroller with the config file:
   ```python
   villas-controller -c config.json daemon
   ```
2. Use the command line and type:
   ```bash
   villas-controller -b "amqp://guest:guest@localhost/%2F" daemon
   ```
   
> [!WARNING]
> The second option crashes.
> ```bash
>     comps = [c for c in self.config.components if c.enabled]
>                    ^^^^^^^^^^^^^^^^^^^^^^
> TypeError: 'NoneType' object is not iterable
> ```

If everything works, you should see an output like:

```bash
2026-03-18 11:18:51 | INFO | villas.controller | Connecting to: amqp://guest:guest@localhost/%2F
2026-03-18 11:18:51 | INFO | villas.controller.controller | Starting mixing for 0 components
2026-03-18 11:18:51 | INFO | kombu.mixins | Connected to amqp://guest:**@127.0.0.1:5672//
2026-03-18 11:18:51 | INFO | villas.controller.api | Starting API at http://localhost:8089/api/v1
2026-03-18 11:18:51 | INFO | villas.controller.controller | Components changed. Restarting mixin
2026-03-18 11:18:51 | INFO | villas.controller.controller | Adding generic manager <Generic Manager: 32018e4e-5519-4633-86ef-59dceb29fb27>
2026-03-18 11:18:51 | INFO | villas.controller.manager.generic.32018e4e-5519-4633-86ef-59dceb29fb27 | Start state publish thread
2026-03-18 11:18:52 | INFO | villas.controller.controller | Starting mixing for 1 components
2026-03-18 11:18:52 | INFO | kombu.mixins | Connected to amqp://guest:**@127.0.0.1:5672//
```

> [!TIP]
> Don't worry about the reoccuring (every 30 seconds) messages, these are just some kind of health checks.

As you can see, you can freely define the `broker_url`. Now, let's have a look at the so called *Components*.

# Components

VILLAScontroller basically consists of two parts,
- A so called `ControllerMixin` instance:
  ```python
  # villas/controller/controller.py
  
  class ControllerMixin(kombu.mixins.ConsumerProducerMixin):
  ```
  Which gets created and started by the `daemon` subcommand:
  ```python
  # villas/controller/commands/daemon.py

  d = ControllerMixin(connection, args)
  d.start()
  ```
  The method call `d.start()` starts the kombu Event-Loop. This Event-Loop is responsible for *sending and receiving* messages all the time. VILLAScontroller basically runs in this Event-Loop.
- Components:
  ```python
  # villas/controller/component.py

  class Component:
  ```
  Pretty much everything (exceptions exist) in VILLASController is a *Component*. If you look up which classes inherit from `Component` you will end up with a hierarchy such as this (it might take some time for the SVG to load:
  ![villas-controller-uml](villas-controller-uml.svg)
  The same diagram can also be found on the [official documentation](https://villas.fein-aachen.org/docs/controller/protocol).

  Components, e.g. `DummySimulator`, `VILLASnodeManager`, etc. are created through the config file. Inside the config file you can define which Components you want or need and also give them properties. For now, let's focus on `VILLASnodeManager` since this class is the one that can *speak* to a villas-node process.

## VILLASnodeManager

First of all, let's see how we can create a `VILLASnodeManager` component. If you take a look at `ControllerMixin.__init__()` you will notice the lines
```python
# villas/controller/controller.py
self.config: Config = args.config


comps = [c for c in self.config.components if c.enabled]
for comp in comps:
    LOGGER.info('Adding %s', comp)
    manager.add_component(comp)
```

Since `self.config` is an instance of the class `Config` let's take a look at what is defined under `sellf.config.components`:
```python
# villas/controller/config.py

@property
def components(self) -> List[Component]:
    return [Component.from_dict(c) for c in self.config.components]
```

> [!TIP]
> Yes, it has a bit of a name mangling going on. However the `self.config.components` in the `Config.comoponents()` method leads to nowhere and Python automatically calls the further defined method `Config.__getattr__()` method:
> ```python
> # villas/controller/config.py
> 
> def __getattr__(self, attr):
>     return self.config.get(attr)
> ```

# Controller Mixin
**WORK IN PROGRESS**
