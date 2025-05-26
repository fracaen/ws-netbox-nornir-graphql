# Workbook 2: Nornir

## Objectives

The objective of this workbook is to:
1. Understand how to install the Nornir and NetBox Inventory Plugin
2. Understand how to validate the NetBox Inventory Plugin
3. Understand how to filter the inventory
4. Understand how to run prebuilt tasks via plugins
5. Understand how to run custom tasks
6. Understand how to inspect the Nornir Result Object


## Exercise 1 - Install Nornir and NetBox Inventory Plugin
The lab has Nornir installed and a NetBox Inventory Plugin configured.
This was acheived via a:

```
uv pip install nornir nornir-netbox
```

You can also view the Nornir configuration file, containing the NetBox Inventory Plugin configuration, in the `demo` directory. Within the config file, you will find the following configuration:  

```python
nr = InitNornir(
    runner={"plugin": "threaded", "options": {"num_workers": 20}},
    inventory={
        "plugin": "NetBoxInventory2",
        "options": {
            "nb_url": os.getenv("NETBOX_FQDN"), # Read the NETBOX_FQDN environment variable
            "nb_token": os.getenv("NETBOX_TOKEN"), # Read the NETBOX_TOKEN environment variable
            "filter_parameters": {}, # Filter the devices returned from NetBox
            "use_platform_slug": True, # Use the platform slug as the device type
        },
    },
)
```

> ![NOTE]
> Note how the configuration is provided to `InitNornir`. Meaning the configuration will be used to initialize the Nornir object, at the point we reference `nr`.


## Exercise 2 – Inventory: Validate NetBox Inventory Plugin
We now need to validate that the Nornir inventory plugin is working correctly.

1. Run `task lab` to start the ipython shell.
```bash
task lab
```

2. Print the inventory hosts.

```python
print(nr.inventory.hosts)
```

Within each of these objects, will be the various attributes from the Device object pulled from the NetBox Device model.

3. Print the inventory data for a single host.

```python
print(nr.inventory.hosts["leaf1"])
``` 

From the output you would of seen that the inventory plugin has retrieved the device information from NetBox and stored it in the Nornir inventory.

Now we have inventory data, the next step is to filter the inventory to define "who" we want to run our tasks against.

## Exercise 3 – Inventory: Filtering
We will now work with Nornir filtering. There are 2 types of filtering basic and advanced. Lets dive straight into the advanced filtering.
This will be achieved by using something called the "F Object".

1. Run `task lab`, to start the ipython shell.
```bash
task lab
```

2. Filter the inventory for a specific host.
```python
nr = nr.filter(F(name="leaf1"))
print(nr.inventory.hosts)
```

3. Filter the inventory for a specific tenant.
```python
from demo.config import nr # re-initialize nr

nr = nr.filter(F(tenant__name="pod1"))
print(nr.inventory.hosts)
```

> [!NOTE]
> Note how we use `__` to look up a child key. This is because `name` is a child of `tenant`.
> Also note how we can chain filters together. Like so:
```python
nr.filter(F(tenant__name="pod1") & F(name="leaf1"))
```

## Exercise 4 – Tasks: Show OSPF with Nornir and Netmiko
Now we will use the Netmiko plugin to run a command against a device.

> [!NOTE]
> In this exercise we do not have to create a task as we can use a built-in task from the Nornir Netmiko plugin.

1. Run `task lab`, to start the ipython shell.
```bash
task lab
```

2. Show Routing Table on all devices in tenant "pod1".
```python
from nornir_netmiko.tasks import netmiko_send_command
from nornir_utils.plugins.functions import print_result

nr = nr.filter(F(tenant__name="pod1"))

result = nr.run(task=netmiko_send_command, command_string="show ip route")
print_result(result)
```

To breakdown the command:
- `nr.run`: Run a task against the inventory.
- `task=netmiko_send_command`: Use the Netmiko plugin.
- `command="show ip route"`: Run the command `show ip route`.
- `print_result(result)`: Print the result.


