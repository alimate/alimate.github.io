---
layout: post
title: "RESTful API Design: How to handle errors?"
permalink: /blog/2016/9/24/rest-api-error-handling
excerpt: "We, as software developers, like to only consider happy paths in our scenarios and consequently, tend to forget the fact
that Error Happens, even more than those ordinary happy cases ..."
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2016-9-24-rest-api-error-codes.md"
---

# Do care about your clients
---
"*We all do*". The well known first reaction of all software developers when they first come to this sentence. But,
of course, not when we're going to handle errors.<br>
We, as software developers, like to only consider happy paths in our scenarios and consequently, tend to forget the fact
that *Error Happens*, even more than those ordinary happy cases. Honestly, I can't even remember when was the last time I got an ok
response from a payment API, those are *fail by default*.<br>
How did you handle errors while designing and implementing your very last REST API? How did you handle validation errors? How did you deal with uncaught exceptions in your services? If you didn't answer these questions before, no it's time to consider answering them before designing a new REST API.<br>
Of course, The most well known and unfortunate approach is *let the damn exception propagates* until our beloved client sees the
beautiful stacktrace on her client! If she can't handle our `NullPointerException`, she shouldn't call herself a developer.

# Forget your platform: It's an *Integration Style*
----
In their amazing book, [Enterprise Integration Patterns][1], Gregor Hohpe and Bobby Woolf described one of the most important aspects of applications as the following:

> Interesting applications rarely live in isolation. Whether your sales application must interface
> with your inventory application, your procurement application must connect to an auction site,
> or your PDA’s PIM must synchronize with the corporate calendar server, it seems like any
> application can be made better by integrating it with other applications.

Then they introduced four *Integration Styles*. Regardless of the fact that the book was mainly about *Messaging Patterns*, we're going to focus on *Remote Procedure Invocation*. Remote Procedure Invocation is an umbrella term for all approaches that expose some of application
functionalities through a *Remote Interface*, like REST and RPC. <br>
Now that we've established that REST is an Integration Style:

> **Do Not** expose your platform specific conventions into the REST API

The whole point of using an Integration Style, like REST, is to let different applications developed in different platforms work together,
hopefully in a more seamless and less painful fashion. The last thing a python client developer wanna see is a bunch of *Pascal Case URLs*, just because the API was developed with C#. Don't shove all those *Beans* into your message bodies just because you're using Java.<br>
For a moment, forget about the platform and strive to provide an API aligned with mainstream approaches in REST API design, like the
conventions provided by [Zalando team][2].

# Use HTTP status codes until you bleed!
----
Let's start with a simple rule:

{% highlight http %}
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Wed, 31 Aug 2016 20:51:25 GMT

{"status": "failed"}
{% endhighlight %}
> **Never ever** convey error information in 2xx and 3xx response bodies!

Many so-called RESTful services out there are using HTTP status codes incorrectly. One fundamentally wrong approach is to convey
error information through a `200 OK` response. The other common and irresponsible approach is to return the stacktrace as part of `500 Internal Server Error` body:<br>

{% highlight http %}
HTTP/1.1 500 Internal Server Error
Content-Type: text/plain
Content-Length: 209
Date: Sun, 18 Sep 2016 09:03:48 GMT

JAXBException occurred : unexpected element (uri:"", local:"LoginRequests").
Expected elements are <{}LoginRequest>. unexpected element
(uri:"", local:"LoginRequests"). Expected elements are <{}LoginRequest>.
{% endhighlight %}
Here is another simple rule to follow *unconditionally*:

> **Say goodbye** to your stacktrace!

There is a handful of error status codes available on `4xx` and `5xx` ranges that you can use to signal client or server errors. For example, suppose we have a *Geeks API* and we can create a new geek using `POST` on `/geeks` endpoint. If client didn't send the required parameters,
we can tell her about the error using a `400 Bad Request` response. If authentication is required but client did not send any authentication parameters in her request, we can respond with a `401 Unauthorized` response. <br>
Other typical error responses may use `403 Forbidden`, `404 Not Found`, `429 Too Many Requests`, `500 Internal Server Error` and `503 Service Unavailable`. Of course these are not all available status codes to use, for an exhaustive list of them, consider reading [this Wikipedia entry][status-codes-wiki].

