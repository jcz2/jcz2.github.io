---
layout: post
title: Modeling stateful computations with State Monad
date: 2022-12-11 16:33 +0100
---
In functional programming, we strive to use pure functions.
One of the things that pure functions don't do is mutate data.
Yet mutating is very common in an object-oriented approach.
This is how we model the domain, by creating classes that represent entities.
These classes have some internal state and a set of behaviors
that mutate this state. Since in the functional approach, we avoid functions
that mutate data, what we do instead is pass this data explicitly by parameters
and instead of mutating it, we return a new value.
To see how this works in practice let's start with a simple to-do app.
Let's model it using an object-oriented approach and then we will update it into a more functional style and observe the difference.
```scala
package todolistv1

import java.util.UUID.randomUUID
import scala.collection.mutable

case class Todo(id: String, description: String, completed: Boolean)

class TodoList {
  private val todos = mutable.Map[String, Todo]()

  def add(description: String): Todo = {
    val todo = Todo(randomUUID.toString, description, completed = false)
    todos.addOne(todo.id, todo)
    todo
  }

  def remove(id: String): Unit = todos.remove(id)

  def update(id: String, completed: Boolean): Unit = {
    val todo = todos.get(id)
    if (todo.isDefined) {
      todos.update(id, todo.get.copy(completed = completed))
    }
  }

  def check(id: String): Unit = update(id, completed = true)

  def uncheck(id: String): Unit = update(id, completed = false)

  def isCompleted: Boolean = todos.forall { case (_, todo) => todo.completed }

  def clear(): Unit = todos.clear()
}
```
Here we have a `TodoList` class with basic operations like adding, removing, checking, and unchecking todos.
We can also check if all the todos in the list are completed and clear the list of todos.

Next, let's take a look at our test client that calls our to-do list and performs operations on it. Let's say the client wants to add a todo, mark it as completed, then check if all todos are completed and if so, clear the list of todos.
This client although may seem contrived, will serve the purpose of illustrating the point.
```scala
object Main extends App {
  val todoList = new TodoList()
  val todo = todoList.add("Make lunch")
  todoList.check(todo.id)
  if (todoList.isCompleted) {
    todoList.clear()
  }
}
```
We can see that with the object-oriented approach, we just invoke methods that mutate the internal state of the object.
As mentioned already in functional programming we'd rather avoid mutable state and make use of pure functions that explicitly define all of their dependencies and instead of performing mutations, they return new updated values.
With this in mind let's refactor our code.
```scala
package todolistv2

import java.util.UUID.randomUUID

case class Todo(id: String, description: String, completed: Boolean)
case class TodoList(todos: Map[String, Todo] = Map())

object TodoList {
  def add(todoList: TodoList, description: String): (Todo, TodoList) = {
    val todo = Todo(randomUUID.toString, description, completed = false)
    val newTodoList = todoList.copy(todos = todoList.todos.updated(todo.id, todo))
    (todo, newTodoList)
  }

  def remove(todoList: TodoList, id: String): TodoList =
    todoList.copy(todos = todoList.todos.removed(id))

  def check(todoList: TodoList, id: String): TodoList =
    update(todoList, id, completed = true)

  def uncheck(todoList: TodoList, id: String): TodoList =
    update(todoList, id, completed = false)

  def update(todoList: TodoList, id: String, completed: Boolean): TodoList =
    todoList.todos.get(id)
      .map(todo => {
        todoList.copy(todos = todoList.todos.updated(id, todo.copy(completed = completed)))
      })
      .getOrElse(todoList)

  def isCompleted(todoList: TodoList): Boolean =
    todoList.todos.forall { case (_, todo) => todo.completed }

  def clear(todoList: TodoList): TodoList =
    todoList.copy(todos = Map[String, Todo]())
}
```
What we've done here is add the to-do list as an explicit parameter.
Each of the methods takes the to-do list and returns a new to-do list.
An added advantage is that we can now turn our `TodoList` class into a case class. We also move the methods operating on `TodoList` to
its companion object.
Let's take a look at our client:
```scala
object Main extends App {
  val todoList = TodoList()
  val (todo, todoList1) = TodoList.add(todoList, "Make lunch")
  val todoList2 = TodoList.check(todoList1, todo.id)
  val todoList3 = if (TodoList.isCompleted(todoList2)) {
    TodoList.clear(todoList2)
  } else {
    todoList2
  }
}
```
Although our methods are pure, we introduced a problem.
Notice how we have to pass the updated to-do list around
to every function. This makes the code not only more verbose
but now it's easy to pass the incorrect version of the to-do list.
Since each version of the to-do list is of the same type,
if I by mistake pass `todoList1` to `isCompleted` instead of `todoList2`:
```scala
val todoList3 = if (TodoList.isCompleted(todoList1)) {
  TodoList.clear(todoList2)
} else {
  todoList2
}
```
The compiler won't let me know I made a mistake and will compile the code.
I will only learn there is a mistake by running the code.