## Exercise 5 – Results: Inspect the Nornir Result Object
We printed out the result. But what is actually inside the result object?
We will now use Nornir Inspect to inspect the result object.

Whilst staying in the same shell, perform the following:

1. Show the type of `result`:
```python
type(result)
# Output:
# nornir.core.task.AggregatedResult
```

2. Show the result object at a high level. In other words, print the object relationships.

```python
nornir_inspect(result, vals=False)
# Output:
# <class 'nornir.core.task.AggregatedResult'>
# ├── <class 'nornir.core.task.MultiResult'> ['leaf11']
# │   └── <class 'nornir.core.task.Result'> [0]
# ├── <class 'nornir.core.task.MultiResult'> ['leaf12']
# │   └── <class 'nornir.core.task.Result'> [0]
# ├── <class 'nornir.core.task.MultiResult'> ['leaf13']
# ...
```

3. Show the result object at a low level. In other words, we will now also print the attributes of the objects.

```python
nornir_inspect(result)
# Output:
# <class 'nornir.core.task.AggregatedResult'>
# ├── failed = False
# ├── failed_hosts = {}
# ├── name = netmiko_send_command
# ├── <class 'nornir.core.task.MultiResult'> ['leaf11']
# │   ├── failed = False
# │   ├── failed_hosts = {}
# │   ├── name = netmiko_send_command
# │   └── <class 'nornir.core.task.Result'> [0]
# │       ├── changed = False
# │       ├── diff = 
# │       ├── exception = None
# ...
```

Understanding this low level view of the result object is important as it will help us to navigate the result object when we want to extract data from it. For example place data into a DataFrame, or use the data for network testing.

## Exercise 6 – Tasks: Render Jinja2 Template with Nornir
We will now leverge the template_file task from the Nornir Jinja2 plugin to render a Jinja2 template.

1. Run `task lab`, to start the ipython shell.
```bash
task lab
```

2. We have created a basic Jinja template (`vlan.j2`), that will render a list of vlans. You can view the template in the `demo/templates` directory.

3. Render a Jinja2 template. 
Note: the `template_file` plugin task is used to render the template. Also note that we pass the variables to the plugin. These can be provided as kwargs. 

```python
from nornir_jinja2.plugins.tasks import template_file
from nornir_utils.plugins.functions import print_result

nr = nr.filter(F(tenant__name="pod1"))

vlans = [100, 200, 300]

result = nr.run(
    task=template_file, 
    path="./demo/templates", 
    template="vlan.j2", 
    name="Render VLAN Template", 
    vlans=vlans)

print_result(result)
```

4. You will now see the output from Nornir of the rendered output.

The next step in this process is to write the rendered output to a file.

## Exercise 7 – Tasks: Write to File with Nornir
Lets now use Nornir to write output to a file.

1. Run `task lab`, to start the ipython shell.
```bash
task lab
```

2. Write the rendered template to a file.
```python
from nornir_utils.plugins.tasks.files import write_file
from nornir_utils.plugins.functions import print_result
from nornir.core.task import Task
import uuid

nr = nr.filter(F(tenant__name="pod1"))

result = nr.run(
    task=write_file,
    content="Hello from AutoCon3!\n",
    append=True,
    filename="./demo/output/hello.txt"
)

print_result(result)
```

3. View the file.
```bash
cat ./demo/output/hello.txt
```

4. You will see a file has been created and some simple content added to a file. The same file has been appended to multiple times. Each entry from a host from the filter nr object.

## Exercise 8 - Tasks: Create a Custom Task
Lets now create a simple custom task. At this point we wont make this task do anything network related, but just get a fill for Nornir custom tasks.

1. Run `task lab`, to start the ipython shell.
```bash
task lab
```