# Not enough information?
----
Even when we're using HTTP status codes to indicate client or server errors, there may be some cases that we need to provide more information about the error. Sometimes we use the same status code to respond for multiple error cases. For example, the `POST /geeks` endpoint may return `400 Bad Request` for both validation errors and when that geek already exists. In such scenarios, How client would be able to tell those errors apart?<br>
One common solution for this problem is to use **Application Level Error Codes**. These are codes you define to indicate one and only one error case. For example, you may reserve error code `42` to say the geek already exists:
{% highlight json %}
{
  "status_code": 400,
  "reason_phrase": "Bad Request",
  "error_code": 42
}
{% endhighlight %}
`status_code` and `reason_phrase` fields would come handy when someone is testing the API via browser. Anyway, clients can find out what exactly went wrong just by inspecting the `error_code` field.<br>
Using a numeric error code is the most common approach to implement application level error codes. However,  selecting the error code is a one of the main challenges of this approach. If we assign our error codes with no strategy and upfront thinking whatsoever, we may end up with a error code mess. One solution to this challenge is to define *Ranges* of error codes. For example, if our fictional REST service has two resources, we can define three ranges like the following:

 - 1 to 50 for general error codes, e.g. `1` for invalid JSON payload
 - 51 to 100 for Geeks API (under `/geeks`), e.g. `51` for geek already exist
 - 101 to 150 for Technologies API (under `/technologies`)

Since different resources aren't equally error prone, we may assign different range sizes to different resources. Estimating each range size is the other challenge we should thought through while designing the API.<br>
The other approach which I tend to prefer to the numeric error codes is **Resource Based Error Codes**. That is, we use a string prefix for each error code and that prefix is determined by the resource itself. For example:

 - All general errors are prefixed with `gen-`, e.g. `gen-1` for invalid JSON payload
 - Geeks API errors are prefixed with `geeks-`, e.g. `geeks-1` for geek already exists
 - Technologies API errors are prefixed with `techs-`, e.g. `techs-1` for whatever

This way our error codes are a little more verbose but wouldn't have those mentioned challenges:
{% highlight json %}
{
  "status_code": 400,
  "reason_phrase": "Bad Request",
  "error_code": "geeks-1"
}
{% endhighlight %}

# Be more expressive!
---
What is your first reaction when you see the following error response?
{% highlight json %}
{
  "status_code": 400,
  "reason_phrase": "Bad Request",
  "error_code": "geeks-1"
}
{% endhighlight %}
I bet you would search the API documentation (if any!) like a clueless chicken to find out what the heck this `geeks-1` code means? Clients would have a nicer API experience if we provide an error message corresponding to the error code:
{% highlight json %}
{
  "status_code": 400,
  "reason_phrase": "Bad Request",
  "error_code": "geeks-1",
  "error_message": "The geek already exists"
}
{% endhighlight %}
If you live in a multilingual context, you could go one step further by changing the `error_message` language through *Content Negotiation* process. For example, if the client set the `Accept-Language` header to `fr-FR`, the error could be ([Google Translate says so][google-translate-en-fr]):
{% highlight json %}
{
  "status_code": 400,
  "reason_phrase": "Bad Request",
  "error_code": "geeks-1",
  "error_message": "Le connaisseur existe déjà"
}
{% endhighlight %}

# How about validation errors?
---
There is this possibility that client has more than one validation error in her request. With our current error response schema, we can only report one error with each http response. That is, in multiple validation errors scenario, first we complain about the first validation error, then the second error and so on. If we could report all validation errors at once, in addition to have a more pleasant API, we could save some extra round trips.<br>
Anyway, we can refactor our error response model to return an array of errors (surprise!) instead of just one error:<br>
{% highlight json %}
{
  "status_code": 400,
  "reason_phrase": "Bad Request",
  "errors": [
      {"code": "geeks-2", "message": "The first_name is mandatory"},
      {"code": "geeks-3", "message": "The last_name is mandatory"}
  ]
}
{% endhighlight %}

# "Talk is cheap, show me the code"
----
Here I'm gonna provide a very minimal implementation of what we've talked about so far with Spring Boot. All the following codes are available at [github][github-source], you can skip rest of the article and check them out now. Anyway, Let's start with error codes, The `ErrorCode` interface will act as a super-type for all our error codes:<br>
{% highlight java %}
/**
 * Represents API error code. Each API should implement this interface to
 * provide an error code for each error case.
 *
 * @implNote Enum implementations are good fit for this scenario.
 *
 * @author Ali Dehghani
 */
