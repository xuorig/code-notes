#### puma/lib/rack/handler/puma.rb
  - Defines the handler (takes App and conf as args)
  - Sets up a new laucher with conf and events
  
#### puma/lib/puma/launcher.rb
  - Runs `#run`, starts the server
  
#### puma/lib/puma/single.rb < puma/lib/puma/runner.rb
  - Runs `#run` on the "runner"
  - Loads the App `@launcher.config.app` 
  - Loads and binds the TcpServers
  - Starts a `Puma::Server`, setting min thread max thread

#### puma/lib/puma/server.rb `#run`
  - intializes a thread pool `puma/lib/puma/thread_pool.rb`
    - Maintains a todo list
    - Threads pop off that list when available and executes block with the client
  - calls `handle_servers`, the main loop that handles client connections
  - every new client socket gets added to the pool in a non blocking way (there is a special check socket used for executing commands such as `restart`, `stop` etc.)
  - When the pool picks off a client from the todos, checks if it should queue requests or execute inline.
  - If we allow queuing requests, Puma adds the client to a Reactor object.

#### puma/lib/puma/reactor.rb
  - Reactor maintains a list `inputs` of clients.
  - Also an "event loop" reading from the clients when ready.
  - Clients are added back into the app ThreadPool after

#### puma/lib/puma/server.rb
  - This time the client is picked up again from the thread pool, but is ready to be processed
  - `process_client` is called, along with `handle_request`
  - handle request can finally read the request, normalize the env for Rack app, and call `@app.call` (Rails app for edxample)
  - Puma constructs the response and sends it back to the client


