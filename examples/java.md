# Java examples

## Good

```java
public void handleError(final ServiceBusErrorContext context) {
  log.error("Service Bus error in entity '{}' (source: {}): {}",
      context.getEntityPath(),
      context.getErrorSource(),
      context.getException().getMessage(),
      context.getException());
}
```
Correct formatted log with context.

```java
log.error("Unexpected exception", e.getMessage());
```
Does not reveal stack trace.

```java
log.info("User created")
````
Correct log level, is info.

## Counterexamples

```java
public void handleError(final ServiceBusErrorContext context) {
  final String errorMessage = "Service Bus error in entity '%s': %s%n" +
      context.getEntityPath() + context.getException().getMessage();
  log.error(errorMessage);
}
```
Incorrect formatting, should have used .format().

```java
log.error("Unexpected exception", e);
```
Reveals stacktrace.

```java
log.error("User created")
```
Wrong log level, is info but not error.