public interface ErrorCode {
    String ERROR_CODE_FOR_UNKNOWN_ERROR = "unknown";

    /**
     * Represents the error code.
     *
     * @return The resource based error code
     */
    String code();

    /**
     * The corresponding HTTP status for the given error code
     *
     * @return Corresponding HTTP status code, e.g. 400 Bad Request for a validation
     * error code
     */
    HttpStatus httpStatus();

    /**
     * Default implementation representing the Unknown Error Code. When the
     * {@linkplain ErrorCodes} couldn't find any appropriate
     * {@linkplain ErrorCode} for any given {@linkplain Exception}, it will
     * use this implementation by default.
     */
    enum UnknownErrorCode implements ErrorCode {
        INSTANCE;

        @Override
        public String code() {
            return ERROR_CODE_FOR_UNKNOWN_ERROR;
        }

        @Override
        public HttpStatus httpStatus() {
            return HttpStatus.INTERNAL_SERVER_ERROR;
        }
    }
}
{% endhighlight %}
Each error code has a `String` based `code` and a `HttpStatus` corresponding to that code. There is a default implementation of this interface embedded into the interface itself called `UnknownErrorCode`. This implementation will be used when we couldn't figure out what exactly went wrong in the server, which obviously is a server error.<br>
As I said each resource should provide an implementation of this interface to define all its possible error codes. For example, Geeks API did that like the following:<br>
{% highlight java %}
enum GeeksApiErrorCodes implements ErrorCode {
    GEEK_ALREADY_EXISTS("geeks-1", HttpStatus.BAD_REQUEST);

    private final String code;
    private final HttpStatus httpStatus;

    GeeksApiErrorCodes(String code, HttpStatus httpStatus) {
        this.code = code;
        this.httpStatus = httpStatus;
    }

    @Override
    public String code() {
        return code;
    }

    @Override
    public HttpStatus httpStatus() {
        return httpStatus;
    }
}
{% endhighlight %}
Then we should somehow convert different exceptions to appropriate `ErrorCode`s. `ExceptionToErrorCode` strategy interface would do that for us:<br>
{% highlight java %}
/**
 * Strategy interface responsible for converting an instance of
 * {@linkplain Exception} to appropriate instance of {@linkplain ErrorCode}.
 * Implementations of this interface should be accessed indirectly through
 * the {@linkplain ErrorCodes}'s factory method.
 *
 * @implSpec The {@linkplain #toErrorCode(Exception)} method should be called iff the
 * call to the {@linkplain #canHandle(Exception)} returns true for the same exception.
 *
 * @see ErrorCode
 * @see ErrorCodes
 *
 * @author Ali Dehghani
 */
public interface ExceptionToErrorCode {
    /**
     * Determines whether this implementation can handle the given exception or not.
     * Calling the {@linkplain #toErrorCode(Exception)} when this method returns
     * false, is strongly discouraged.
     *
     * @param exception The exception to examine
     * @return true if the implementation can handle the exception, false otherwise.
     */
    boolean canHandle(Exception exception);

    /**
     * Performs the actual mechanics of converting the given {@code exception}
     * to an instance of {@linkplain ErrorCode}.
     *
     * @implSpec Call this method iff the {@linkplain #canHandle(Exception)} returns
     * true.
     *
     * @param exception The exception to convert
     * @return An instance of {@linkplain ErrorCode} corresponding to the given
     * {@code exception}
     */
    ErrorCode toErrorCode(Exception exception);
}
{% endhighlight %}
Again each resource should provide an implementation for this interface per each defined error codes. Geeks API does that like the following:<br>
{% highlight java %}
class GeeksApiExceptionMappers {
    @Component
    static class GeeksAlreadyExceptionToErrorCode implements ExceptionToErrorCode {
        @Override
        public boolean canHandle(Exception exception) {
            return exception instanceof GeekAlreadyExists;
        }

