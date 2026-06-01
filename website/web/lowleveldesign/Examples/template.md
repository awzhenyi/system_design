---
sidebar_position: -1
---

# Low Level Design Interview Template

This template outlines the structured approach for tackling low-level design (LLD) interview questions. Follow these steps to systematically design object-oriented solutions.

## What is Low-Level Design?

Low-level design (LLD) sits between coding interviews (algorithms) and system design (distributed systems). Unlike system design which defines the architecture of a distributed system, LLD defines the **internal structure of a single component** within that system.

**Example:**
- **System Design**: Sketch out services, databases, and message queues for Uber's backend
- **LLD**: Design the `Trip` class, `PricingCalculator` interface, and how these objects collaborate within a single service

### What Interviewers Evaluate

Interviewers are evaluating four key aspects:

1. **How you break down problems** into clear entities with well-defined responsibilities
2. **How your classes interact** without becoming a tangled mess
3. **Whether your design can handle change** without requiring a complete rewrite
4. **How clearly you communicate** your reasoning and trade-offs

> A good design that you can't explain is indistinguishable from a bad design.

## Two Approaches: Comprehensive vs. Focused Framework

This template provides a comprehensive 10-step approach. For a more focused, time-efficient framework optimized for interviews, see the [Five-Phase Delivery Framework](#five-phase-delivery-framework) section below.

## Step 1: Understand Requirements and Clarify Scope

### Ask Clarifying Questions
- **Functional Requirements**: What are the core features that need to be implemented?
  - What are the main use cases?
  - What operations should the system support?
  - Are there any specific workflows or business rules?
  
- **Non-Functional Requirements**:
  - **Scale**: How many users? Expected load/throughput?
  - **Performance**: What are the latency requirements?
  - **Consistency**: Strong consistency or eventual consistency acceptable? 
  - **Availability**: What is the acceptable downtime?
  - **Durability**: How critical is data persistence?
  
- **Constraints and Assumptions**:
  - What are the technical constraints?
  - What can we assume about the environment?
  - Are there any integration requirements?

### Define Scope
- List the features you'll implement (prioritize if needed)
- Identify what's out of scope
- Get confirmation from the interviewer on the scope

## Step 2: Identify Core Entities and Their Relationships

### Identify Main Entities
- List the primary objects/concepts in the system
- Think about nouns in the problem statement
- Examples: User, Order, Payment, Product, Cart, etc.

**Filtering Entities:**
Not every noun becomes a class. Apply this filter:
- **Does it maintain changing state or enforce rules?** → It's probably an entity (class)
- **Is it just information attached to something else?** → It's probably just a field or an input parameter

**Example (Parking Lot):**
- `Vehicle`: External to your system. You don't manage it, you don't care about its state. You just need to know its type (motorcycle, car, large). That's an **enum**, not a class.
- `Ticket`: Internal state your system creates and manages. That's a **class**.

**Find the Orchestrator:**
- The entity that coordinates the workflow
- In a parking lot: `ParkingLot`
- In a chess game: `Game`
- This is the entry point for external requests

### Define Entity Attributes
- For each entity, identify key attributes
- Consider what data each entity needs to store
- Think about relationships between entities

### Identify Relationships
- **One-to-One**: One entity relates to exactly one other entity
- **One-to-Many**: One entity relates to multiple instances of another
- **Many-to-Many**: Multiple entities relate to multiple instances
- Sketch quick relationships—which entity owns which, who creates whom
- **Skip detailed UML**: Simple boxes, arrows, and labels communicate structure just fine. Modern engineers design in code, not diagrams.

## Step 3: Design Classes and Interfaces

### Define Class Structure
- **Core Classes**: Main business logic classes
- **Service Classes**: Classes that orchestrate operations
- **Repository/DAO Classes**: Classes for data access
- **Utility/Helper Classes**: Supporting classes

**Important**: Tie everything back to your requirements list. If you can't point to a requirement that needs a field or method, you probably don't need it. Candidates frequently add methods and fields that aren't needed for the requirements they agreed on. This burns precious time and increases design complexity.

### Design Interfaces
- Define contracts/interfaces before implementations
- Focus on what operations are needed, not how they're implemented
- Consider abstraction and polymorphism

### OOP Concepts as Tools

**Encapsulation**: Keep fields private. Expose behavior through methods. If you're returning references to mutable internal collections, you've broken encapsulation.

**Abstraction**: Define interfaces for variations. Multiple payment methods? Different vehicle types? Create an interface. The caller doesn't need to know which implementation they're using.

**Polymorphism**: Lets objects handle themselves. No type checking, no switch statements on types. Each object knows how to perform its own operations.

**Inheritance**: ⚠️ **Where candidates get into trouble**. Modern design favors **composition over inheritance**. Default to interfaces. Only use inheritance when you genuinely need to share stable implementation across classes. In most LLD interviews, you don't need inheritance at all.

### Identify Methods for Each Class
- **Public Methods**: API that other classes will use
- **Private Methods**: Internal helper methods
- **Getter/Setter Methods**: For accessing/modifying attributes
- Consider method signatures: parameters, return types, exceptions

### Apply Design Principles

**Principles That Actually Matter:**

- **KISS (Keep It Simple)**: The most violated principle in LLD interviews. Start simple and add complexity only when simplicity stops working. Over-engineering is a red flag.

- **Single Responsibility**: Each class does one thing. If you can't describe what a class does in one sentence without using "and," it's doing too much.
  - ❌ Bad: A `Report` class that generates content, formats PDFs, and saves files
  - ✅ Good: Separate `ReportGenerator`, `PDFFormatter`, and `FileSaver` classes

- **Separation of Concerns**: Keep layers apart. UI code shouldn't contain business logic. Business logic shouldn't know how data is stored. This is about architectural boundaries—presentation, domain, persistence.

**Other SOLID Principles:**
- **O**pen/Closed: Open for extension, closed for modification
- **L**iskov Substitution: Subtypes must be substitutable for their base types (e.g., a `Penguin` class that throws an exception when you call `fly()` violates this)
- **I**nterface Segregation: Clients shouldn't depend on interfaces they don't use
- **D**ependency Inversion: Depend on abstractions, not concretions

> **Key Insight**: The rest of SOLID matters, but they're more about recognizing when your design is getting awkward than actively applying rules. Develop a "smell" for good code through practice.

## Step 4: Apply Design Patterns

### Patterns That Actually Matter

Don't memorize all 23 Gang of Four patterns. These four cover 90% of what you'll encounter:

1. **Strategy Pattern**: Used when logic might change. When you see `if (type == "credit")` or `switch (vehicleType)`, that's a Strategy pattern waiting to happen. This is probably the most common pattern in LLD interviews.

2. **Observer Pattern**: Lets objects subscribe to events—especially useful when you don't know who might care up-front. Examples: stock price update triggering multiple displays, order placement notifying inventory/analytics/notifications.

3. **State Machine**: Handles objects whose behavior depends on their current state. Vending machines, document workflows, game states. If the word "state" appears multiple times in requirements, think state machine.

4. **Factory Pattern**: Centralizes object creation. Instead of `new EmailNotification()` scattered throughout code, call `notificationFactory.create(type)`. When you add SMS, you update the factory. Everything else stays the same.

**Other Patterns (Situational):**
- Builder, Singleton, Decorator, Facade: Use only when they genuinely improve the design
- Most interviews: Using one or two patterns (or none) is perfectly fine

> **Key Insight**: Patterns should emerge from your design decisions, not drive them. If you're forcing one in, step back. In most interviews, using one or two patterns (or none) is perfectly fine.

### Justify Pattern Usage
- Explain why a particular pattern fits the problem
- Show how it improves the design
- Avoid over-engineering; use patterns only when they add value

## Step 5: Handle Edge Cases and Error Scenarios

### Identify Edge Cases
- **Null/Empty Inputs**: What happens with null or empty data?
- **Boundary Conditions**: Minimum/maximum values, empty collections
- **Concurrent Access**: Multiple threads accessing shared resources
- **Invalid States**: Operations on objects in invalid states

### Design Error Handling
- Define custom exceptions if needed
- Decide on error propagation strategy
- Consider validation at entry points
- Design for graceful degradation

### Handle Concurrency
- Identify shared resources
- Consider thread-safety requirements
- Use appropriate synchronization mechanisms
- Consider lock-free alternatives where possible

## Step 6: Design Data Models and Storage

### Design Data Structures
- Choose appropriate data structures for in-memory operations
- Consider time and space complexity
- Think about data access patterns

### Design Database Schema (if applicable)
- Tables and their relationships
- Primary keys, foreign keys, indexes
- Normalization vs. denormalization trade-offs

### Consider Caching Strategy
- What data should be cached?
- Cache invalidation strategy
- Cache eviction policy (LRU, LFU, etc.)

## Step 7: Design API/Public Interface

### Define Public APIs
- Method signatures
- Input/output contracts
- Error responses
- Consider RESTful design if applicable

### Document API Contracts
- Parameter types and constraints
- Return types
- Exceptions that can be thrown
- Preconditions and postconditions

## Step 8: Discuss Trade-offs and Optimizations

### Identify Trade-offs
- **Time vs. Space**: Faster algorithms might use more memory
- **Consistency vs. Availability**: CAP theorem considerations
- **Simplicity vs. Flexibility**: Simple design vs. extensible design
- **Read vs. Write Optimization**: Optimize for common operations

### Discuss Optimizations
- Where can we optimize?
- What are the bottlenecks?
- Consider indexing, caching, lazy loading
- Discuss when optimizations are premature

## Step 9: Code Structure and Organization

### Organize Code
- Package/module structure
- Separation of concerns
- Layered architecture (if applicable):
  - Presentation Layer
  - Business Logic Layer
  - Data Access Layer

### Consider Code Quality
- Naming conventions
- Code reusability
- Maintainability
- Testability

## Step 10: Testing Strategy

### Identify Test Cases
- **Unit Tests**: Test individual methods/classes
- **Integration Tests**: Test interactions between components
- **Edge Cases**: Test boundary conditions and error scenarios
- **Concurrency Tests**: Test thread-safety

### Design for Testability
- Dependency injection
- Mocking strategies
- Test data setup

## Common LLD Interview Questions

### Problems to Practice, In Order

These are ordered to build skills progressively:

1. **Design Connect Four** - Game problems test state machines and win condition checking
2. **Design Amazon Locker** - Similar to parking lot but with different constraints, good for seeing how designs differ based on requirements
3. **Design an Elevator System** - Introduces scheduling algorithms and more complex state management
4. **Design a Parking Lot** - The classic starter problem. Tests entity extraction, state management, and basic orchestration
5. **Design a Rate Limiter** - Different flavor of problem, algorithm-focused but still tests class design
6. **Design an Inventory Management System** - More complex business logic, multiple entity types

### Other Typical Problem Domains
- **ATM System**: Design an ATM machine system
- **Vending Machine**: Design a vending machine
- **Library Management**: Design a library management system
- **Online Shopping Cart**: Design a shopping cart system
- **Tic-Tac-Toe**: Design a game system
- **Snakes and Ladders**: Design a board game system

**Practice Strategy:**
For each problem, follow the delivery framework. Set a timer if it helps. The goal is to build muscle memory for the five phases so that in the actual interview, the structure is automatic and you can focus on the problem itself.

After you attempt a problem, compare your solution to breakdowns. Don't just check if you got the "right answer" (there usually isn't one). Ask yourself:
- Did you identify the same entities or over/under-model?
- Is your orchestrator handling the right responsibilities?
- Where did you put state and does that make sense?
- Are your class APIs clean or do callers need to know too much?

