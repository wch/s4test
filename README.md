Issues with S4 class unions
===========================

This repository contains two simple packages that illustrate some issues that I'm running into with S4 class unions. The issues are very hard for me to pin down, but they seem to be related to the presence or absence of subclass references.

This document is an attempt to illustrate some of the odd behavior and hopefully gain some insight into what's going on.



## Initial setup

This set of examples requires installing the stripped-down test packages in this repository:

```R
install.packages(c('s4union', 's4subclass'), repos=NULL, type='source')
```
These two packages are very simple. The s4union package just contains the `Uclass` class union, and the s4subclass package just contains the `Subclass` class. They have the following definitions:

```R
setClassUnion("Uclass", c("logical", "numeric"))

setClass("Subclass", contains = c("numeric"))
```



## Scenario 1: both classes loaded via `library()`

In this scenario, the behavior is unsurprising. If the packages are loaded with `library()`, then the classes are not super/sub classes of each other.

```R
library(s4union)
library(s4subclass)
getClass("Uclass")    # 'Subclass' is not a subclass
# Extended class definition ( "ClassUnionRepresentation" )
# Virtual Class "Uclass" [package "s4union"]

# No Slots, prototype of class "logical"

# Known Subclasses: 
# Class "logical", directly
# Class "numeric", directly
# Class "integer", by class "numeric", distance 2
# Class "ordered", by class "numeric", distance 4

getClass("Subclass")  # Does not extend Uclass
# Class "Subclass" [package "s4subclass"]

# Slots:
              
# Name:    .Data
# Class: numeric

# Extends: 
# Class "numeric", from data part
# Class "vector", by class "numeric", distance 2


# Unloading removes the classes, as expected
unloadNamespace("s4subclass")
unloadNamespace("s4union")
c("Uclass", "Subclass") %in% getClasses()
# [1] FALSE FALSE
```


## Scenario 2: class union loaded via `library()`, subclass created at console

If the s4union package (with `Uclass`) is loaded with `library()`, but `Subclass` is created from the R console, then the behavior is different. When inspecting `Uclass`, `Subclass` is not listed as a subclass, but when inspecting `Subclass`, it says that it extends `Uclass`. So the relation is one-sided.

Furthermore, even after unloading the s4union package, `Subclass` still thinks it extends `Uclass`, even though `Uclass` is gone.

```R
# In a new R session
library(s4union)
setClass("Subclass", contains = c("numeric"))
getClass("Uclass")    # 'Subclass' is not listed as a subclass
getClass("Subclass")  # Extends Uclass


unloadNamespace("s4union")
getClass("Uclass")    # As expected, Uclass is not defined
getClass("Subclass")  # This still extends Uclass (!)
# Class "Subclass" [in ".GlobalEnv"]
#  ...
# Extends: 
# Class "numeric", from data part
# Class "vector", by class "numeric", distance 2
# Class "Uclass", by class "numeric", distance 2
```



## Scenario 3: class union loaded via `library()`, subclass loaded with devtools

When we run `library(s4union)` and then load the `s4subclass` package using `load_all()` from devtools, the behavior is different compared to scenario 1. `load_all()` uses `sys.source()` to load the R files, as opposed to `library()`, which calls `loadNamespace()`, which eventually calls `lazyLoad()` on the packaged .rdb/.rdx files.

In this scenario, `Subclass` is listed as extending `Uclass`. Note that `load_all()` tries to mimic the procedure in `library()`; part of this is creating a namespace for the s4subclass package.

```R
# In a new R session
library(devtools)
library(s4union)
load_all("s4subclass/")

# Unlike when loaded with library(), this is listed as extending Uclass
getClass("Subclass")
# Class "Subclass" [package "s4subclass"]
#
# Slots:
#             
# Name:    .Data
# Class: numeric
#
# Extends: 
# Class "numeric", from data part
# Class "vector", by class "numeric", distance 2
# Class "EnumerationValue", by class "numeric", distance 2
# Class "Uclass", by class "numeric", distance 2
```

Notice that it also extends `EnumerationValue`, which is a class union from the RCurl package, which is automatically loaded because it is an import for devtools. It's defined as:

```R
setClassUnion("EnumerationValue", c("numeric", "integer", "character", "EnumValue"))
```


The problem happens when you attempt to unload the namespace. This results in a warning message:

```R
unloadNamespace("s4subclass")
# Warning message:
# In .removeSuperclassBackRefs(cl, cldef, searchWhere) :
#   could not find superclass "EnumerationValue" to clean up when removing subclass references to class "Subclass"
```

For some reason, R appears to be able to clean up the reference to `Uclass`, but not to `EnumerationValue`.


Note that if you load all the packages with `library()`, unloading works just fine:

```R
# In a new R session
library(devtools)
library(s4union)
library(s4subclass)
getClass("Subclass")   # No mention of EnumerationValue or Uclass

unloadNamespace("s4subclass") # No error
```


## The hard questions

* Why is there the warning about `EnumerationValue` when the package is loaded with `load_all()`? (Also, why isn't there a warning about `Uclass`?)

* Is it possible to get it to behave like `library()`? (Note that `load_all()` calls `sys_source()`, whereas `library()` makes use of `lazyLoad()`.)





Other notes
===========

See https://github.com/hadley/devtools/issues/157 and https://github.com/hadley/devtools/issues/168 for real-world problems that seem to be related to S4 class unions.


