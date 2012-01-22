---
title: Support for method validation
layout: default
author: Gunnar Morling
---

# #{page.title}

[Link to JIRA ticket](https://hibernate.onjira.com/browse/BVAL-241)

## Contents

* [Goal](#Goal)
* [Declaring method level constraints (to become 3.5?)](#method_level)
    * [Requirements on methods to be validated](#requirements)
    * [Defining parameter constraints](#parameter)
        * [Cross-parameter constraints](#cross_parameter)
        * [Naming parameters](#naming)
    * [Defining return value constraints](#return_value)
    * [Marking parameters and return values for cascaded validation](#cascaded)
    * [Inheritance hierarchies](#inheritance)
        * [Option 1: Don't allow allow refinement](#no_refinement)
        * [Option 2: allow return value refinement (AND style)](#return_refinement)
        * [Option 3: allow param (OR style) and return value (AND style) refinement](#parameter_refinement)
* [Validating method level constraints](#validating)
    * [MethodConstraintViolation (to become 4.3)](#method_constraint_violation)
    * [Triggering validation](#triggering)
        * [Option 1: reuse @Valid or have a new one](#at_valid)
        * [Option 2: let each integrator use a specific annotation or propose a BV one](#specific)
        * [Option 3: put it on the method validated (a global set of groups per method) or on a per parameter](on_method)
* [Extensions to the meta-data API](#meta_data)
* [Extensions to the XML schema for constraint mappings](#xml)
    * [MethodConstraintViolationException (to become 8.2)](#mcve)
* [Required changes in 1.0 wording](#changes)
* [Misc.](#misc)
    * [Validating invariants](#invariants)
    * [Applying property constraints to setter methods](#setters)
        * [Option 1: A class-level annotation](#class)
        * [Option 2: A property level annotation](#property)
        * [Option 3: An option in the configuration API](#option)

## Goal <a id="Goal"/>

In addition to the validation of JavaBeans, the Bean Validation API also allows to declare and validate constraints on a method level. This allows to use the Bean Validation API for a programming style known as "programming by contract". More specifically this means that any Bean Validation constraints can be used to describe

* the preconditions that must be met before a method may legally be invoked and
* postconditions that are guaranteed after a method invocation returns.

Compared to traditional means of checking the sanity of a method's argument values (within the method implementation) and its return value (by the caller after invoking the method) this approach has several advantages:

* These checks don't have to be performed manually, which results in less code to write and maintain.
* The pre- and postconditions applying for a method don't have to be expressed again in the method's JavaDoc, since any of it's annotations will automatically be included in the generated JavaDoc. This means less redundancy which reduces the chance of inconsistencies between implementation and comments.

## Declaring method level constraints (to become 3.5?) <a id="method_level"/>

Note: In the following, the term "method level constraint" refers to constraints declared on methods as well as constructors.

Method constraints are defined by adding constraint annotations to method or constructor parameters (parameter constraints) or methods (return value constraints). 

As with bean constraints, this can happen using either actual Java annotations or using an XML constraint mapping file (see ...). BV providers are free to provide additional means of defining method constraints such as an API-based aproach.

### Requirements on methods to be validated <a id="requirements"/>

Methods which shall be annotated with parameter or return value constraints must be non-static. 

There is no restriction with respect to the visibility of validated methods from the perspective of this specification, but it's possible that certain technologies integrating with method validation (see ...) support only the validation of methods with certain visibilities.

### Defining parameter constraints <a id="parameter"/>
 
Parameter constraints are defined by putting constraint annotations to method or constructor parameters.

Example xy: Declaring parameter constraints

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

Note that declaring these constraints does not automatically cause their validation when the concerned methods are invoked. It's the responsibility of an integration layer to trigger the validation of the constraints using a method interceptor, dynamic proxy or similar. See chapter ... for more details.

Tip: In order to use constraint annotations for method parameters, their element type must be `ELEMENTTYPE.METHOD`. All built-in constraints support this element type and it's considered a best practice to do the same for custom constraints also if they are not primarily intended to be used as parameter constraints.

#### Cross-parameter constraints ([BVAL-232](https://hibernate.onjira.com/browse/BVAL-232)) <a id="cross_parameter"/>

DISCUSSION: There are several options for implementing cross-parameter constraints. I feel rather unsure about which one to pursue, likely I'd prefer to provide #3 and #4. #2 seems obvious at first but has actually more disadvantages compared to #3.

##### Option 1: Don't support cross-parameter constraints

* Pro: Wait for actual user demand, let BV providers come up with specific solutions, see what works out best
* Con: Needs BV providers to come up with specific solutions ;-)

##### Option 2: New interface `MethodConstraintValidator`

We could have a new interface `MethodConstraintValidator` which gets the parameters passed as `Object[]` to the `isValid()` method:

	/**
	 * Defines the logic to validate a given cross-parameter method level constraint A.
	 *
	 * @author Gunnar Morling
	 */
	public interface MethodConstraintValidator<A extends Annotation> {

		/**
		 * Initializes the validator in preparation for isValid calls.
		 * The constraint annotation for a given constraint declaration
		 * is passed.
		 * <p/>
		 * This method is guaranteed to be called before any use of this instance for
		 * validation.
		 *
		 * @param constraintAnnotation annotation instance for a given constraint declaration
		 */
		void initialize(A constraintAnnotation);

		/**
		 * Implement the validation logic.
		 * The state of <code>value</code> must not be altered.
		 *
		 * This method can be accessed concurrently, thread-safety must be ensured
		 * by the implementation.
		 *
		 * @param parameterValues The parameter values to be validated.
		 * @param context context in which the constraint is evaluated
		 *
		 * @return false if <code>value</code> does not pass the constraint
		 */
		boolean isValid(Object[] parameterValues, ConstraintValidatorContext context);
	}

Example:

	public class ReservationService {

		@DateParameterCheck
		void bookHotel(@NotNull Customer customer, @NotNull Date from, @NotNull Date to) {
			//...
		}
	}

	public class DateParameterCheckValidator implements MethodConstraintValidator<DateParameterCheck> {

		@Override
		public void initialize(DateParameterCheck constraint) {}

		@Override
		public boolean isValid(Object[] parameterValues, ConstraintValidatorContext context) {
			if(parameterValues[1] == null || parameterValues[2] == null) {
				return true;
			}

			return ((Date)parameterValues[1]).before((Date)parameterValues[2]);
		}
}

* Pro: Rathers straight-forward
* Con: Not that type-safe: Fiddling with object array, casts etc. required. How to avoid using constraints at methods with wrong signature? This will just fail upon invoking `isValid()`.

##### Option 3: Invoke validator methods by signature matching

Instead of having a static `isValid()` method, one could be invoked by signature matching. We would have an initialization contract:

	public interface Initializable<A extends Annotation> {

		void initialize(A constraintAnnotation);

	}

Validator implementations must define a matching `isValid()` method per supported signature:

	public class ReservationService {

		@DateParameterCheck
		void bookHotel(@NotNull Customer customer, @NotNull Date from, @NotNull Date to) {
			//...
		}

		@DateParameterCheck //from must be before to AND alternativeTo
		void bookHotel(@NotNull Customer customer, @NotNull Date from, @NotNull Date to, @NotNull Date alternativeTo) {
			//...
		}
	}

	public class DateParameterCheckValidator implements Initializable<DateParameterCheck> {

		@Override
		public void initialize(DateParameterCheck constraint) {}

		public boolean isValid(Customer customer, Date from, Date to, ConstraintValidatorContext context) {
			if(from == null || to == null) {
				return true;
			}
			return from.before(to);
		}

		public boolean isValid(Customer customer, Date from, Date to, Date alternativeTo, ConstraintValidatorContext context) {
			if(from == null || to == null || alternativeTo == null) {
				return true;
			}
			return from.before(to) && from.before(alternativeTo);
		}
	}

* Pro: Implementation of validators much simpler to write and read (no casts required)
* Pro: Allows BV providers to check for matching validators and fail if none exists (exception raised by BV and NOT in validator)
* Con: `isValid()` method can't be defined in validator interface
* Con: Not refactoring safe (changing signatures), but option #2 isn't as well)

##### Option 4: Have a script based approach:

We might define a special script based constraint:

	public class ReservationService {

		@ParameterAssert(script="arg1.before(arg2)", lang="javascript")
		void bookHotel(@NotNull Customer customer, @NotNull Date from, @NotNull Date to) {
			//...
		}
	}

* Pro: Good to read and write
* Con: Not type safe

Parameter names would be retrieved via the `ParameterNameProvider` (see next section).

Probably basic constraints (`@NotNull` etc.) should be checked beforehand in order to allow for concise script expressions without redundant null checks.

#### Naming parameters <a id="naming"/>

If the validation of a parameter constraint fails the concerned parameter needs to be identified in the resulting `MethodConstraintViolation` (see ...).

Java doesn't provide a portable way to retrieve parameter names. Bean Validation therefore defines the `ParameterNameProvider` API to which the retrieval of parameter names is delegated:

	public interface ParameterNameProvider {

		String[] getParameterNames(Constructor< ? > constructor) throws ValidationException;

		String[] getParameterNames(Method method) throws ValidationException;
	}

A conforming BV implementation provides a default `ParameterNameProvider` implementation which returns parameter names in the form `arg<PARAMETER_INDEX>`, where `PARAMETER_INDEX` starts at 0 for the first parameter, e.g. `arg0`, `arg1`etc.

BV providers are free to provide additional implementations (e.g. based on annotations specifying parameter names, debug symbols etc.). If a user wishes to use another parameter name provider than the default implementation, she may specify the provider to use with help of the bootstrap API (see ...) or the XML configuration (see ...).

### Defining return value constraints <a id="return_value"/>

Return value constraints are defined by putting constraint annotations directly to the method itself.

Example xy: Declaring return value constraints

	public class OrderService {

		@NotNull
		@Size(min=1)
		public Set<CreditCardProcessor> getCreditCardProcessors() {
			//...
		}

		@NotNull
		@Future
		public Date getNextAvailableDeliveryDate() {
			//...
		}
	}

Here the following postconditions are defined which are guaranteed to the caller of the methods of the `OrderService` class:

* The set of `CreditCardProcessor` objects returned by `getCreditCardProcessors()` will neither be null nor empty.

* The `Date` object returned by `getNextAvailableDeliveryDate()` will not be null and be in the future.

As with parameter constraints, these return value constraints are not automatically validated upon method invocation but instead an integration layer invoking the validation is required.

DISCUSSION: Should property constraints (on getter methods) also be handled as method constraint?

### Marking parameters and return values for cascaded validation <a id="cascaded"/>

Similar to normal bean validation, the `@Valid` annotation can be used, to declare that a cascaded validation of given method parameters or return values shall be performed by the Bean Validation provider.

Generally the same rules as for standard object graph validation (see 3.5.1) apply, in particular

* null arguments and return values are ignored
* the validation is recursive, that is, if validated parameter or return value objects have references marked with `@Valid` themselves, these references will also be validated
* Bean Validation providers must guarantee the prevention of infinite loops during cascaded validation.

Example xy: Marking parameters and return values for cascaded validation

	public class OrderService {

		@Valid
		public Order getOrderByPk(@NotNull @Valid OrderPK orderPk) {
			//...
		}
	
		@NotNull
		@Valid
		public Set<Order> getOrdersByCustomer(@NotNull @Valid CustomerPK customerPk) {
			//...
		}		
	}

Here the following recursive validations will happen when validating the methods of the `OrderService` class:

* Validation of the constraints on the object passed for the `orderPk` parameter and the returned `Order` object of the `getOrderByPk()` method
* Validation of the constraints on the object passed for the `customerPk` parameter and the constraints on each object contained within the returned `Set<Order>` of the getOrdersByCustomer() method

Again, solely marking parameters and return values for cascaded validation does not trigger the actual validation.

DISCUSSION: There were discussions whether to use `@Valid` or a new annotation such as `@ValidParameter`. 

IMO introducing a new annotation doesn't really make sense, as the `@Valid` annotation is used here in its originally intended sense: marking a (referenced) object for cascaded validation.

### Inheritance hierarchies <a id="inheritance"/>

When defining method level constraints within inheritance hierarchies (that is, class inheritance by extending base classes and interface inheritance by implementing interfaces) one generally has to deal with the [Liskov substitution principle](http://en.wikipedia.org/wiki/Liskov_substitution_principle) which mandates that

* a methods preconditions (as represented by parameter constraints) may not be strengthened in sub types
* a methods postconditions (as represented by return value constraints) may not be weakened in sub types

#### Option 1: Don't allow allow refinement <a id="no_refinement"/>

#### Option 2: allow return value refinement (AND style) <a id="return_refinement"/>

#### Option 3: allow param (OR style) and return value (AND style) refinement <a id="parameter_refinement"/>

## Validating method level constraints <a id="validating"/>

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

### Methods for method level validation (to become 4.1.2) <a id="mfm"/>

The method `<T> Set<MethodConstraintViolation<T>> validateParameter(T object, Method method, Object parameterValue, int parameterIndex, Class<?>... groups)` validates the value (identified by `parameterValue`) for a single method parameter (identified by `method` and `parameterIndex`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.

The method `<T> Set<MethodConstraintViolation<T>> validateAllParameters(T object, Method method, Object[] parameterValues, Class<?>... groups);` validates the arguments (as given in `parameterValues`) for the parameters of a given method (identified by `method`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.

The method `<T> Set<MethodConstraintViolation<T>> validateConstructorParameter(T object, Constructor<T> constructor, Object parameterValue, int parameterIndex, Class<?>... groups);` validates the value (identified by `parameterValue`) for a single method parameter (identified by `constructor` and `parameterIndex`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.

The method `<T> Set<MethodConstraintViolation<T>> validateAllConstructorParameters(T object, Constructor<T> constructor, Object[] parameterValues, Class<?>... groups);` validates the arguments (as given in `parameterValues`) for the parameters of a given constructor (identified by `constructor`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.

The method `<T> Set<MethodConstraintViolation<T>> validateReturnValue(T object, Method method, Object returnValue, Class<?>... groups);` validates the return value (specified by `returnValue`) of a given method (identified by `method`). A `Set` containing all `MethodConstraintViolation` objects representing the failing constraints is returned, an empty `Set` is returned otherwise.


#### Examples

### MethodConstraintViolation (to become 4.3) <a id="method_constraint_violation"/>

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

### Triggering validation <a id="triggering"/>

It's important to understand that BV itself doesn't trigger the evaluation of any method level constraints. That is, just annotating any methods with parameter or return value constraints doesn't automatically enforce these constraints, just as annotating any fields or properties with bean constraints doesn't enforce these either.

Instead method level constraints must be validated by invoking the appropriate methods on `javax.validation.Validator`. This typically happens using approaches and techniques such as:

* CDI/EJB interceptors
* aspect-oriented programming
* Java proxies

Possible options as per Emmanuel's mail:

#### Option 1: reuse @Valid or have a new one <a id="at_valid"/>

#### Option 2: let each integrator use a specific annotation or propose a BV one <a id="specific"/>

#### Option 3: put it on the method validated (a global set of groups per method) or on a per parameter <a id="on_method"/>

## Extensions to the meta-data API <a id="meta_data"/>

## Extensions to the XML schema for constraint mappings <a id="xml"/>

## MethodConstraintViolationException (to become 8.2) <a id="mcve"/>

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

## Required changes in 1.0 wording <a id="changes"/>

* section [2.1](http://beanvalidation.org/1.0/spec/#constraintsdefinitionimplementation-constraintdefinition): `ElementType.PARAMETER` should be mandatory now
* section [3.1.2](http://beanvalidation.org/1.0/spec/#constraintdeclarationvalidationprocess-requirements-property): Remove sentence "Constraints on non getter methods are not supported."

## Misc. <a id="misc"/>

### Validating invariants <a id="invariants"/>

DISCUSSION: Should there be some way to trigger validation of bean constraints upon method invocations?

IMO this falls in the same category as triggering method validation itself and should be handled by integrators, e.g. by defining a interceptor binding annotation for CDI.

### Applying property constraints to setter methods <a id="setters"/>

It might be useful to have the possibility to apply property constraints (defined on getter methods) also as parameter constraints within the corresponding setter methods.

DISCUSSION: Might that be required/helpful by JAX-RS?

#### Option 1: A class-level annotation <a id="class"/>

	@ApplyPropertyConstraintsToSetters
	public class Foo {
	
	}

#### Option 2: A property level annotation <a id="property"/>

	public class Foo {
	
		@ApplyToSetter
		@Min(5)
		public int getBar() { return bar; }
	}

#### Option 3: An option in the configuration API <a id="option"/>

	Validator validator = Validation.byDefaultProvider()
	       .configure()
	       .applyPropertyConstraintsToSetters()
	       .buildValidatorFactory()
	       .getValidator();

The options don't really exclude but amend each other.