The goal isn't to match some canonical solution. It's to develop design intuition you can apply to any problem.

### Key Focus Areas
- Object-oriented design principles
- Design patterns application
- Class relationships and hierarchies
- Method design and signatures
- Error handling and edge cases
- Concurrency and thread-safety
- Data modeling

## Five-Phase Delivery Framework

This is a more focused, time-efficient framework optimized for interviews. Each phase builds on the previous one, so by the time you're writing code, the design decisions have already been made.

### Phase 1: Requirements (~5 minutes)

Every LLD interview starts with a vague prompt: "Design Tic Tac Toe" or "Design a parking lot system." Your job is to turn that into something concrete before you touch design.

**Ask questions around four themes:**
1. **Operations**: What operations must this system support?
2. **State Transitions**: What defines success, failure, or state transitions?
3. **Error Handling**: How should invalid inputs be handled?
4. **Scope**: What's explicitly in and out of scope?

Write down the answers and formulate them into a clear set of requirements you can design against.

### Phase 2: Entities and Relationships (~3 minutes)

Scan your requirements for the meaningful nouns. These are the *things* that need to exist in your system. Apply the filter mentioned in Step 2 to determine what becomes a class vs. a field.

Find the orchestrator—the entity that coordinates the workflow. Sketch quick relationships. Don't overthink notation.