        @Override
        public ErrorCode toErrorCode(Exception exception) {
            return GeeksApiErrorCodes.GEEK_ALREADY_EXISTS;
        }
    }
}
{% endhighlight %}
This basically catch all `GeekAlreadyExists` exceptions and convert them to `GeeksApiErrorCodes.GEEK_ALREADY_EXISTS` error code.<br>
Since there would be quite a large number of implementations like this, It's better to delegate finding the right implementation for each exception to another abstraction. `ErrorCodes` factory would do that like the following:<br>
{% highlight java %}
/**
 * Acts as a factory for {@linkplain ExceptionToErrorCode} implementations.
 * By calling the {@linkplain #of(Exception)} factory method, clients can
 * find the corresponding {@linkplain ErrorCode} for the given
 * {@linkplain Exception}. Behind the scenes, this factory method, first finds
 * all available implementations of the {@linkplain ExceptionToErrorCode}
 * strategy interface. Then selects the first implementation that can actually
 * handles the given exception and delegate the exception translation process
 * to that implementation.
 *
 * @implNote If the factory method couldn't find any implementation for the given
 * {@linkplain Exception}, It would return the
 * {@linkplain me.alidg.rest.errors.ErrorCode.UnknownErrorCode} which represent
 * an Unknown Error.
 *
 * @author Ali Dehghani
 */
@Component
class ErrorCodes {
    private final ApplicationContext context;

    ErrorCodes(ApplicationContext context) {
        this.context = context;
    }

    /**
     * Factory method to find the right {@linkplain ExceptionToErrorCode}
     * implementation and delegates the conversion task to that implementation.
     * If it couldn't find any registered implementation, It would return an
     * Unknown Error represented by the {@linkplain UnknownError} implementation.
     *
     * @implNote Currently this method queries Spring's
     * {@linkplain ApplicationContext} to find the {@linkplain ExceptionToErrorCode}
     * implementations. So in order to register your {@linkplain ExceptionToErrorCode}
     * implementation, you should annotate your implementation with one of
     * Spring Stereotype annotations, e.g. {@linkplain Component}.
     * Our recommendation is to use the {@linkplain Component} annotation or
     * another meta annotation based on this annotation.
     *
     * @param exception The exception to find the implementation based on that
     * @return An instance of {@linkplain ErrorCode} corresponding the given
     * {@code exception}
     */
    ErrorCode of(Exception exception) {
        return implementations()
                .filter(impl -> impl.canHandle(exception))
                .findFirst()
                .map(impl -> impl.toErrorCode(exception))
                .orElse(ErrorCode.UnknownErrorCode.INSTANCE);
    }

    /**
     * Query the {@linkplain #context} to find all available implementations of
     * {@linkplain ExceptionToErrorCode}.
     */
    private Stream<ExceptionToErrorCode> implementations() {
        return context.getBeansOfType(ExceptionToErrorCode.class).values().stream();
    }
}
{% endhighlight %}
Here comes the `ErrorResponse` class to used to as the JSON representation of each error:<br>
{% highlight java %}
/**
 * An immutable data structure representing HTTP error response bodies. JSON
 * representation of this class would be something like the following:
 * <pre>
 *     {
 *         "status_code": 404,
 *         "reason_phrase": "Not Found",
 *         "errors": [
 *             {"code": "res-15", "message": "some error message"},
 *             {"code": "res-16", "message": "yet another message"}
 *         ]
 *     }
 * </pre>
 *
 * @author Ali Dehghani
 */
@JsonAutoDetect(fieldVisibility = ANY)
class ErrorResponse {
    /**
     * The 4xx or 5xx status code for error cases, e.g. 404
     */
    private final int statusCode;

    /**
     * The HTTP reason phrase corresponding the {@linkplain #statusCode},
     * e.g. Not Found
     *
     * @see <a href="https://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html">
     * Status Code and Reason Phrase</a>
     */
    private final String reasonPhrase;

    /**
     * List of application-level error code and message combinations.
     * Using these errors we provide more information about the
     * actual error
     */
    private final List<ApiError> errors;

    private ErrorResponse(int statusCode, String reason, List<ApiError> errors) {
        // Some preconditoon checks

        this.statusCode = statusCode;
        this.reasonPhrase = reason;
        this.errors = errors;
    }

    /**
     * Static factory method to create a {@linkplain ErrorResponse} with multiple
     * {@linkplain ApiError}s. The canonical use case of this factory method is when
     * we're handling validation exceptions, since we may have multiple validation
     * errors.
     */
    static ErrorResponse ofErrors(HttpStatus status, List<ApiError> errors) {
        return new ErrorResponse(status.value(), status.getReasonPhrase(), errors);
    }

    /**
     * Static factory method to create a {@linkplain ErrorResponse} with a single
     * {@linkplain ApiError}. The canonical use case for this method is when we trying
     * to create {@linkplain ErrorResponse}es for regular non-validation exceptions.
     */
    static ErrorResponse of(HttpStatus status, ApiError error) {
        return ofErrors(status, Collections.singletonList(error));
    }

