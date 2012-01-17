---
title: Support for method validation
layout: default
author: Gunnar Morling
---

# #{page.title}

[Link to JIRA ticket](https://hibernate.onjira.com/browse/BVAL-241)

## Goal

As proposed in BV 1.0 Appendix C, the notion of method validation shall be established. This will allow to use the Bean Validation API for programming by contract.

## Defining method level constraints

Note: In the following, the term "method level" constraint refers to constraints declared on methods as well as constructors.

Method constraints are defined by ading constraint annotations to method or constructor parameters (parameter constraints) or methods (return value constraints). As with bean constraints, this can happen using either actual Java annotations or using an XML constraint mapping file (see ...). BV providers are free to provide additional means of defining method constraints such as an API-based aproach.

### Requirements on methods to be validated

Methods which shall be annotated with parameter or return value constraints must be non-static. There is no restriction with respect to the visibility of validated methods from the perspective of this specification, but it's possible that certain technologies integrating with method validation (see ...) support only the validation of methods with certain visibilities.

### Defining parameter constraints
 
Parameter constraints are defined by simply annotating method or constructor parameters:

An example:

	public class OrderService {

		public OrderService(@NotNull CreditCardProcessor creditCardProcessor) {
			//...
		}

		public void placeOrder(@NotNull @Size(min=3, max=20) String customerCode, @NotNull Item item, @Min(1) quantity) {
			//...
		}
	}

Here the following preconditions are defined which must be satisfied in order to legally invoke the methods of the `OrderService` class:

* The `CreditCardProcessor` passed to the constructor must not be null.
* The customer code passed to the `placeOrder()` method must not be null and must be between 3 and 20 characters long.
* The `Item` passed to the `placeOrder()` method must not be null.
* The quantity passed to the `placeOrder()` method must be 1 at least.

Tip: In order to use constraint annotations for method parameters, their element type must be `ELEMENTTYPE.METHOD`. All built-in constraints support this element type and it's considered a best practice to do the same for custom constraints also if they are not primarily intended to be used as parameter constraints.

#### Cross-parameter constraints

#### Naming parameters

If the validation of a parameter constraint fails the concerned parameter needs to be identified in the resulting `MethodConstraintViolation` (see ...).

Java doesn't provide a portable way to retrieve parameter names. Bean Validation therefore defines the `ParameterNameProvider` API to which the retrieval of parameter names is delegated:

	public interface ParameterNameProvider {

		String[] getParameterNames(Constructor< ? > constructor) throws ValidationException;

		String[] getParameterNames(Method method) throws ValidationException;
	}

A conforming BV implementation provides a default `ParameterNameProvider` implementation which returns parameter names in the form `arg<PARAMETER_INDEX>`, where `PARAMETER_INDEX` starts at 0 for the first parameter, e.g. `arg0`, `arg1`etc.

BV providers are free to provide additional implementations (e.g. based on annotations specifying parameter names, debug symbols etc.). If a user wishes to use another parameter name provider than the default implementation, she may specify the provider to use with help of the bootstrap API (see ...) or the XML configuration (see ...).

### Defining return value constraints

TODO: Should property constraints (on getter methods) also be handled as method constraint?

### Marking parameters/return values for cascaded validation

### Inheritance hierarchies

When defining method level constraints within inheritance hierarchies (that is, class inheritance by extending base classes and interface inheritance by implementing interfaces) one generally has to deal with the [Liskov substitution principle](http://en.wikipedia.org/wiki/Liskov_substitution_principle) which mandates that

* a methods preconditions (as represented by parameter constraints) may not be strengthened in sub types
* a methods postconditions (as represented by return value constraints) may not be weakened in sub types

#### Option 1: Don't allow allow refinement

#### Option 2: allow return value refinement (AND style)

#### Option 3: allow param (OR style) and return value (AND style) refinement

## Validating method level constraints

As standard bean constraints method level constraints are evaluated using the `javax.validation.Validator` API.

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
		T object, Constructor<T> constructor, Object[] parameterValues, Class<?>... groups);

TODO: Add these methods to the listing in section 4.1

### Methods for method level validation (to become 4.1.2)

The method `<T> Set<MethodConstraintViolation<T>> validateParameter(T object, Method method, Object parameterValue, int parameterIndex, Class<?>... groups)` validates the value (identified by `parameterValue`) for a single method parameter (identified by `method` and `parameterIndex`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.

The method `<T> Set<MethodConstraintViolation<T>> validateAllParameters(T object, Method method, Object[] parameterValues, Class<?>... groups);` validates the arguments (as given in `parameterValues`) for the parameters of a given method (identified by `method`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.

The method `<T> Set<MethodConstraintViolation<T>> validateConstructorParameter(T object, Constructor<T> constructor, Object parameterValue, int parameterIndex, Class<?>... groups);` validates the value (identified by `parameterValue`) for a single method parameter (identified by `constructor` and `parameterIndex`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.

The method `<T> Set<MethodConstraintViolation<T>> validateAllConstructorParameters(T object, Constructor<T> constructor, Object[] parameterValues, Class<?>... groups);` validates the arguments (as given in `parameterValues`) for the parameters of a given constructor (identified by `constructor`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.

The method `<T> Set<MethodConstraintViolation<T>> validateReturnValue(T object, Method method, Object returnValue, Class<?>... groups);` validates the return value (specified by `returnValue`) of a given method (identified by `method`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.


#### Examples

### MethodConstraintViolation (to become 4.3)

`MethodConstraintViolation` is the class describing a single method constraint failure. A (possibly empty) set of `MethodConstraintViolation` is returned for a method validation.
	
	/**
	 * Describes the violation of a method-level constraint by providing access
	 * to the method/parameter hosting the violated constraint etc.
	 *
	 * @author Gunnar Morling
	 */
	public interface MethodConstraintViolation<T> extends ConstraintViolation<T> {
	
		/**
		 * The kind of a {@link MethodConstraintViolation}.
		 *
		 * @author Gunnar Morling
		 */
		public static enum Kind {

			/**
			 * Identifies constraint violations occurred during the validation of a
			 * constructor parameter.
			 */
			CONSTRUCTOR_PARAMETER,
			
			/**
			 * Identifies constraint violations occurred during the validation of a
			 * method parameter.
			 */
			METHOD_PARAMETER,
	
			/**
			 * Identifies constraint violations occurred during the validation of a
			 * method's return value.
			 */
			RETURN_VALUE
		}
	
		/**
		 * Returns the method during which's validation this constraint violation
		 * occurred.
		 *
		 * @return The method during which's validation this constraint violation
		 *         occurred.
		 */
		Method getMethod();
	
		/**
		 * Returns the index of the parameter holding the constraint which caused
		 * this constraint violation.
		 *
		 * @return The index of the parameter holding the constraint which caused
		 *         this constraint violation or null if this constraint violation is
		 *         not of {@link Kind#PARAMETER}.
		 */
		Integer getParameterIndex();
	
		/**
		 * <p>
		 * Returns the name of the parameter holding the constraint which caused
		 * this constraint violation.
		 * </p>
		 * <p>
		 * Currently a synthetic name following the form <code>"arg" + index</code>
		 * will be returned, e.g. <code>"arg0"</code>. In future versions it might
		 * be possible to specify real parameter names, e.g. using an
		 * annotation-based approach around <code>javax.inject.Named</code>.
		 * </p>
		 *
		 * @return The name of the parameter holding the constraint which caused
		 *         this constraint violation or null if this constraint violation is
		 *         not of {@link Kind#PARAMETER}.
		 */
		String getParameterName();
	
		/**
		 * Returns the kind of this method constraint violation.
		 *
		 * @return The kind of this method constraint violation.
		 */
		Kind getKind();
	}

The `getMethod()` method returns a `java.lang.reflect.Method` object representing the method hosting the violated constraint.

Alternative: Should this be the invoked method? Or do we need both?

The `getParameterIndex()` method returns the index of the parameter hosting the violated constraint in case it is a parameter constraint, otherwise `null`.

The `getParameterName()` method returns the name of the parameter hosting the violated constraint in case it is a parameter constraint, otherwise `null`. The returned name will be determined by the `ParameterNameProvider` used by the current validator (see ...).

The `getKind()` method returns the `Kind` of the constraint violation, which can either be `Kind.CONSTRUCTOR_PARAMETER`, `Kind.METHOD_PARAMETER` or `Kind.RETURN_VALUE`.

TODO: describe behavior of getPropertyPath() (and getRootBean()) as inherited from `ConstraintViolation`)

#### Examples

### Triggering validation

It's important to understand that BV itself doesn't trigger the evaluation of any method level constraints. That is, just annotating any methods with parameter or return value constraints doesn't automatically enforce these constraints, just as annotating any fields or properties with bean constraints does enforce these either.

Instead method level constraints must be validated by invoking the appropriate methods on `j.v.Validator`. This typically happens using approaches and techniques such as:

* CDI/EJB interceptors
* aspect-oriented programming
* Java proxies

#### Options

From Emmanuel's mail:

* reuse @Valid or have a new one
* let each integrator use a specific annotation or propose a BV one
* put it on the method validated (a global set of groups per method) or on a per parameter

### Validating invariants

## Extensions to the meta-data API

## Extensions to the XML schema for constraint mappings

## MethodConstraintViolationException (to become 8.2)

The method validation mechanism is typically not invoked manually during normal program execution, but rather automatically using a proxy, method interceptor or similar. Typically the program flow shouldn't continue its normal execution in case a parameter or return value constraint is violated which is realized by throwing an exception. 

Bean Validation provides a reference exception for such cases. Frameworks and applications are encouraged to use `MethodConstraintViolationException` as opposed to a custom exception to increase consistency of the Java platform.

	/**
	 * Exception class to be thrown by integrators of the BV method validation feature.
	 *
	 * @author Gunnar Morling
	 */
	public class MethodConstraintViolationException extends ValidationException {
	
		private static final long serialVersionUID = 5694703022614920634L;
		
		private final Set<MethodConstraintViolation<?>> constraintViolations;
		
		/**
		 * Creates a new {@link MethodConstraintViolationException}.
		 *
		 * @param constraintViolations A set of constraint violations for which this exception shall be created.
		 */
		public MethodConstraintViolationException(Set<? extends MethodConstraintViolation<?>> constraintViolations) {
		
			this( null, constraintViolations );
		}
		
		/**
		 * Creates a new {@link MethodConstraintViolationException}.
		 *
		 * @param message The message for the exception to be created.
		 * @param constraintViolations A set of constraint violations for which this exception shall be created.
		*/
		public MethodConstraintViolationException(String message,
			Set<? extends MethodConstraintViolation<?>> constraintViolations) {
		
			super( message );
			this.constraintViolations = constraintViolations == null ? 
				Collections.<MethodConstraintViolation<?>>emptySet() : 
				Collections .unmodifiableSet( constraintViolations );
		}
		
		/**
		 * Returns the set of constraint violations reported during a validation.
		 *
		 * @return An unmodifiable set of {@link MethodConstraintViolation}s occurred during a method level validation call.
		 */
		public Set<MethodConstraintViolation<?>> getConstraintViolations() {
			return constraintViolations;
		}
	
	}

## Required changes in 1.0 wording

* section [2.1](http://beanvalidation.org/1.0/spec/#constraintsdefinitionimplementation-constraintdefinition): `ElementType.PARAMETER` should be mandatory now
* section [3.1.2](http://beanvalidation.org/1.0/spec/#constraintdeclarationvalidationprocess-requirements-property): Remove sentence "Constraints on non getter methods are not supported."

## Misc.

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
		public int getBar() { return bar; }
	}

#### Option 3: An option in the configuration API:

	Validator validator = Validation.byDefaultProvider()
	       .configure()
	       .applyPropertyConstraintsToSetters()
	       .buildValidatorFactory()
	       .getValidator();

The options don't really exclude but amend each other.