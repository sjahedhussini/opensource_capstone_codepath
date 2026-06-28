# opensource Capstone Codepath
# Contribution [1]: [[Java] support copy non-serializable objects such as java.lang.Package]

**Contribution Number:** [1]  
**Student:** [Sayed Jahed Hussini]  
**Issue:** [[GitHub issue link](https://github.com/apache/fory/issues/2941)]  
**Status:** [Phase I] [Complete]

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of two areas I'm actively working to deepen: Java internals and how serialization frameworks handle edge cases in complex object graphs. My background in machine learning research means I regularly work with frameworks like Hibernate and encounter objects that carry non-serializable references as part of their internal structure. I came across project Fory because of Apache Spark and working with big data. Hence, it's a familiar project that I have exposure to.   
Additionally, the inconsistency is concrete and well-defined, which is what I want for a first contribution.


---

## Understanding the Issue

### Problem Description

The bug arises when the user calls ```fory.copy(object)``` on an object whose class graph contains a non-serlizable object, it throws an ```UnsupportedExceptionType```  and wraps it in a ```CopyException```. The bug was produced when used on ```java.lang.Package```. The root cause might be Fory's copy path internally routes through the same serializer-lookup logic as its serialize/deserialize path, which hard-fails on classes marked as not supporting serialization.

### Expected Behavior
```fory.copy(object)``` should successfully produce a deep copy of the object, even when some fields reference non-serializable types like ```java.lang.Package```. For fields whose types genuinely cannot be meaningfully copied, Fory should either copy the reference as-is or skip the field. But it should not crash. The maintainer's comment on the issue suggests that returning the original reference for unsupported classes is a reasonable approach, as long as the user has explicitly registered a serializer for those types, making the behavior transparent.

### Current Behavior
Calling ```fory.copy()``` on an object that transitively contains a ```java.lang.Package``` field throws a ```RuntimeException``` wrapping an ```UnsupportedOperationException: Class class java.lang.Package``` doesn't support serialization.

### Affected Components

Based on the stack trace, the relevant parts of the codebase are:

- ```org.apache.fory.Fory``` — the top-level ```copy()``` and ```copyObject()``` methods that initiate the copy path
- ```org.apache.fory.resolver.ClassResolver``` — specifically ```getSerializerClass()``` and ```getOrUpdateClassInfo()```, where the unsupported-class check is enforced
- ```org.apache.fory.serializer.AbstractObjectSerializer``` — ```copy()``` and ```copyFields()```, which recursively walk the object graph
- ```org.apache.fory.builder.BaseObjectCodecBuilder``` — the JIT codegen layer that fails when trying to build a codec for a non-serializable type during copy code generation

---

## Reproduction Process

### Environment Setup

I am using Intellij IDE and after forking the project I used it to explore the codebase. One thing that helped me was reading the "CONTRIBUTING" and "DEVELOPMENT" guidelines before exploring the codebase. There were two issues (one with Java version and one with a specific package that was being called) that was pointed out and when I run the test for the first time to test if the codebase is working, I already knew the answer. 

### Steps to Reproduce

1. Forked the code and then downloaded it to my local machine.
2. Following the guidelines from the project, I run the code to make sure I have a clean base code. Then I tracked down the related test file. In my case, it was related to the copy feature, so I found the corresponding file. After that, I wrote a test to reproduce the error. The test I wrote was:
   ```
   @Test
    public void testCopyNonSerializablePackage() {
        Fory fory = Fory.builder()
                .withRefCopy(true)
                .requireClassRegistration(false)   // get past the registration gate
                .build();
        Package pkg = String.class.getPackage(); // java.lang
        Package copy = fory.copy(pkg);           // now expect the real bug
    }
   ```
I encountered a problem that I had to spend some time figuring out. Specifically, I was expecting the `java.lang.UnsupportedOperationException: Class class java.lang.Package doesn't support serialization` exception, but when I run it the exception was `org.apache.fory.exception.ClassUnregisteredException: Class java.lang.Package is not registered`. I found out that this was expected behavior by the original maintainers. They wanted to make sure we don't use a package that is not registered rather than run it and get an exception. Also, the original issue was raised when using it specifically in `JAVA` mode. But when I was running the test it was being run in cross language path. To resolve this issue, I run added the `.withLanguage(Language.JAVA)` line to force that it was run in Java mode. 
4. Now that I run the above test, I get the following excpetion `org.apache.fory.exception.CopyException: java.lang.UnsupportedOperationException: Class class java.lang.Package doesn't support serialization.`. It's the exact same error that was asked about in the github. 

### Reproduction Evidence

- **Commit showing reproduction:** [[Link to commit in your fork]](https://github.com/sjahedhussini/fory/tree/fix-issue-2941)
- **Screenshots/logs:** [If applicable]
- **My findings:** Explained above. 

---

## Solution Approach

### Analysis

When tracing the exception, the Java class `ClassResolver.java` is named in the stack trace. When tracing the calls, I found here's exactly why your `java.lang.Package` blows up:
```
if (config.checkJdkClassSerializable()) {
  if (cls.getName().startsWith("java") && !(Serializable.class.isAssignableFrom(cls))) {
    throw new UnsupportedOperationException(...);
  }
}
```
`java.lang.Package` satisfies all three conditions: the flag is on, its name starts with "java", and it does not implement Serializable. So it throws the error and is the root cause of the error. This is a serialization check, and it's correct for serialization. But `copy()` reuses this exact same `getSerializerClass` path, and copy has different requirements: copying happens in memory, so a class doesn't need to be serializable to be copyable. The bug is that the copy path inherits a constraint that only applies to serialization.

### Proposed Solution

- 

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** 
`Fory.copy(obj)` throws `UnsupportedOperationException: Class java.lang.Package doesn't support serialization` (wrapped in `CopyException`) when the object graph contains a non-Serializable JDK class. The same object round-trips fine through serialize/deserialize. The root cause is in `ClassResolver.getSerializerClass(Class<?>, boolean)` (~line 1503 of the `ClassResolver.java`):
```
if (config.checkJdkClassSerializable()) {
  if (cls.getName().startsWith("java") && !(Serializable.class.isAssignableFrom(cls))) {
    throw new UnsupportedOperationException(...);
  }
}
```
This is a serialization guard and is correct for `serialize` (a non-Serializable JDK class can't be turned into valid bytes), but wrong for copy, which is in-memory and has no such requirement. Because copy and serialize share the single per-class serializer created via `createSerializer`, the guard fires at serializer-creation time and blocks copy too.

**Match:** 
The codebase already has copy-specific serialization machinery I can build on:
- `ForyCopyable` / `ForyCopyableSerializer` (seen in `createSerializer`): when a class implements `ForyCopyable`, Fory wraps its serializer to give copy distinct behavior from serialize. This proves copy and serialize are meant to be able to diverge.
- `extRegistry.abstractTypeInfo` lookup in `createSerializer`: the established pattern for routing a class to a specific serializer based on type.
- `AbstractObjectSerializer.copyFields` (from the original issue trace): the default reflection/field-based copy that a class would reach if it weren't blocked by the guard.

**Plan:** [Step-by-step implementation plan]
I posted a comment on the issue thread to ask the code maintianers on how to solve the problem. There are two distinct ways to solve it and I am not sure which they prefer. I am waiting for their response before implementing the changes. 
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [4] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [[GitHub PR URL when submitted]](https://github.com/apache/fory/pull/3797)

**PR Description:** fix(java): support copy of non-serializable JDK classes such as java.lang.Package

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review]

---

## Learnings & Reflections

### Technical Skills Gained

I didn't learn a lot technically, as the problem was simple, and my main goal was to start the contribution process. I did learn about the contribution process. 

### Challenges Overcome

Nothing too difficult in this contribution. 

### What I'd Do Differently Next Time

I would pick the problem more cautiously to solve it. I picked a project that is active with a lot of activity, but I still didn't get any response to my inquiries in regards to the issue. 

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