### Phase 3: Class Design (~10-15 minutes)

Go entity by entity, starting from the entry point of your system. For each one:
- Figure out what state it needs to remember to enforce the requirements
- Determine what operations it needs to support
- Tie everything back to your requirements list

By the end of this phase, you have pseudocode class outlines showing fields and method signatures for each entity.

### Phase 4: Implementation (~10 minutes)

Pick the most interesting methods and implement them. Start with the happy path, then handle edge cases. Most interviewers want pseudocode, not syntactically perfect code.

Your interviewer will usually tell you which methods to implement. If not, pick the ones that show how your classes cooperate and how state transitions occur.

### Phase 5: Verification (~2-5 minutes)

Trace through a concrete scenario. "A car enters, gets ticket T1, parks in spot B. Later exits with T1, pays the fee." Does your state actually change correctly? This catches bugs before your interviewer finds them.

If there's time, the interviewer might ask extension questions: "How would you add undo?" or "What if we need multiple floors?" These test whether your design can evolve without a rewrite.

## Best Practices

### Communication

**Communication is half the battle.** Talk through your reasoning. Interviewers can't read your mind, and a silent candidate makes for an anxious interviewer.

- **Think Aloud**: Explain your thought process continuously
- **Explain Why**: Say things like "I'm making this a separate class because it has its own state to manage" or "I considered putting this logic here, but it makes more sense in the orchestrator because..."
- **Ask Questions**: Don't assume; clarify requirements
- **If Stuck**: Say so. "I'm thinking about where this responsibility should live" is better than awkward silence
- **Take Hints**: If the interviewer offers a hint, take it gracefully—that's not a negative signal

