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
> [!INFO]
> We will always run VILLAScontroller as a daemon, hence we will be using the deamon SUBCOMMAND (check villas-controller -h)

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
