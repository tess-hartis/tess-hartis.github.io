User input is unpredictable. Every programmer knows that data should not be
stored without being validated beforehand. While there are many ways of going
about validation, some are more practical than others. Many have encountered (or
used) "torrential exceptions":

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
public static DogName of(string name){

    if(name.isEmpty())
    	throw new IllegalArgumentException
    	("name cannot be empty");

    if(name.length() > 50)
	throw new IllegalArgumentException
	("name too long");

    if(!name.isValid)
	throw new Illegal ArgumentException
	("invalid for another reason");

    //other checks omitted for brevity

    return new DogName.of(name);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Exceptions in this style can be a nuisance. They must be individually handled
throughout the program or via middleware. The code becomes littered with
if-statements, excessive exception handling, and is left with instability and
the looming threat of a stray exception being thrown.

In the case of validating multiple input fields, an exception will halt the
application at the first error. This "one-by-one" style of feedback is
inefficient and irritating. If the user had to input 10 fields and all were
invalid, only one exception message would be visible at a time.

When validating user input, it should be no surprise that the user would provide
invalid data. Invalid data is just as common as valid data - there is nothing
exceptional about it! Exceptions should be reserved for the exceptional.

Imagine that your car regularly overheats while driving and you decide to keep
multiple gallons of water in your car to cool the engine when needed. Is this
solution really easier and safer than dealing with the problem upfront?
Welcoming exceptions and handling them after-the-fact is dangerous.

One of the limits of non-functional programming languages is that methods can
only have one return type. Throwing exceptions when validating data is an
attempt to get around this issue. Either the data will be valid and an object
will be returned, or it will be invalid and an exception will be thrown.

By utilizing the Vavr or LanguageExt libraries, functional programming concepts
can be implemented in Java and C\#, respectively. Both libraries' validation
APIs allow an object to exist in one of two states - valid or invalid. Methods
can return **either** a valid object or some other object to represent the
errors. This could be a string, list of strings, or custom error object. The
result is encapsulated within the validation wrapper, minimizing the risk of an
exception explosion. There are many ways to implement these powerful tools, but
this post will focus on validating user input.

Benefits of using this style of validation greatly outweigh the small learning
curve:

-   Exceptions are no longer needed for handling invalid data, which means
    less cluttered code. Exceptions can go back to handling exceptional cases!

-   Does not require excessive and noisy annotations on the entity model

-   The object is only created after being validated, instead of being created
    and then validated

-   The errors can be collected and returned to the user as one object, instead
    of one-by-one exceptions

 

Implementation Examples
=======================

The following code snippets will demonstrate a few ways to implement validation
using the Vavr and LanguageExt libraries. The examples include a `Dog` entity,
which is created with properties `DogName`, `DogAge`, and `DogBreed`.

**1. Vavr**
-------

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
import io.vavr.control.Validation;
import io.vavr.control.Validation.valid;
import io.vavr.control.Validation.invalid;

public class DogName{

    public String value;

    private DogName(string value){

        this.value = value;
    }

    public static Validation<String, DogName> validate(String name){

        if (name.isEmpty())
                return invalid("name cannot be empty");

	if(name.length() > 50)
        	return invalid("name too long")

	//other checks left out for brevity

        else return valid(new DogName(name));

	}
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the example above, the method returns either a `DogName` or a `String` error
message. You could also validate against a regex pattern, or use pattern
matching, but this post is keeping it simple. By convention, the invalid state
is always on the left side, and the valid state is on the right side.

Assume similar methods were created for the `DogBreed` and `DogAge` properties.
The example below is a POST method from a REST API controller using the Spring
web framework.

Combining validations into one:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity.*;
import io.vavr.control.Validation;
import static io.vavr.API.*;
import static io.vavr.Patterns.*;

@PostMapping
public ResponseEntity create(@RequestBody PostDogDto dto) {

//the individual field validations would ideally NOT be in a controller method body, but would be extracted elsewhere, such as a command. They are included here for ease of reading and understanding.

    var name = DogName.validate(dto.dog_name);
    var breed = DogBreed.validate(dto.dog_breed);
    var age = DogAge.validate(dto.dog_age);

    return Validation.combine(name, breed, dog)
            .ap(Dog::of)

//then do something to access the result
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `name`, `breed`, and `age` variables return wrapped validations. The power
of Vavr's Validation API shines with the `Validation.combine()` method. After
combining multiple wrapped validations, the `.ap()` method assesses them all in
parallel and returns a final wrapped validation as the result. In functional
programming, this is called the applicative.

The final validation is valid if each previous validation is valid, or invalid
if at least one previous validation is invalid. The most important thing here is
that no exceptions are thrown, and all of the errors are collected without
stopping the flow of the code.

The final step is accessing the result, which is wrapped inside a validation,
and then defining appropriate behavior for each case. A simple, yet highly
verbose, way of doing this could be with an if-statement:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
var result = Validation.combine(name, breed, dog).ap(Dog::of)

if(result.isValid()){
    result.map(sessionRepository::saveAndFlush);
    return new ResponseEntity<>(HttpStatus.CREATED);
}

else {
    result.mapError(Value::toJavaList);
    return new ResponseEntity<>(HttpStatus.UNPROCESSABLE_ENTITY);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 

Another way of checking the results is to use pattern matching. An issue with
this method is that the success and error cases are mapped twice: once in the
`result` variable and once in the return statement via the lambda expressions:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
var result = Validation.combine(name, breed, dog)
	            .ap(Dog::of)
	            .map(sessionRepository::saveAndFlush)
		        .mapError(Value::toJavaList);

return Match(result).of(
    Case($Valid($()),() ->
          new ResponseEntity<>(HttpStatus.CREATED)),
    Case($Invalid($()), e ->
	  new ResponseEntity<>(e,HttpStatus.BAD_REQUEST)));
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 

A more concise way of achieving the same result without the redundancy is to
utilize the `.fold()` method. Using fold allows us to map both the invalid and
valid cases onto a `ResponseEntity` all in one short statement. Although not
shown below, this method could be condensed even further if
`Validation.combine()` and `.ap()` were extracted to a command:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
return Validation.combine(name, description, length)
        .ap(Session::of)
        .fold(e -> unprocessableEntity().body(e.toJavaList()),
              c -> ok(sessionRepository.saveAndFlush(c)));
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 

**2. LanguageExt**
--------------

Implementing functional validation via the LanguageExt library is very similar
to using Vavr. However, there are quite a few syntactic differences.

Creating a simple validation object:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
using LanguageExt;
using static LanguageExt.Prelude;
using LanguageExt.Common;

public record DogName
{

    public readonly string Value;

    private DogName(string value)
    {
        Value = value;
    }

    public static Validation<Error, DogName> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Fail<Error, DogName>("name cannot be empty");

        if (value.Length > 50)
            return Fail<Error, DogName>("name is too long");

        return Success<Error, DogName>(new DogName(value));
    }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One thing to note is that LanguageExt comes with a built-in `Error` object. This
is beneficial because you do not need to build your own! However, the syntax for
returning valid or invalid from a conditional is not as elegant as in Vavr.
Instead of merely returning `valid` or `invalid`, you must explicitly
specify `Success<Error, A>` or `Fail<Error, A>`.

Assume all the other dog properties were created using the same validation
technique. This example illustrates a combination of those three validations
within the CreateDogCommand:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
using LanguageExt;
using LanguageExt.Common;
using static LanguageExt.Prelude;

//inject repository interface instance

public async Task<Validation<Error, Dog>> Handle(CreateDogCommand command, CancellationToken cancellationToken)
{
    var name = DogName.Create(command.Name);
    var breed = DogBreed.Create(command.Breed);
    var age = DogAge.Create(command.Age);

    var dog = (name, breed, age).Apply(Dog.Create);

    ignore(dog.Map(async x => await _dogRepository.Add(x)));
    return dog;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There is lots to unpack in the example above, since it is a command and also an
async method. However, under the extraneous details, it functions like Vavr. The
validations are collected and processed using the `.Apply` method. Like `.ap()`,
this method unwraps the validations, then returns a separate wrapped validation
result. If any of the previous validations are a `Fail`, the resulting
validation will also be a `Fail`.

Inside the `ignore` method, the successful dog object is mapped and added via
the dog repository. `Ignore` signals that anything other than the successful
case (the dog object) should be ignored. The method returns a validation object
of **either** an error or dog.

The last example handles the validation result within a POST method using
ASP.NET:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
[HttpPost]
public async Task<IActionResult> PostDog([FromBody] CreateDogCommand request)
{
 var dog = await _mediator.Send(request);
    return dog.Match<IActionResult>(
        d => Ok(),
        e =>
        {
            var errors = e.Select(x => x.Message).ToList();
            return UnprocessableEntity(new {errors});
        });
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pattern matching is used to handle each validation case. The successful case
returns an OK response. The error case maps the messages to a list and returns
an UnprocessableEntity response. Although not as concise as Vavr's fold method,
it achieves the same result.

Conclusion
==========

Vavr and LanguageExt provide powerful fuctional tools for object-oriented
programming. The validation APIs enable methods or objects to exist in more than
one state - valid or invalid - eliminating the need for handling validation via
exceptions. This allows for cleaner and more concise code. Greatest of all, the
errors can be collected and returned to the user as one object.

The validation APIs of these libraries are just the tip of the iceberg. Learn
more about their amazing capabilities here:

-   [Vavr](https://docs.vavr.io/)

-   [LanguageExt](https://languageext.readthedocs.io/en/latest/README.html)

 