2. Create a custom task.
```python
from nornir.core.task import Task, Result
from nornir_utils.plugins.functions import print_result

def task_custom(task: Task, x: int, y: int) -> Result:
    z = x + y

    return Result(
        host=task.host, 
        result=f"Hello from {task.host}! The result is {z}"
    )

result = nr.run(task=task_custom, x=1, y=2)
print_result(result)
```

3. You will see the result of the custom task for each host in the inventory.

4. Re-run task 2 but change the inputs of x and y.
```python
result = nr.run(task=task_custom, x=2, y=2)
print_result(result)
```

You will see the results change. The result is now 4.
This is a simple example. But as we will see rather then performing a simple sum, in the real world we will perform network related tasks and pass the results between the tasks to perform different operations.


## Exercise 9 – Tasks: Query GraphQL with Nornir
Lets now step things up and use Nornir to query GraphQL.
To do this we will create a custom task that will query GraphQL and return the data from the GraphQL query, for each of the hosts.

1. Run `task lab`, to start the ipython shell.
```bash
task lab
```

2. Query GraphQL.
```python
from nornir_utils.plugins.functions import print_result
from nornir.core.task import Result, Task
from nornir.core.filter import F

import requests

nr = nr.filter(F(tenant__name="pod1"))

GRAPHQL_QUERY = """
query {
    device_list {
        name
        platform {
            name
        }
    }
}
"""

# Get data from NetBox using GraphQL on a per host basis.
def task_collect_graphql(task: Task, netbox_fqdn: str, netbox_token: str, query: str) -> Result:
    """
    This task will query NetBox using GraphQL and return the data from the GraphQL query
    """

    headers = {
        "Authorization": f"Token {netbox_token}",
        "Content-Type": "application/json",
        "Accept": "application/json",
    }

    graphql_variables = {"deviceName": task.host.name}

    result = requests.post(
        url=f"{netbox_fqdn}/graphql/",
        json={"query": query, "variables": graphql_variables},
        headers=headers,
    ).json()

    return Result(host=task.host, result=result["data"]["device_list"][0])

nr.filter(tenant="pod1")

result = nr.run(
    task=task_collect_graphql,
    name="Query GraphQL",
    netbox_fqdn=NETBOX_FQDN,
    netbox_token=NETBOX_TOKEN,
    query=GRAPHQL_QUERY,
)
print_result(result)
```

You will now see the results. i.e the data from GraphQL from NetBox for each of the hosts. 


## Exercise 10 – Putting it all together
Lets now put everything together and perform the following for your **lab device**.
1. Nornir inventory filter
2. Query GraphQL
3. Render Jinja2 template
4. Write to file

> [!IMPORTANT]
> Please ensure you have added your STUDENT_ID the `.env` file before running the exercise.


1. Locate the `demo/003_nr_build.py` file.
2. Open the file, and review the code.

3. What is the difference between a "custom" task and a "plugin" task?

<details>
<summary>Answer</summary>
A custom task is a task that is written by the user, and is not provided by Nornir. A plugin task is a task that is provided by an install plugin.
</details>

4. What is the main task we are running with `nr.run`?

<details>
<summary>Answer</summary>
The main task we are running with <code>nr.run</code> is <code>task_build_config</code>.
</details>

5. What sub tasks are being run within `task_build_config`?

<details>
<summary>Answer</summary>
<code>task_build_config</code> is running a sub "custom" task <code>task_collect_graphql</code>, and then passing the data from the task to another plugin task <code>template_file</code>, finally writing the result to a file using the <code>write_file</code> plugin task.
</details>

6. Run the script, `demo/003_nr_build.py`, but clicking on the play button in the top right corner.
7. You will now see the generated network config generated in the `demo/output` directory.

We have now learned how to use Nornir to query GraphQL, render a Jinja2 template, and write the output to a file

But also we are at a fantastic point in this workshop. 
You now have a rendered configuration. In the workshop today we will use this configuration to deploy this to a sudo production environment, but in the real world you could push this to a pre-prod environment for testing, or passing it to Batfish to build a modelled network.

. 