    /**
     * An immutable data structure representing each application-level error. JSON
     * representation of this class would be something like the following:
     * <pre>
     *     {"code": "res-12", "message": "some error"}
     * </pre>
     *
     * @author Ali Dehghani
     */
    @JsonAutoDetect(fieldVisibility = ANY)
    static class ApiError {
        /**
         * The error code
         */
        private final String errorCode;

        /**
         * Possibly localized error message
         */
        private final String message;

        ApiError(String errorCode, String message) {
            this.errorCode = errorCode;
            this.message = message;
        }
    }
}
{% endhighlight %}
And finally to glue all of these together, a Spring MVC `ExceptionHandler` would catch all exceptions and render appropriate `ErrorResponse`es:<br>
{% highlight java %}
/**
 * Exception handler that catches all exceptions thrown by the REST layer
 * and convert them to the appropriate {@linkplain ErrorResponse}s with a
 * suitable HTTP status code.
 *
 * @see ErrorCode
 * @see ErrorCodes
 * @see ErrorResponse
 *
 * @author Ali Dehghani
 */
@ControllerAdvice
class ApiExceptionHandler {
    private static final String NO_MESSAGE_AVAILABLE = "No message available";

    /**
     * Factory to convert the given {@linkplain Exception} to an instance of
     * {@linkplain ErrorCode}
     */
    private final ErrorCodes errorCodes;

    /**
     * Responsible for finding the appropriate error message(s) based on the given
     * {@linkplain ErrorCode} and {@linkplain Locale}
     */
    private final MessageSource apiErrorMessageSource;

    /**
     * Construct a valid instance of the exception handler
     *
     * @throws NullPointerException If either of required parameters were {@code null}
     */
    ApiExceptionHandler(ErrorCodes errorCodes, MessageSource apiErrorMessageSource) {
        Objects.requireNonNull(errorCodes);
        Objects.requireNonNull(apiErrorMessageSource);

        this.errorCodes = errorCodes;
        this.apiErrorMessageSource = apiErrorMessageSource;
    }

    /**
     * Catches all non-validation exceptions and tries to convert them to
     * appropriate HTTP Error responses
     *
     * <p>First using the {@linkplain #errorCodes} will find the corresponding
     * {@linkplain ErrorCode} for the given {@code exception}. Then based on
     * the resolved {@linkplain Locale}, a suitable instance of
     * {@linkplain ErrorResponse} with appropriate and localized message will
     * return to the client. {@linkplain ErrorCode} itself determines the HTTP
     * status of the response.
     *
     * @param exception The exception to convert
     * @param locale The locale that usually resolved by {@code Accept-Language}
     * header. This locale will determine the language of the returned error
     * message.
     * @return An appropriate HTTP Error Response with suitable status code
     * and error messages
     */
    @ExceptionHandler(ServiceException.class)
    ResponseEntity<ErrorResponse> handleExceptions(ServiceException exception,
                                                   Locale locale) {
        ErrorCode errorCode = errorCodes.of(exception);
        ErrorResponse errorResponse = ErrorResponse.of(errorCode.httpStatus(),
                                                       toApiError(errorCode, locale));

        return ResponseEntity.status(errorCode.httpStatus()).body(errorResponse);
    }

    /**
     * Convert the passed {@code errorCode} to an instance of
     * {@linkplain ErrorResponse} using the given {@code locale}
     */
    private ErrorResponse.ApiError toApiError(ErrorCode errorCode, Locale locale) {
        String message;
        try {
            message = apiErrorMessageSource.getMessage(errorCode.code(),
                                                       new Object[]{},
                                                       locale);
        } catch (NoSuchMessageException e) {
            message = NO_MESSAGE_AVAILABLE;
        }

        return new ErrorResponse.ApiError(errorCode.code(), message);
    }
}
{% endhighlight %}

[1]: https://www.amazon.com/Enterprise-Integration-Patterns-Designing-Deploying/dp/0321200683
[2]: https://zalando.github.io/restful-api-guidelines/TOC.html
[status-codes-wiki]: https://en.wikipedia.org/wiki/List_of_HTTP_status_codes
[google-translate-en-fr]: https://translate.google.com/#en/fr/The%20geek%20already%20exists
[github-source]: https://github.com/alimate/Resource-Based-Error-Codes
