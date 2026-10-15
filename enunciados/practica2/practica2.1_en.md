<!-- TOC start -->
- [Práctica 2 - Parte I: Plants vs Zombies Refactored](#assignment-2---part-i-pvz-refactored)
	- [Introduction](#introduction)
		- [The Model-View-Controller (MVC) architectural pattern](#mvc-pattern)
		- [Overview of the assignment](#overview)
	- [Refactoring the solution of the previous assignment](#refactoring-the-solution-of-the-previous-assignment)
		- [The command inheritance hierarchy](#command-pattern)
		- [Using interfaces to define different projections of the `Game` class](#projections-of-the-game-class)
		- [The game-object inheritance hierarchy](#game-object-inheritance-hierarchy)
		- [Delegating functionality of the `game` to the `container`](#game-object-container)
		- [Game object factories](#game-object-factories)
		- [The `GameItem` interface and callbacks](#game-item-interface)
		- [Generalizing the interactions between game objects](#generalizacion-interacciones)
	- [Testing](#pruebas)
<!-- TOC end -->

<!-- TOC --><a name="assignment-2---part-i-pvz-refactored"></a>
# Assignment 2 - Part I: Plants vs Zombies Refactored

**Submission (of parts I and II): November 16th, 09:00**

**Objetivos:** Use of inheritance, polymorphism, abstract classes and interfaces.

<!--
**Preguntas frecuentes**: Como es habitual (y normal), que tengáis dudas, las iremos recopilando en este [documento de preguntas frecuentes](../faq.md). Para saber los últimos cambios que se han introducido [puedes consultar la historia del documento](https://github.com/informaticaucm-TPI/2324-SpaceInvaders-SOLUCION/commits/main/enunciados/faq.md).
-->

<!-- TOC --><a name="introduction"></a>
## Introduction

<!-- TOC --><a name="mvc-pattern"></a>
### The Model-View-Controller (MVC) architectural pattern
You may have noticed that the package structure of the code provided in the previous assignment conforms to the Model-View-Controller (MVC) architectural pattern:

+ The **Model**  component of an MVC application contains the logic. In our case, the *Model* contains the rules of the game, the handling of the different elements of the game and, in general, everything concerned with the game such as remaining cycles, whether or not the game has finished, whether or not the player has won, etc. Correspondingly, in our package structure, the `logic` package contains the `Game` class and other classes. The `Game` class contains a method `update` to update the state of the game and methods that are called by the `run` method of the `Controller` class (see below) when the corresponding command is entered, such as the `reset` method.

+ The **View** component of an MVC application manages the user interface or user interfaces (e.g. typically, the interface presented to an administrator is different to that presented to a user). In our case, the *View* component only has to manage a simple console display. Correspondingly, in our package structure, the `view` package contains the `GamePrinter` class responsible for displaying the state of the game on the console. The constructor of `GameView` receives an instance of the `Game` class.

+ The **Controller** component of an MVC application recieves the user input, parses it and passes the resulting data to the appropriate part of the *Model* for processing before telling the *View* to display the results. In our case, the *Controller* handles only simple commands rather than, for example, capturing keyboard presses or receiving network requests. Correspondingly, in our package structure, the `control` package contains the `Controller` class whose `run` method contains the main loop of the game: while the game has not finished, call the the appropriate method of the `GameView` class to display the state of the game, then prompt the player to enter a command, then call the appropriate method of the `Game` class to process the chosen command. Notice that the `Controller` class should **not** contain any of the game logic, since the logic belongs in the *Model*.

<!-- TOC --><a name="overview"></a>
### Overview of this assignment

In this assignment, we apply the mechanisms that OOP offers to improve and extend
the code developed in the previous assignment as follows:

- In Part I of this assignment, we refactor[^1] the code of the previous assignment in the following
ways (involving two inheritance hierarchies):

  - Firstly, by removing some code from the controller `run` method and distributing
    its functionality among an inheritance hierarchy of classes, enabling the `Controller`
    class to be generic, i.e. it has no dependence on the exact commands used.

  - Secondly, by defining an inheritance hierarchy of game objects, the leaves of which will
    be the classes that
    were used in the previous assignment, namely `Peashooter`, `Sunflower` and `CommonZombie`,
    enabling the `Game` class and the `GameObjectContainer` class defined below to be generic,
    i.e. to have no
    dependence on the exact game objects used (**they certainly do not use `isinstance`
    or `getClass`!**)
    The use of inheritance enables us to avoid having identical, or nearly identical, code
    in different classes. It also enables us to reorganise how we store the information
    about the state of the game, using a single data structure instead of multiple lists.
    
    To preserve the genericity of the `Game` and `GameObjectContainer` classes without
    introducing *ad-hoc* properties in the game objects we will introduce a generic
    mechanism for handling interactions between game objects using a technique called
    *double-dispatch*.
    
  - Thirdly, we add interfaces to be implemented by the `Game` class which provide three
    partial views of its public methods, thereby classifying its client classes into
    three groups (classes in the *View*, classes in the *Controller* and classes internal
    to the *Model*) and ensuring that each group can only invoke the subset of
    the `Game` class methods contained in its partial view.
    
  The aim of these changes is to produce code that is more extensible, adaptable, maintainable,
  robust, etc. without changing the functionality. Note that if the functionality has not changed,
  the refactored code should pass the same tests as the code submitted for Assignment 1.

- Once the code has been refactored, in part II of this assignment, we add new commands and new
  elements to the game, thereby demonstrating how the new structure facilitates these extensions.

In summary, the objective of the first part of Assignment 2 is to make the code more extensible,
adaptable, maintainable, robust, etc. without changing its functionality.

[^1]: Refactoring means changing the structure of code (to improve it, presumably) without changing
its functionality.

<!-- TOC --><a name="refactoring-the-solution-of-the-previous-assignment"></a>
## Refactoring the solution to the previous assignment

<!-- TOC --><a name="command-pattern"></a>
### The command inheritance hierarchy

In the application built in the previous assignment, the user could enter several different commands in
order to update the game,
reset the game, ask for help, etc. The objective of the first half of the refactoring is to introduce
a structure to the part of the code that is concerned with processing the user commands which
will facilitate the addition of new commands, i.e. which will enable new commands to be added
with minimal modifications to the existing code. This structure is the well-known software
design pattern referred to as the *command pattern*[^2].

[^2]: As one would expect, this is a very well-known problem
in OOP which has solutions that are very well tried and tested, so we do not need to invent our own.
The solution we will use is a variant of the *Command* design pattern, one of the twenty-three
software patterns presented in the foundational software patterns book "Design Patterns: Elements of
Reusable Object-Oriented Software" first published in 1994. The general idea of the command
pattern is to encapsulate each command in its own class. You will study software design patterns in
general, and this software design pattern in particular, in the Software Engineering course.

The following classes are involved in our application of the *command pattern*:

-  `Command`: an interface that defines the public methods of any command, namely `execute`, `parse` and
   `helpText`.

- `AbstractCommmand`: an abstract class that encapsulates the functionality common to all the concrete
   commands. It has four attributes of type `String` that are initialised in its constructor -- `name`,
   `shortcut` (the abbreviated name), `details` (the initial part of its help message), `help`
   (the details of its help message) -- and *getter* methods for at least `name` and `shortcut` whose
   values are needed by the `parse` methods of the subclasses (instead of making these attributes
   `protected`).

- `NoParamsCommand`: an abstract class that inherits from `AbstractCommmand` and which is the superclass of all
   those concrete command classes that represent a command with no parameters, such as `HelpCommand`.
   The *raison d'être* of this class is that the parsing of all commands with no parameters only differs
   in the value of the `name` and `shortcut` attributes used to compare with the input text, so that the
   `parse` method of all the corresponding classes can be inherited from the `NoParamsCommand` class.
   Evidently, the same is not true of the `execute` method of the classes representing commands with no
   parameters, which will be different for every command, for which reason, the `NoParamsCommand` class
   must be abstract.

- Concrete command classes: `HelpCommand`, `ExitCommand`, etc., one such class for each program command.
  In the case of a command with parameters, the corresponding class has attributes to store the value
  of these parameters. Each concrete command class also contains the four attributes `name`, `shortcut`,
  `details` and `help` of the `AbstractCommand` class (but cannot access them directly since they are
  `private` to the `AbstractCommand` class). Though these attributes are assigned values in the
  constructor of the `AbstractCommand` class, the values used are specific to each concrete command class.
  This is achieved by passing these values up from the constructor of each concrete command class, in
  which they are defined as constants. For example, the code of the `HelpCommand` class has the following
  aspect:
    
  ```java
  public class HelpCommand extends NoParamsCommand {

	private static final String NAME = Messages.COMMAND_HELP_NAME;
	private static final String SHORTCUT = Messages.COMMAND_HELP_SHORTCUT;
	private static final String DETAILS = Messages.COMMAND_HELP_DETAILS;
	private static final String HELP = Messages.COMMAND_HELP_HELP;

	public HelpCommand(){
	    super(NAME, SHORTCUT, DETAILS, HELP);
	}

    // Implementation of the execute method
  }
  ```
  
  Each concrete command class has (at least) the following four methods (note that they may be inherited):

  - `protected boolean matchCommand(String)`: checks if the first word of the text introduced by the user
     via the keyboard (minimally processed) corresponds to the command name. It is called by the parse method.

  - `public Command parse(String[])`: checks if the text introduced by the user via the keyboard
     (minimally processed and placed in an array of strings) corresponds to a use of the command, which
     involves checking the number and *syntactic validity* of the parameter values [^4]:

     * If so, it returns an instance of the owning class, which then represents this use of the command.
       In the case of a command with no parameters, the `parse` method can simply return the value `this`.
       But in the case of a command with parameters, to avoid creating *fragile code* [^5], it must return
       a new instance of the command class, rather than changing the values of the attributes
       of the containing object and then returning `this`.

     * If not, it returns the value `null`.
  
  - `public void execute(Game, GameView)`: executes the functionality associated to the command by calling
    a method of the `Game` class, updating the view where necessary (in some commands, this method may also
    perform some other actions). If a command execution fails,
    updating the view consists of printing an error message instead of printing the current state of the
    game.
    
  - `public String helpText()` which returns that part of the text concerning this command that is returned
    by the `help` command.

- `Controller`: the code of the controller class is now reduced to only a few lines since most of the
  functionality that it included in Assignment 1 is now delegated to the concrete command classes. Note
  that the *Controller* MVC component contains the `Controller` class and the classes of the `Command`
  class hierarchy [^6].


[^4]: **Design principles**: Syntactic validity of the arguments should be checked in the parsing of the *Controller*
MVC component whereas semantic validity of the arguments (e.g. a position is invalid due to being off the board)
should be checked in the *Model* MVC component. This is in accordance with the design principle known as
*Separation of Concerns*.

[^5]: **Design principles**: Fragile code is code which makes implicit assumptions about its
environment (where this refers to the rest of the program code together with the program environment)
which can lead to it working incorrectly if its environment changes and no longer fulfills these assumptions.
In the case in point, returning the value `this` is making the implicit assumption
that there will only ever be one instance of the corresponding command class in the program at the same time
(the one that is in the `AVAILABLE_COMMANDS` list). Command objects that represent commands without parameters
are *stateless* since, in the absence of attributes, an instance is simply a pointer to a set of methods;
for such objects the above-mentioned assumption is in no way restrictive. However, command objects that
represent commands with parameters are *stateful* and
though the above assumption holds for such commands in the current program, it may not hold for them in evolutions
of the program, e.g. a program where we store a stack of command objects in order to implement an *undo* command,
so why make this assumption, given that it is completely unnecessary to do so?

[^6]: **Design principles**: Already in Assignment 1, ensuring that the *Controller* MVC component did not
contain any logic was in accordance with the design principle known as *thin controllers, fat models*, a
property that we conserve in Assignment 2.
Ensuring that the central part of the *Controller* MVC component, i.e. the `Controller` class, is a high-level
class that does not depend on any concrete implementation of the `Command` interface is in accordance with
the **D** of [the **SOLID** design principles](https://simple.wikipedia.org/wiki/SOLID_(object-oriented_design))
(the ***D**ependency Inversion Principle*).

**Main loop of the program**. In the previous assignment, in order to know which command to execute,
the main loop of the program
in the `run` method of the controller contained a switch or if-else ladder with one option for each of
the commands. In the reduced version of the controller, the `run` method has the
following aspect (your code does not have be be exactly the same but should be similar):

```java
while (!game.isFinished()) {

    Command command = CommandGenerator.parse(view.getPrompt());

    if (command != null) 
        command.execute(game, view);
    else 
        view.showError(Messages.UNKNOWN_COMMAND);
}   
```

Basically, while the game is not finished (due to internal game reasons or to a user exit), the
program reads the text entered by the user, parses it to obtain an object implementing the `Command`
interface and then calls the `execute` method of this object to effect the functionality of the command
entered by the user and to update the view after doing so, if necessary. In the case where the
input text does not correspond to any of the existing commands, the error message
`Messages.UNKNOWN_COMMAND` is printed.

**Genericity of the `Controller` class**. The most important part of the code for the
main loop shown above is the following line assigning a value to the polymorphic variable
`command`:

```java
Command command = CommandGenerator.parse(...);
```
The key point is that the controller is generic: it simply knows how to handle objects that
have an `execute` method (it relies on the `CommandGenerator` invoking a `parse` method and,
for the `help` command, a `helpText` method on those same objects) 
but does not know which concrete command is being executed nor anything about the result of
that execution; the knowledge of what functionality corresponds to each
command is contained in each concrete command class. It is this mechanism that facilitates the
addition of new concrete commands with minimal changes to the existing code. 

**The `CommandGenerator` class**. The `parse` method of this class is a static method
which returns an instance of the concrete command class that corresponds to the text entered
by the user. To this end, the `CommandGenerator` class has a static attribute containing a list
of instances implementing the `Command` interface, concretely, it contains exactly one instance of 
each of the existing concrete command classes. The `parse` method of the `CommandGenerator` method
traverses this list calling the `parse` method of each object on it.
If any of these `parse` methods returns a non-null value (which will be one of the concrete
command classes), the `parse` of the `CommandGenerator` returns this non-null value,
otherwise it returns the value `null`.

The `CommandGenerator` contains another static method `commandHelp` that generates
the output of the help command (so must be called by the `execute` method of the `HelpCommand`
class). Like the `parse` command, it accomplishes its task by traversing the `AVAILABLE_COMMANDS`
list but in this case invoking the `helpText()` method of each object on the list.

The following is a skeleton of this code:

```java
public class CommandGenerator {

    private static final List<Command> AVAILABLE_COMMANDS = Arrays.asList(
        new UpdateCommand(),
        new ResetCommand(),
        new HelpCommand(),
        new ExitCommand(),
        // ...
    );

    public static Command parse(String[] commandWords){
	//...
    }

    public static String commandHelp(){
	//...
    }

}
```


<!-- TOC --><a name="projections-of-the-game-class"></a>
### Using interfaces to define different projections of the `Game` class

The `Game` class offers services to different parts of the program via its public methods, namely:

- classes of the *Controller* component invoke methods that implement the commands entered by the user; after the above-described refactoring, the calls to these methods are made from the body of the `execute` methods of the `Command` classes. It also invokes methods that return information about the state of the game, such as `isFinished`.

- classes of the *View* component invoke methods that return information about the state of the game needed to display the current state.

- classes of the *Model* component, here the game objects (part of the *Model*, as is the `Game` class itself), invoke those methods of `game` that concern interactions between game objects. Since these calls *to* the game from the game objects result from calls *by* the game to the game objects (via the container, usually as part of an update), they are referred to as *callbacks*.

Notice that, currently, all of the public methods of the `Game` are visible and available to all parts of the application: nothing prevents the *Model* invoking a method of `Game` that was designed for the *Controller* to invoke, e.g. a game object invoking the `reset` method, or the *View* invoking a method of `Game` designed for the *Model* to invoke, e.g. the game printer invoking the `isEmpty` method, etc.  One way of limiting this *coupling* while at the same time specifying more clearly the dependencies between the different components of the application (without
breaking the `Game` class into smaller classes) is to use the Java `interface` construct. The
idea is to use interfaces to define different projections of the public `Game` functionality (i.e. partial views on the services offered by the `Game` class), each containing the methods used by one of the three components of the application, so that
the code of each component can refer only to the corresponding interface, never to the `Game` class itself. The compiler will then detect any attempt to invoke a method that is
not declared on the corresponding interface[^7]. To that end, we define the following three interfaces :

- `GameWorld` to represent the *Model*'s internal view of the services offered by the `Game` class.

```java
public interface GameWorld {

  public static final int NUM_ROWS = 4;
  public static final int NUM_COLS = 8;

  boolean isEmpty(Position position);
  boolean isInsideBoard(Position position);

  // ...
}
```

- `GameStatus` to represent the *View*'s view of the services offered by the `Game` class:

```java
public interface GameStatus {
  int getCycles();
  // ...
}
```

- `GameModel` to represent the *Controller*'s view of the services offered by the `Game` class:

```java
public interface GameModel {

  public boolean isFinished();
  public void update();
  public void reset();
  // ...
}
```

The `Game` class must then implement these interfaces:

```java
public class Game implements GameStatus, GameWorld, GameModel {

  public static final int INITIAL_COINS = 50;

  private GameObjectContainer container;

  // Methods declared in GameModel
  // ...
	
  // Methods declared in GameWorld
  // ...
	
  // Methods declared in GameStatus
  // ...
	
  // Other methods
  // ...
}
```

Finally, in each of the three components of the program, we must replace each occurrence of the type `Game` by the corresponding interface type. For example, the execute method of the `Command` class now has the following form:

```java
public abstract void execute(GameModel game, GameView view);
```

[^7]: **Design principles**: This is in accordance with the **I** of the **SOLID** design principles (the *Interface Segregation Principle*).



<!-- TOC --><a name="game-object-inheritance-hierarchy"></a>
### The game-object inheritance hierarchy

We have seen that the use of inheritance in the `Command` inheritance hierarchy significantly reduces the
repetition of code. Moreover, the use of
inheritance and polymorphism in the *command pattern* greatly facilitates the introduction of new commands,
the key aspect being that the `Controller` class is generic, i.e. it does not handle specific commands
but only handles objects of the abstract class `Command` using polymorphic variables.

Similarly, use of inheritance in an inheritance hierarchy of game objects would also reduce the repetition
of code and facilitate the introduction of new game objects. The key aspect to obtaining the latter benefit
is that the `Game` class and the `GameObjectContainer` class defined below
be generic, i.e. they should not handle specific game objects only objects of an abstract class
called `GameObject`, from which all the concrete game object classes
(currently `Peashooter`, `Sunflower` and `CommonZombie`) derive. So *the code of the `game` and of the
`container` must **not** seek to
identify the dynamic type (i.e. which concrete subclass of `GameObject`) of the objects it is
handling* [^8].

The `GameObject` class should contain all the attributes and methods that are common to all the concrete
game object classes. Each concrete class inherits attributes and methods from the `GameObject` class and
implements its own behaviour by

 - providing implementations for the abstract methods that it inherits,
 - possibly overwriting some of the non-abstract methods that it inherits,
 - possibly adding new methods.

To specify the attributes and methods of the `GameObject` class, we need to think about the behaviour that
is common to all game objects. All game objects have a position on the board so the `GameObject` will contain the
corresponding attribute[^9]. All of them also need an attribute to store the `game` and a
boolean attribute to indicate whether they are alive or not. They have, at least, methods to manipulate their
position and a method to communicate whether they are alive or not. They will also have the method:

```
public void update()
```

Other possible methods: decomposing part of the behaviour of the elements of the game into life-cycle methods
has proved useful:

- `onEnter()`: invoked when this element enters the game
- `onExit()`: invoked when this element leaves or is removed from the game

Note that in many simple objects these methods will be trivial or even empty.

We now provide the skeleton of the code for the `GameObject` class. The `GameItem` interface implemented
by this class is described below.

```java
public abstract class GameObject implements GameItem {

  protected GameWorld game;
  protected Position position;

  GameObject(GameWorld game, Position p) {
    this.game = game;
    this.position = p;
  }

  public boolean isInPosition(Position other) {
    return this.position.equals(other);
  }

  public Position getPosition() { ... }

  abstract public String getIcon();

  abstract public String getDescription();

  abstract public boolean isAlive();

  abstract public void onEnter();

  abstract public void onExit();

  // ...
}
```

Observations regarding the game object hierarchy:

- in the `GameObject` class, the choice of whether to define an abstract method to be overwritten in all
  of the subclasses or a method containing default behaviour (which may
  be simply an empty body) to be overwritten only in those subclasses that do not implement this default
  behaviour, is a design decision that should be made according to which
  option produces the simplest code. For example, in the interest of simplifying the code, you may decide
  not to implement the `GameObject` methods `onEnter` and `onExit` as abstract.
  
- attributes and non-abstract methods should always be placed in the highest class possible of the
  inheritance hierarchy [^10]
  
- anticipating the introduction of more types of zombies and more types of plants, we suggest you also create
  two abstract subclasses of `GameObject` called `Plant` and `Zombie`, where `Peashooter` and `Sunflower`
  derive from the former and `CommonZombie` derives from the latter.



[^8]: **Design principles**: ensuring that `Game` and `GameObjectContainer` are high-level classes that do
not depend on any concrete implementation, is in accordance with the **D** of the **SOLID** design principles
(the ***D**ependency Inversion Principle*). A possible improvement would be to place those of the `GameObject`
methods used by the `Game` and by the `GameObjectContainer` in interfaces and then ensure that the latter
classes only refer to these interfaces, never to the abstract class itself.

[^9]: Consider replacing the current `Position` class with a [Java `record`](https://docs.oracle.com/en/java/javase/17/language/records.html).

[^10]: **Design principles**: This is in accordance with the DRY (*Don't Repeat Yourself*) design principle. 

<!-- TOC --><a name="game-object-container"></a>
### Delegating functionality of the `game` to the `container`

In the same way as we simplified the `Controller` class, it would also be a good idea to simplify the `Game` class
by reducing its role to that of coordinating the other classes. This can be done by *delegating* much of the functionality
of the `Game` class to other classes, reducing the body of many of its methods to mere invocations of methods of
other classes, which is where the work is really carried out. The main class to which methods are delegated is the
`GameObjectContainer` (this class could be considered to represent the board of the game).

The `GameObjectContainer` encapsulates the data structure used to store the game objects (hereinafter we
will refer to the unique instance of this class as the `container`) as well as the methods for managing them
(updating them, deleting them etc.). For the store itself, we can use any type of collection; for simplicity,
we propose to use the `ArrayList` class[^11], typed using the `List` interface, declared as follows:

```java
public class GameObjectContainer {

	private List<GameObject> gameObjects;

	public GameObjectContainer() {
		gameObjects = new ArrayList<>();
	}
    //...
```

Notice that, thanks to the use of the `GameObject` abstract class and associated inheritance hierarchy:

- The container encapsulates a single data structure for storing all
  the different elements of the game (rather than a data structure for each type of element as in assignment 1).
  It is of great importance that **all** the implementation details of the `GameObjectContainer` be private so,
  for example, it should not export the value of any of the attributes of the `ArrayList` class that it is
  using to store the game objects (such as the number of elements currently in the array). This information
  hiding enables the implementation of the container to be changed without affecting the rest of the program
  code [^12].

- Similarly to the methods of the `Game` class, the methods of the container class do not know which
  game objects are being stored and manipulated, dealing only in abstractions. Neither the
  `Game` nor the `GameObjectContainer` should contain any references to concrete subclasses of the
  `GameObject` class.

- Most of the logic of the game is in the game object methods (i.e. the methods of the concrete
  subclasses of the `GameObject` class). Each such concrete subclass knows how it is updated, what
  happens when it is
  attacked, etc. The code provided still assumes the use of a `ZombieManager` but with a reduced role
  w.r.t. assignment 1, since the zombies themselves are now stored by the container in the common store.

[^11]: `ArrayList` is a class defined in the Java collections library. The classes of this library
make use of *parametric polymorphism*, known as *Java generics* in Java, first
used in functional programming in the mid-70s and introduced in Java in 2004 (with Java 5).
The polymorphism associated with inheritance is called *inclusion polymorphism* or *subtype polymorphism*.
Java generics will be studied in detail in the TP2 course.

[^12]: **Design principles**: This is in accordance with the design principle known as *Separation of Concerns* or,
more specific to classes in OOP, the *Single Responsibility Principle*, the **S** of the **SOLID** design
principles. As a *sanity check*, imagine, for example, that you decided to change the implementation of the
`GameObjectContainer` to store the game objects in a 2-dimensional array of objects that do not know
their own position, instead of using a list of objects that do know their own position. Would you need to
introduce any modifications in the code of the `Game` class of your implementation? The answer should be no.

<!-- TOC --><a name="game-object-factories"></a>
### Game object factories

*** To be added shortly ***

<!-- TOC --><a name="game-item-interface"></a>
### The `GameItem` interface and callbacks

*** To be added shortly ***

<!-- TOC --><a name="generalising-interactions"></a>
### Generalising the interactions between game objects

*** To be added shortly ***

<!-- TOC --><a name="pruebas"></a>
## Testing

*** To be added shortly ***



















