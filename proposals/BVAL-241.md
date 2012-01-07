---
title: Support for method validation
layout: default
author: Gunnar Morling
---

# #{page.title}

[Link to JIRA ticket](https://hibernate.onjira.com/browse/BVAL-241)

## Goal

As proposed in BV 1.0 Appendix C, the notion of method validation shall be established.

## Defining method level constraints

visibility

### Defining parameter constraints

Parameter constraints are defined by simply annotating method or constructor parameters:

#### Requirements on methods to be validated

TODO: Remove sentence

> Constraints on non getter methods are not supported.

from section 3.1.2

Methods or constructors which shall be annotated with parameter constraints must fullfil the following requirements:

* non-static

An example:

	public class OrderService {

		public OrderService(@NotNull CreditCardProcessor creditCardProcessor) {
			//...
		}

		public void placeOrder(@NotNull CustomerPK customerPK, @NotNull Item item, @Min(1) quantity) {
			//...
		}
	}



Tip: In order to use constraint annotations for method parameters, their element type must be `ELEMENTTYPE.METHOD`. All built-in constraints support this element type and it's considered a best practice to do the same for custom constraint also if they are not primarily intended to be used as parameter constraints.

#### Cross-parameter constraints

### Defining return value constraint

TODO: Should property constraints (on getter methods) also be handled as method constraint?

### Marking parameters/return values for cascaded validation

### Inheritance hierarchies

When defining method level constraints within inheritance hierarchies (that is, class inheritance by extending base classes and interface inheritance by implementing interfaces) one generally has to deal with the [Liskov substitution principle](http://en.wikipedia.org/wiki/Liskov_substitution_principle) which mandates that

* a methods preconditions (as represented by parameter constraints) may not be strengthened in sub types
* a methods postconditions (as represented by return value constraints) may not be weakened in sub types

#### Option 1: Don't allow allow refinement

#### Option 2: allow return value refinement (AND style)

#### Option 3: allow param (OR style) and return value (AND style) refinement

### Naming parameters

If the validation of a parameter constraint fails the concerned parameter needs to be identified in the resulting `MethodConstraintViolation` (see ...).

Java doesn't provide a portable way to retrieve parameter names. Bean Validation therefore defines the `ParameterNameProvider` API to which the retrieval of parameter names is delegated:

	public interface ParameterNameProvider {

		String[] getParameterNames(Constructor< ? > constructor) throws ValidationException;

		String[] getParameterNames(Method method) throws ValidationException;
	}

A conforming BV implementation provides a default `ParameterNameProvider` implementation which returns parameter names in the form `arg<PARAMETER_INDEX>`, where `PARAMETER_INDEX` starts at 0 for the first parameter, e.g. `arg0`, `arg1`etc.

BV providers are free to provide additional implementations (e.g. based on annotations specifying parameter names, debug symbols etc.). If a user wishes to use another parameter name provider than the default implementation, she may specify the provider to use with help of the bootstrap API (see ...) or the XML configuration (see ...).

### Applying property constraints to setter methods

It might be useful to have the possibility to apply property constraints (defined on getter methods) also as parameter constraints within the corresponding setter methods.

TODO: Might that be required/helpful by JAX-RS?

#### Option 1: A class-level annotation:

@ApplyPropertyConstraintsToSetters
public class Foo {

}

#### Option 2: A property level annotation:

public class Foo {

 @ApplyToSetter
 @Min(5)
 public int getBar() {
   return bar;
 }
}

#### Option 3: An option in the configuration API:

Validator validator = Validation.byDefaultProvider()
       .configure()
       .applyPropertyConstraintsToSetters()
       .buildValidatorFactory()
       .getValidator();

The options don't really exclude but amend each other.

## Validating method level constraints

### Extensions to the `j.v.Validator` API

The following new methods are suggested on `j.v.Validator` (see [GitHub](https://github.com/gunnarmorling/beanvalidation-api/blob/BVAL-244/src/main/java/javax/validation/Validator.java)): 

	<T> Set<MethodConstraintViolation<T>> validateParameter(
		T object, Method method, Object parameterValue, int parameterIndex, Class<?>... groups);

	<T> Set<MethodConstraintViolation<T>> validateAllParameters(
		T object, Method method, Object[] parameterValues, Class<?>... groups);

	<T> Set<MethodConstraintViolation<T>> validateReturnValue(
		T object, Method method, Object returnValue, Class<?>... groups);

	<T> Set<MethodConstraintViolation<T>> validateConstructorParameter(
		T object, Constructor<T> constructor, Object parameterValue, int parameterIndex, Class<?>... groups);

	<T> Set<MethodConstraintViolation<T>> validateAllConstructorParameters(
		T object, Constructor<T> constructor, , Object[] parameterValues, Class<?>... groups);

### MethodConstraintViolation

### Triggering validation

It's important to understand that BV itself doesn't trigger the evaluation of any method level constraints. That is, just annotating any methods with parameter or return value constraints doesn't automatically enforce these constraints, just as annotating any fields or properties with bean constraints does enforce these either.

Instead method level constraints must be validated by invoking the appropriate methods on `j.v.Validator`. This typically happens using approaches and techniques such as:

* CDI/EJB interceptors
* aspect-oriented programming
* Java proxies

#### Validating invariants

## Extensions to the meta-data API

## Extensions to the XML schema for constraint mappings