### Time Management

**Follow the Framework:**
The delivery framework is your pacing guide. If you're 15 minutes in and still on requirements, you're in trouble. If you're 20 minutes in without any class design on the board, you need to move faster.

**Common Failure Modes:**
- Spending too long on early phases and running out of time before showing meaningful implementation
- Diving straight into code without establishing structure (you'll likely need to backtrack)

**Recommended Timeline (Five-Phase Framework):**
- **Requirements**: ~5 minutes
- **Entities and Relationships**: ~3 minutes
- **Class Design**: ~10-15 minutes
- **Implementation**: ~10 minutes
- **Verification**: ~2-5 minutes

### Common Mistakes to Avoid

1. **Over-Engineering** ⚠️ **Most Common Mistake**
   - You're not building production software. You're demonstrating design thinking.
   - If a simple solution works, use it.
   - Don't add Strategy patterns for one implementation.
   - Don't build elaborate factory hierarchies for creating two object types.
   - Don't introduce interfaces where concrete classes would suffice.
   - When the interviewer asks "how would you extend this?", *that's* when you explain how the design would evolve. But don't build for extensions upfront.

2. **Jumping to code too quickly** - Establish structure first
3. **Not asking clarifying questions** - Turn vague prompts into concrete requirements
4. **Spending too long on early phases** - Use the framework to pace yourself
5. **Ignoring edge cases** - But handle them after the happy path
6. **Not considering concurrency** - When relevant to the problem
7. **Poor naming conventions** - Clear names communicate intent
8. **Tight coupling between classes** - Design for loose coupling
9. **Not applying design principles** - But don't force them unnecessarily
10. **Adding unnecessary complexity** - KISS principle is most violated

## Example Flow

1. **Listen carefully** to the problem statement
2. **Ask clarifying questions** to understand requirements
3. **Identify entities** and their relationships
4. **Design classes** with methods and attributes
5. **Apply design patterns** where appropriate
6. **Handle edge cases** and error scenarios
7. **Discuss trade-offs** and optimizations
8. **Refine the design** based on feedback

## Summary

A successful low-level design interview demonstrates:
- Strong object-oriented design skills
- Understanding of design principles and patterns (especially KISS and Single Responsibility)
- Ability to handle edge cases and concurrency
- Clear communication and problem-solving approach
- Practical coding and design experience

### Key Takeaways

**LLD interviews aren't about memorizing patterns or principles.** They're about demonstrating that you can take an ambiguous problem and turn it into clean, maintainable code structure. That's a skill you use every day as a software engineer.

**The good news:** The learning curve is shorter than algorithm prep. After working through 5-6 problems actively, you'll notice the pattern recognition becoming automatic. You'll hear "design a parking lot" and immediately know to look for the orchestrator, identify what's internal state versus external input, and structure your class APIs around clear responsibilities.

**The framework gives you a reliable path through any problem.** The principles keep your design clean. And the practice builds the intuition to apply both under pressure.

Remember: The interviewer is evaluating your thought process, design skills, and ability to write clean, maintainable code—not just the final solution. A good design that you can't explain is indistinguishable from a bad design.

## Additional Resources

This template incorporates insights from [Hello Interview's guide on preparing for LLD interviews](https://www.hellointerview.com/blog/how-to-prepare-lld), which emphasizes:
- Focusing on fundamentals that actually matter (KISS, Single Responsibility, Separation of Concerns)
- Learning the four patterns that cover 90% of interview scenarios
- Using a structured delivery framework to avoid getting lost
- Practicing with real problems to build design intuition

The key is developing intuition through practice, not memorizing definitions. After working through several problems, you'll develop a "smell" for good code that guides your design decisions naturally.

