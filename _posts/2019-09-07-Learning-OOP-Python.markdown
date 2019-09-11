---
layout: post
title:  "Learning OOP in Python Part1"
date:   2019-09-07 12:00:00 -0400
categories: jekyll update
---

## Overview

- Classes
- Member Functions
- Member Variables
- Object Lifetime

## Classes

A means to bundle data and functionality together.  We use classes in order to group together behavior and state so that we do not need to think about the "details" of what is going on.  Instead, we interact with a higher level interface which is easier to reason about.  

An instance of a class is a unique object that has it's own state, each instance is a unique instantiation of that object and there state is independent of one another.

### Example:

```python
class Person:
    def __init__(self, name):
        self._name = name
    def print():
        print(self._name)

def __name__ == "__main__":
    bob = Person("bob")
    jeff = Person("jeff")
    bob.print()
    jeff.print()
```

#### Output
```
bob
jeff
```
## Member Function

A member function is a way to access and manipulate an instance of a class.  Member functions can be declared in Python by having self as the first argument of the function declaration.  This self variable references the state of the class instance.

Member variables are useful when you want to manipulate the state of an object and are a gateway to manipulating the internal state of an Object

### Example

```python
class Ball:
    def __init__(self):
        self._loc_x = 0
        self._loc_y = 0
        self._is_hopping = False
    def ToggleHopping(self):
        self._is_hopping = not self._is_hopping
    def Print(self):
        if self._is_hopping:
            print("Im Hopping")
        else:
            print("Still")
if __name__ == '__main__':
    ball = Ball()
    ball.Print()
    ball.ToggleHopping()
    ball.Print()
```

#### Output

```
Still
Im Hopping
```

### Member Variable

A member variable is some piece of data that belongs to an instance of a class.  This is meant to represent the internal state of the class.  Each instance of a class has separate instances of member variables.  Member variables in Python are always publicly accessible, however it is best practice to prefix any member variable with `_` which signals that the variable should not be modified directly.

When you want to expose private variables you can do so using member functions or getters/setters.  This provides you a mechanism to control what the state gets changed to inside of the object.

#### Example

```python
map_width = 400
map_height = 400
bad_ball_names = [ 'Wilson', 'Square' ]
class Ball:
    def __init__(self, name):
        self._locX = 0
        self._locY = 0
        self.set_name(name)
        self.Color = "Red"
    def set_loc_x(self, x):
        if 0 < x < map_width:
            self._locX = x
    def set_loc_y(self, y):
        if 0 < y < map_height:
            self._locY = y
    def set_name(self, name):
        if name not in bad_ball_names:
            self._name = name
    def get_loc_x(self):
        return self._locX
    def get_loc_y(self):
        return self._locY
    def get_name(self):
        return self._name
    def print(self):
        print("Location ({},{}) Name {}".format(self._locX, self._locY, self._name))

if __name__ == '__main__':
    b = Ball("Big Ball")
    b.print()
```

##### Output

```
Location (0,0) Name Big Ball
```

### Getters/Setters The better way

In python 3 they added a better way to do getter and setters.  Here is the same example as above using the @property annotation.  If your version of Python supports this prefer this method.

One thing to note is that the @property annotation is a data descriptor and exposes the variable.  So you must have a @property (getter) before defining a setter.  Using this method there is no way to only have a setter.

#### Example

```python
map_width = 400
map_height = 400
bad_ball_names = [ 'Wilson', 'Square' ]

class Ball:
    def __init__(self, name):
        self._locX = 0
        self._locY = 0
        self.name = name
        self.Color = "Red"
	# This comes first
    @property
    def loc_x(self):
        return self._locX
	# This extends the property and adds a setter
    @loc_x.setter
    def loc_x(self, x):
        if 0 < x < map_width:
            self._locX = x
    @property
    def loc_y(self):
        return loc_y
    @loc_y.setter
    def loc_y(self, y):
        if 0 < y < map_height:
            self._locY = y
    @property
    def name(self):
        return self._name
    @name.setter
    def name(self, name):
        if name not in bad_ball_names:
            self._name = name
    def print(self):
        print("Location ({},{}) Name {}".format(self._locX, self._locY, self._name))
if __name__ == '__main__':
    b = Ball("Big Ball")
    b.loc_x = 24
    b.print()
```

#### Output
```
Location (24,0) Name Big Ball
```

## Object Lifetime

Luckily Python handles most of this for you but it is good to know what is going under the hood.

![Meme](https://www.probytes.net/wp-content/uploads/2018/01/5-1.png)

### Initialization

Whenever you instantiate a new object in python the __init__ method defined in the class is called.  This is typically referred to as the constructor and is meant to set up the object for use.  __new__ is also called but it is not common to see that implemented.

### Destruction

All objects in Python are reference counted.  This means that whenever you create an object in Python the garbage collector keeps track of how many references to that object are made.  When that reference count hits 0 then the garbage collector releases the memory that the object takes up back to the runtime.

You are allowed to define a Destructor in python if you want to do some clean up when that object goes out of scope.  This is typically not needed but is available to you to use.

#### Example

```python
import sys

class Person:
    def __init__(self, name):
        self._name = name
    def __del__(self):
        print("{} is getting fired".format(self._name))

class Manager:
    def __init__(self):
        self._employees = []
    def hire(self, new_employee):
        self._employees.append(new_employee)
    def num_employees(self):
        return len(self._employees)
    def print_employees(self):
        for minion in self._employees:
            print(minion._name)

bob = Manager()

def make_person():
    riff_raff = Person("Riff Raff")
    print(sys.getrefcount(riff_raff))
    bob.hire(riff_raff)
    print(sys.getrefcount(riff_raff))

if __name__ == "__main__":
    make_person()
    bob.print_employees()
    print(sys.getrefcount(bob._employees[0]))
```

#### Output

```
2
3
Riff Raff
2
Riff Raff is getting fired
```

## Exercise

The banking system is becoming out of date and is unable to scale to modern requirements, lets create a new banking system in a language that can leverage AWS cloud services like lambda.  Our banking system is going to have to support the following things.

    1.  Allow users to create new banks, banks will have the following properties
        - Name (read only)
        - Location (read only)
        - Country (read only)
            - must be validated using [pycountry](https://pypi.org/project/pycountry/) for existance
    2.  Each bank will need accounts, those accounts will have
        - Account Number (read only)
            - should be unique and incrementing
        - Owner
        - Secondary Owners (more than one)
        - Balance
            - only owner call withdraw money, secondary owners can view this
    3.  We will want to have a banned users list

Here are the following functions we will want to have
    - Create a bank
    - Print out all banks
    - Select a bank
    - List all accounts in a bank
    - Select account
    - Add money to an account
    - Print account balance

# Appendix

## References
1.  https://docs.python.org/3/tutorial/classes.html
2.  https://code.tutsplus.com/tutorials/what-are-python-namespaces-and-why-are-they-needed--cms-28598
3.  https://technobeans.com/2012/01/17/object-lifetime-in-python/
4.  https://www.journaldev.com/14893/python-property-decorator
5.  https://stackoverflow.com/questions/17576009/python-class-property-use-setter-but-evade-getter