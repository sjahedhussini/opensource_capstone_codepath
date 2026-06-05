# opensource Capstone Codepath
# Contribution [1]: [[Java] support copy non-serializable objects such as java.lang.Package]

**Contribution Number:** [1]  
**Student:** [Sayed Jahed Hussini]  
**Issue:** [[GitHub issue link](https://github.com/apache/fory/issues/2941)]  
**Status:** [Phase I] [In Progress]

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
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

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