This is where the State monad comes in.
State monad takes care of passing the state between functions automatically
so we don't have to do it by hand, therefore reducing the possiblity of error.

State monad represents a function of type `S => (S, A)` where `S` is the state and `A` is the return value of the function. 
How can we model one of our functions using the State monad?
Let's take the `add` method for example, its signature is:
```scala
(todoList: TodoList, description: String) => (Todo, TodoList)
```
It takes a to-do list and a description and returns a new to-do and a new to-do list with the newly created to-do added to the list.
How can we adjust this type to fit the type required by State monad?
We know that `TodoList` is our state and `Todo` is the return value,
so let's rearrange the types:
```scala
(description: String) => (todoList: TodoList) => (TodoList, Todo)
```
Now we can see that the second part aligns with the type of the State monad:
```scala
TodoList => (TodoList, Todo)
S        => (S,        A   )
```
Which means we can turn it into:
```scala
State[TodoList, Todo]
```
Let's rewrite our `add` function once again:
```scala
(description: String) => State[TodoList, Todo]
```
We can go through the same process for the rest of our methods:
```scala
import java.util.UUID.randomUUID
import cats.data.State

case class Todo(id: String, description: String, completed: Boolean)
case class TodoList(todos: Map[String, Todo])

object TodoList {
  def add(description: String): State[TodoList, Todo] = for {
    todoList <- State.get[TodoList]
    todo = Todo(randomUUID.toString, description, completed = false)
    _ <- State.set(todoList.copy(todos = todoList.todos.updated(todo.id, todo)))
  } yield todo

  def remove(id: String): State[TodoList, Unit] = State { todoList =>
    (todoList.copy(todos = todoList.todos.removed(id)), ())
  }

  def check(id: String): State[TodoList, Unit] = update(id, completed = true)

  def uncheck(id: String): State[TodoList, Unit] = update(id, completed = false)

  def update(id: String, completed: Boolean): State[TodoList, Unit] = State { todoList =>
    val newTodoList = todoList.todos.get(id)
      .map(todo => {
        todoList.copy(todos = todoList.todos.updated(id, todo.copy(completed = completed)))
      })
      .getOrElse(todoList)
    (newTodoList, ())
  }

  def isCompleted: State[TodoList, Boolean] = State { todoList =>
    (todoList, todoList.todos.forall { case (_, todo) => todo.completed })
  }

  def clear(): State[TodoList, Unit] = State { todoList =>
    (todoList.copy(todos = Map[String, Todo]()), ())
  }
}
```

With the changes above, our client can be rewritten to:
```scala
object Main extends App {
  val todoList = (
    for {
      todo <- TodoList.add("Make lunch")
      _ <- TodoList.check(todo.id)
      isCompleted <- TodoList.isCompleted
      _ <- if (isCompleted) {
        TodoList.clear()
      } else {
        State.get[TodoList]
      }
    } yield ()
  ).runS(TodoList(Map())).value
}
```
As we can see at no point, except when we run the State monad with `runS`, we have to pass the state (to-do list in our case) explicitly by hand, therefore removing possibility of error. The state is passed implicitly by the State monad.
It gets even better when we want to add another step to our client.
Let's say we want to add another to-do to our list.
With the second approach we would have to rename all the `todoList` variables
after this step, so `todoList2 -> todoList3`, `todoList3 -> todoList4` to make room for the new to-do list:
```scala
object Main extends App {
  val todoList = TodoList()
  val (todo, todoList1) = TodoList.add(todoList, "Make lunch")
  val (todo1, todoList2) = TodoList.add(todoList1, "Make dinner")
  val todoList3 = TodoList.check(todoList2, todo.id)
  val todoList4 = if (TodoList.isCompleted(todoList3)) {
    TodoList.clear(todoList3)
  } else {
    todoList3
  }
}
```
In comparison, with the State monad approach, we can add a new line
to our code and the state will be propagated automatically, no other refactoring is necessary:
```scala
object Main extends App {
  val todoList = (
    for {
      todo <- TodoList.add("Make lunch")
      todo1 <- TodoList.add("Make dinner") <---
      _ <- TodoList.check(todo.id)
      isCompleted <- TodoList.isCompleted
      _ <- if (isCompleted) {
        TodoList.clear()
      } else {
        State.get[TodoList]
      }
    } yield ()
  ).runS(TodoList(Map())).value
}
```
As we can see functional approach creates more verbosity compared to the imperative approach because we have to be explicit about all the dependencies our functions have.
To reduce this verbosity we can use the State monad.
The downside is that the code may look more complex, especially for a person who is not acquainted with functional programming. At the end of the day, whether we use State monad or not is a choice based on preference. State monad doesn't allow us to do "more" than we would be able to without it. It's just a way to model and compose stateful functions in a safer and less verbose manner.

You can find the code [here](https://github.com/jcz2/state-monad-todo-list)