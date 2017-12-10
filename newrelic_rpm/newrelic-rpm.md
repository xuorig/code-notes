# NewRelic's RPM

## Main classes/modules

- NewRelic::Agent
- NewRelic::Agent::Agent
- NewRelic::Agent::EventListener
- NewRelic::Agent::Harvester
- NewRelic::Agent::EventLoop
- NewRelic::Agent::Threading::AgentThread

At boot, when using Unicorn, RPM will defer the start of the worker thread and instead wait for a connection to a Unicorn worker before start the thread. *(I'm Assuming this is because you want to be sure it's going to be in a worker process and not master)*

## SCHEDULING, AND THE WORKER THREAD

### Init Process

 - The main Agent creates a new instance of `Harvester` which takes the event listener as a Parameter.
 - in initialize it adds the `start_transacation` event to the listener to know when a new connection comes in.
 - When the event listener gets a notify for `start_transaction` it calls back `on_connection` on `Harvester`. Which starts the Agent Thread. by calling `agent.after_fork`
 - `agent.after_fork` Creates an `EventLoop` and subscribes to a bunch of events.
 - That EventLoop is ran inside an `AgentThread` which seems to be a wrapper around a "Simple" Ruby Thread.
 - Before starting the thread RPM connects to NewRelic servers

 ...
 
### The Worker Agent / Event Loop

- Inside that thread is an Event Loop running and collecting events.

#### `def on`
We can listen to events by using 

```
loop.on(:event) do
  stuff
end
```

- `on`'s behaviour is quite meta:
It fires an event on itself. An `__add_event` event :o.

- `fire` basically adds an event to be processed inside a queue.

When we call `run` on the `EventLoop` it will start processing anything that's added. The first events it will see are `__add_event` events for which the callback is already defined.

You can guess what it does: setting up a new event, callback pair in `subscribers`.

### Reporting events

Let's only take `report_data` event for example. We call 

```
loop.on(:report_data) do
  transmit_data
end
```

- This will have an effect of adding a new subscriber on the event loop. Next time we `fire` `report_data`, the callback will be called.

- There's an other meta event we can add in the loop. It's called `__add_timer`. Add timer, as the name says, adds a timer to our loop.

- Timers are used to send data every X amount of time. Sending the data over wire can take quite a lot of time so we dont want to bombard our server with possibly millions of event. The time we wait is configurable.

### Processing events

- The loop uses an infinite loop to process new events that are added to the event queue.
- It uses `IO.select` with a timeout which is equal to the amount of time before the next timer should fire.
- Now that we know a timer is ready to be fired, we fire all timers which will add an event to the queue when fired.
- In the next line events are all popped of the queue, and their callbacks are fired too!

#### The Self Pipe Trick

- Now we talked how `on` basically fires a meta event to add a new event + callback pair. What if a user set a really long timer time ? What if no timers are added ? The queue could wait indefinitely because it has no idea we're ready to process a new event. RPM uses the `self_pipe` trick to solve that problem. When we fire an event, the `wakeup` method is called which sends a byte of a `IO.pipe` that the loop created. Using `IO.select` we can listen for that pipe and unblock when we know fire has been called. This way we can process the `__add_event` event!



## CONSUMING AND SENDING DATA OVER WIRE

Key Classes

  - `NewRelic::Agent::NewRelicService` (Speaks to New Relic Servers)
  - When asked to send data, tries and *harvest* from "containers"

Example Error DATA:

  - `NewRelic::Agent::ErrorCollector` -> `NewRelic::Agent::ErrorTraceAgregator`
  - Error Trace Agregator is basically just a queue with a Mutex. When you harvest from the queue, you get the entire queue, and the queue is reset to `[]`

  
  When data is harvested, `send_data_to_endpoint` (errors, /errors_data)
  
  
  
## On the Rails Side

`NewRelic::Agent.notice_error` for example

init_plugin
install_shim

include `ControllerInstrumentations` and `ActionController`

monkey patch `process_action`

`#perform_action_with_newrelic_trace` =

Transaction.start

Do the thing

Transaction.stop

throw stuff inside the ErrorTraceAgregator
