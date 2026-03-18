# Introduction

This tutorial will try to show how you can use VILLAScontroller to communicate with a VILLASnode process. This document will hopefully not be too technical. For a technical documentation please have a look at [with_code_explanation](with_code_explanation/readme.md). 

The goal of this document is to provide a quick entrypoint for someone who is new to the VILLASframework, or a reference point for someone who is already familiar with the VILLASframework. The [VILLASframework](https://github.com/VILLASframework) itself consists of more parts than just [VILLAScontroller](https://github.com/VILLASframework/controller) and [VILLASnode](https://github.com/VILLASframework/node). However in this document we will solely focus on VILLAScontroller and a little bit of VILLASnode. At least for now ...

If you encounter any problems following this tutorial, please feel free to open an Issue. If you already have a solution please feel free to open a Pull Request. I'm more than happy about contributions, so, feel free to open Pull Requests or Issues.

# Installation

**Assumption:** We are on a Linux Distribution (specifically Arch Linux, but hopefully this should also work on other Linux Distributions).

Let's kick things off by installing VILLAScontroller.

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
3. Personally, I installed the `nix` packet manager (yes, on Arch Linux) and installed VILLASnode via [nix.flake](https://github.com/VILLASframework/node/blob/master/flake.nix). I would highly recommend this since I encountered the least trouble this way. The official [VILLASnode documentation](https://github.com/VILLASframework/node/tree/master/packaging/nix) and the instructions on the [official repository](https://github.com/VILLASframework/node/tree/master/packaging/nix) are quite helpful (you definitely don't need to read all of it).

   I also managed to simply compile VILLASnode on my machine. However, I would leave this part as a **work in progress** for now as I did encounter some issues and had to open one Pull Request on the official repository.

   Since we are on Arch Linux, we can also simply use the [PKGBUILD](https://github.com/VILLASframework/node/tree/master/packaging/archlinux). However, I do remember encountering issues here as well. I did manage to install it via PKGBUILD however I had to make a Pull Request on the official repository and I don't remember all the quirks at this moment. Hence I would also like to leave this as a **work in progress** for now.

   I'd be more than happy if you can help me out with the **work in progress** parts, but no worries since I went through the troubles once it shouldn't be too difficult to go through them again and document them this time.
6. Check if installation was successful:
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

This is why a RabbitMQ service needs to be already running. A quick look into the [official documentation](https://villas.fein-aachen.org/docs/controller/usage) shows us how we can run RabbitMQ inside a Docker Container.
```bash
docker run -p 5672:5672 -p 15672:15672 -d rabbitmq:management
```

In order to give `villas-controller` the `broker_url` we can use two methods:

### Method 1

Use a config file. E.g., simply create a new file called `config.json`:
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

The Subcommand is `daemon` because at the end of the day `villas-controller` is supposed to run as a daemon.

If everything works out you should see an output like this:
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

### Method 2

Use the command line and type:
```bash
villas-controller -b "amqp://guest:guest@localhost/%2F" daemon
```
   
> [!WARNING]
> However, this method crashes.
> ```bash
>     comps = [c for c in self.config.components if c.enabled]
>                    ^^^^^^^^^^^^^^^^^^^^^^
> TypeError: 'NoneType' object is not iterable
> ```

As you can see, you can freely define the `broker_url`. It just needs to direct to a running RabbitMQ service. Now, let's have a look at the so called *Components*.

# Components

As mentioned earlier, I will spare you the technical details. However, you can have a look at [with_code_explanation](with_code_explanation/readme.md) to see how you can derive which *key-value* pairs are valid to use in the config file.

Before we start, I would like to mention that pretty much everything in VILLAScontroller is a *Component* (exceptions exist of course). You have to tell `villas-controller` via the config file on startup which components with which settings you want to have.

```json
{
    "broker": {
        "url" : "amqp://guest:guest@localhost/%2F"
    },
    "components": [
        {
            Component 1 and its settings,
        },
        {
            Component 2 and its settings,
        },
        {
            ...
        },
        {
            Component i and its settings,
        },
        {
            ...
        },
        {
            Component n and its settings
        }
    ]
}
```

Eventhough the following UML diagram belongs to the more detailed [with_code_explanation](with_code_explanation/readme.md), I think it still helps in understanding how each Component is build. 
![villas-controller-uml](villas-controller-uml.svg)

A better, sleaker version of the same diagram can be found in the [official documentation](https://villas.fein-aachen.org/docs/controller/protocol).

VILLAScontroller first decides to which *category* a Component belongs. For that it looks at its field `category`. Let's keep our example going with only one Component.

```json
{
    "broker": {
        "url" : "amqp://guest:guest@localhost/%2F"
    },
    "components": [
        {
            "category": Is this a "simulator", a "manager" or a "gateway"
        }
    ]
}
```

Since we want to control a `villas-node` process we will go for the `manager` *category*. 

```json
{
    "broker": {
        "url" : "amqp://guest:guest@localhost/%2F"
    },
    "components": [
        {
            "category": "manager"
        }
    ]
}
```

Why? we will find this out in the next couple lines. Main reason is, because after looking at the `category` VILLAScontroller looks at the field `type`.

```json
{
    "broker": {
        "url" : "amqp://guest:guest@localhost/%2F"
    },
    "components": [
        {
            "category": "manager",
            "type": Is this a "generic", a "kubernetes", a "villas-node", a "villas-relay" or a "miob" manager
        }
    ]
}
```

We will set this one to `villas-node` (if you look at the Code and go to the definition of the next call you will immmediately see why, hint: Look at `Manager.from_dict()` method inside `villas/controller/components/manager.py`)



The *Components* have a hierarchical structure. The first *layer* defines the `category`. A *Component* can be of category 

Once we analyze the code we will realize that the only *Component* that can communicate with a `villas-node` process is the `VILLASnodeManager`. This means, if we want our VILLAScontroller, to give commands to a `villas-node` process we have to tell our VILLAScontroller on startup (via the config file) that we want at least one `VILLASnodeManager`.

```json
{
    "broker": {
        "url" : "amqp://guest:guest@localhost/%2F"
    },
    "components": 
}
```


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
