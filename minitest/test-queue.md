 ## Basic Minitest API
 
 - `Minitest.run`
    - Main API, runs all test suite
    
  - `Minitest::Runnable.runnables`
    - All test suites (subclasses) that inherit Runnable (Test / Spec / Etc)
    
 ```ruby
  ##
  # This is the top-level run method. Everything starts from here. It
  # tells each Runnable sub-class to run, and each of those are
  # responsible for doing whatever they do.
  #
  # The overall structure of a run looks like this:
  #
  #   Minitest.autorun
  #     Minitest.run(args)
  #       Minitest.__run(reporter, options)
  #         Runnable.runnables.each
  #           runnable.run(reporter, options)
  #             self.runnable_methods.each
  #               self.run_one_method(self, runnable_method, reporter)
  #                 Minitest.run_one_method(klass, runnable_method)
  #                   klass.new(runnable_method).run
```
 
## TestQueue Additions
 
  - `TestQueue::Runner`
    - initialized with a test framework (Minitest in this example)
    - keeps a queue of suites to run
  
  - `TestQueue::Runner#execute`
    - main API. Runs all tests
    - Master process opens a TCP or Unix socket server
    - runs #prepare, which can be implemented by custom runners to run things in the master process
    - starts discovering suites (see `#discovering_suites`)
    - spawns child workers (see `#spawn_workers`)
    - starts distributing suites for child workers to run (see `#distribute_queue`)
    
    - `Runner#discovering_suites`
      - forks a child process
      - Grabs all test files to run (`ARGV` in the case of minitest)
      - grabs all suites from every file (`MiniTest::Test.runnables`)
      - puts the suite name and path into the shared socket
      
    - `Runner#spawn_workers`
      - Spawns an amount of workers (configurable)
      - build an iterator (see `TaskQueue::Iterator`)
      - calls run_worker with the iterator (see `TestQueue::Runner::MinitTest#run_worker`)
      
    - `Runner#distribute_queue`
      - runs IO.select on the shared socket
      - handles all commands, most interesting is `NEW SUITE` & `POP`
      - NEW SUITE is master queuing test suites received from the discovery process
      - POP is used by child workers to pop off tests of the queue (once again, see `TaskQueue::Iterator`)
      
    - `TestQueue::Runner::MinitTest#run_worker`
      - Basically whats called by child workers when using MiniTest
      - Quite simple in apparence:
      ```
        ::MiniTest::Test.runnables = iterator
        ::MiniTest.run ? 0 : 1
      ```
      - Minitest ends up calling `Runnable.runnables.each` but runnables is not actually an array, its a TaskQueue::Iterator :zomg:
      - See `TestQueue::Iterator`
      
    - `TestQueue::Iterator`
      - The main loop for child workers
      - Send POP to get a job from master process
        - Get back `WAIT` if it should try again later
        - Or a marshalled payload if there was a job ready to be consumed
      - loads the suite using its name and path. (The child worker has a copy of the @test_framework so it can find suites from a file)
      - The suite is yieled back to `runnables.each` and we're back into the standard minitest flow!
      
### Notes on test order :thinking:

Few spots where think can be sorted / ordered / randomized:

  - in `Minitest.__run`: `suites = Runnable.runnables.reject { |s| s.runnable_methods.empty? }.shuffle`
    - this will shuffle runnables (test suites / subclasses)
    
  - In `MiniTest::Test#runnable_methods:`
    - this will shuffle test methods from a test subclass
  
  ```ruby
    def self.runnable_methods
      methods = methods_matching(/^test_/)

      case self.test_order
      when :random, :parallel then
        max = methods.size
        methods.sort.sort_by { rand max }
      when :alpha, :sorted then
        methods.sort
      else
        raise "Unknown test_order: #{self.test_order.inspect}"
      end
    end
```
