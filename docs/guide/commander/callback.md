## Callback Types

Cyberforge provides callbacks for different stages of tasks, and each task node can have its own callback. There are seven types
of callbacks:

1. `at_job_start`  
Executed at the start of a job.

2. `at_handler_start`  
Executed at the start of a handler.

3. `at_exception`  
Executed when a job or handler encounters an exception.

4. `at_terminate`  
Executed when a job or handler is terminated.

5. `at_handler_end`  
Executed at the end of a handler.

6. `at_job_end`  
Executed at the end of a job.

7. `at_commander_end`  
Executed at the end of the commander.


## Add Callbacks

There are multiple ways to add callbacks. Here, we'll introduce the use of the `add_callback_functions` method, which takes
`which` and `functions_info` as parameters. `which` must be one of the callback types mentioned above, and `functions_info`
is a dictionary representing callback function information. If there are multiple callback functions, this parameter can
also be a list. A function information dictionary typically has the following structure:
```python
{
    "function": callback_function,
    "params": {
        "args": position arguments of callback function,
        "kwargs": key-values arguments of callback function
    },
    "inject_task_node": bool, whether to pass back the task node into the callback function automatically
}
```

Both `params` and `inject_task_node` are optional. `params` is used to specify the parameters of the callback function.
When `inject_task_node` is set to `True`, the callback function will receive the `task_node` as a keyword argument,
representing the task node to which the callback belongs.


## Node & Edge Pattern

If you are more accustomed to the **Node**+**Edge** pattern, you can also use a callback to define your own `add_edge` logic,
for example:
```python title="simple_edge.py"
from Cyberforge.commander._commander import Job, HandlerCoroutine


def add_edge(
    from_node: Job | HandlerCoroutine,
    to_node: Job | HandlerCoroutine,
    data: Any | None = None,
) -> None:
    """To add a simple edge between two nodes.

    The simple edge ensures that the next node is automatically executed after the previous node has completed.

    Args:
        from_node: The previous node.
        to_node: The next node.
        data: Shared data, which allows each node to access data from this object.
    """
    async def next_node(from_node: Job | HandlerCoroutine, to_node: Job | HandlerCoroutine) -> None:
        # This is important; it allows a completed node to be ready again so that it can run multiple times.
        if to_node not in to_node.children:
            to_node._children.append(to_node)
        
        if isinstance(to_node, Job):
            await from_node.put_job(to_node, parent=from_node.commander)
        elif isinstance(to_node, HandlerCoroutine):
            # This is important; it allows a handler object (coroutine object) to run multiple times.
            to_node.reusable = True
            from_node.call_handler(to_node, parent=from_node.commander)
        else:
            assert False, "The connected node should be a Job or handler object."
    
    if isinstance(from_node, HandlerCoroutine):
        from_node.reusable = True

    if data is not None:
        to_node.data = data
    
    from_node.add_callback_functions(
        which="at_job_end" if isinstance(from_node, Job) else "at_handler_end",
        functions_info={
            "function": next_node,
            "params": {
                "args": (from_node, to_node),
                "kwargs": {},
            },
        },
    )
```

If you want to add an edge with conditional judgment, you can define the conditional edge like this:

```python title="conditional_edge.py"
from Cyberforge.commander._commander import Job, HandlerCoroutine


def add_conditional_edge(
    from_node: Job | HandlerCoroutine,
    map: dict[str, Job | HandlerCoroutine],
    data: Any | None = None,
) -> None:
    """Add a conditional edge between two nodes.

    The conditional edge automatically selects the corresponding next node to execute from the map
    dictionary based on the execution result of the previous node.

    Args:
        from_node: The previous node.
        map:
            The mapping dictionary stores the next nodes corresponding to different results of the
            previous node.
        data: Shared data, which allows each node to access data from this object.
    """
    async def next_node(
        from_node: Job | HandlerCoroutine,
        map: dict[Any, Job | HandlerCoroutine],
    ) -> None:
        result = from_node.result
        to_node = map.get(result)
        if to_node is None:
            return
        
        # This is important; it allows a completed node to be ready again so that it can run multiple times.
        if to_node not in to_node.children:
            to_node._children.append(to_node)
        
        if data is not None:
            to_node.data = data
        
        if isinstance(to_node, Job):
            await from_node.put_job(to_node, parent=from_node.commander)
        elif isinstance(to_node, HandlerCoroutine):
            # This is important; it allows a handler object (coroutine object) to run multiple times.
            to_node.reusable = True
            from_node.call_handler(to_node, parent=from_node.commander)
        else:
            assert False, "The connected node should be a Job or handler object."
    
    if isinstance(from_node, HandlerCoroutine):
        from_node.reusable = True
    
    from_node.add_callback_functions(
        which="at_job_end" if isinstance(from_node, Job) else "at_handler_end",
        functions_info={
            "function": next_node,
            "params": {
                "args": (from_node, map),
                "kwargs": {},
            },
        },
    )
```
For details, please refer to [node-edge pattern](./node_edge.md).
