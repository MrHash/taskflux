# TaskMachine
### Modular micro-service task pipelining & orchestration with validated state machine integrity.

Define micro-service tasks and arrange them into state machine managed flows with a simple and expressive API. You can create build tool chains, processing pipelines, utility services, or even serve web pages...

#### State machines for the masses!

# Examples
## Simple pipeline
We define two simple inline tasks which are independent. The machine executes the two tasks in order and then finishes.
```php
$tmb = new TaskMachineBuilder;

// Define some tasks
$tmb->task('hello', function () {
  echo 'Hello World';
});

$tmb->task('goodbye', function () {
  echo 'Goodbye World';
});

// Define and build a machine
$tm = $tmb->machine('greetings')
  // specify an initial task and transition
  ->hello([
    'initial' => true,
    'transition' => 'goodbye'
  ])
   // specify a final task
  ->goodbye(['final' => true])
  ->build();

// Run the machine.
$tm->run('greetings');
```

## Pipeline with DI
Now we introduce some more tasks with DI. Tasks are isolated by definition and optionally have expected inputs and outputs.
```php
// Bootstrap your own Auryn injector and throw it in
$tmb = new TaskMachineBuilder(new TaskFactory($myInjector));

// Define some tasks
$tmb->task(
  'translate',
  function (InputInterface $input, MyTranslationInterface $translator) {
    // Auryn injects fully constructed dependencies. Run your things.
    $translation = $translator->translate($input->get('text'));
    return ['text' => $translation];
  }
);

// Input from previous task is injectable and immutable
$tmb->task('echo', function (InputInterface $input) {
  echo $input->get('text');
});

$tmb->task('goodbye', function () {
  return ['closing' => 'Goodbye World'];
});

 // Define and build machine
$tm = $tmb->machine('translator')
  ->translate([
    'initial' => true,
    'transition' => 'echo'
  ])
  ->echo(['transition' => 'goodbye'])
  ->goodbye(['final' => true])
  ->build();

// Run with input and then echo the output from the last task
$output = $tm->run('translator', ['text' => 'Hello World']);
echo $output->get('closing');
```

## Any faults in the configuration of your machine will result in a build error! Tasks must be linked together correctly and have valid unambiguous transitions.

## Conditional branching
Machines can branch to different tasks based on conditions written in Symfony Expression Language.
```php
$tmb = new TaskMachineBuilder(new TaskFactory($myInjector));

// Define some tasks
$tmb->task('process', function () {
  // This outputs a random true or false result
  $result = (bool)random_int(0,1);
  return ['success' => $result];
});

// Task with your instantiated object which implements TaskHandlerInterface
$tmb->task('finish', new Finisher($myService));

// Task with your handler which implements TaskHandlerInterface
// Your dependencies are injected
$tmb->task('fail', MyCustomServiceInterface::class);

// Define and build a machine with different final outcomes
$tm = $tmb->machine('switcher')
  ->process([
    'initial' => true,
    // Specify switch conditions to subsequent tasks
    'transition' => [
      'output.success' => 'finish',
      '!output.success' => 'fail'
    ]
  ])
  ->finish(['final' => true])
  ->fail(['final' => true])
  ->build();
  
// Run it.
$tm->run('switcher');
```
