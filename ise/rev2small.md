**components are**
**MPRR**
modular, portable, replaceable, reusable

**mistakes of component diagrams**
**DDBPLS**
dont share too much details in components, more abstract than objects
no dangling lollipops and sockets
components need both sockets and lollipops
use ports for interactions 
dont share lines in notation
lollipops can be shared by different components

**definition of software engineering design**
software eng design is the analyzing, evaluating, validating, and specifying the architectural, detailed, and construction models to build software in accordance to their functional and non functional requirements

**importance of software design**
**RQFRME**
responsibilities are mapped to models of software
ensures quality of softwre
forms the foundation of software
reusable solutions
improves project management
increase efficiency in resource allocation

**software design activities**
**ADCIDM**
architecture design
detailed design
construction design
human computer interfaces
documentation
management

**software design fundamentals**
**MAECCSSC**
modularization
abstraction
encapsulation
cohesion and coupling
separation of interface and implementation
sufficiency - how well software can provide the service
completeness - how well designed the software to provide

**C4 model**
Context, the whole system at the highest level eg. customer context interact with portal context
Container, containers inside a context, eg. web app, mobile app, database
Component, components in a container
Code, class diagram

**4+1 view**
**LPIPU**
logical view: logic towards end users
process view: aspects of system
implementation view: system from programmer view, software management
physical view: system engineers view, topology, physical connections
use case view: sequences of interactions between objects and processes

**Gang of four**
**Creational**
- Factory: to create and store many instances of abstract class
- abstract factory: create many objects that relate/belong to each other
- builder: create complex object
- prototype: create object based on existing object
- singleton: only one instance of class

**structural**
- adapter: make two incompatible interfaces work together
- bridge: separate what something does from how it is implemented, tv remote should work with different tv brands.
- composite: reuse existing class in new context
- decorator: add optional features without changing the class
- facade: offer entry point to complicated process eg. place order button handles complicated process
- flyweight: share data instead of storing duplicates eg. many trees in game
- proxy: eg. photo gallery shows thumbnails first, loads full when opened

**behavioural**
- chain of responsibility: pass request until one handler handles it
- command: turn action into object that can be stored or reversed
- interpreter: define rules for understanding a language
- iterator: visit items without seeing how it is stored
- mediator: central coordinator for direct connections
- memento: save object state to be restored later
- observer: events to notify list of observers
- state: change object behavior when state change
- strategy: swap between different ways of doing same task
- template method: keep common workflow fixed, but let subclass customize
- visitor: new operation across objects without changing types

**non execution based testing techniques**
- to test for quality without running code
- **walkthrough**: designer walks through software product, participants ask questions about possible errors to improve product. prep > analysis
- **inspection**: identify product **anomalies**. metrics: inspection rate, fault density, fault detection rate.
	- overview > prep > inspection > rework > follow up

**execution based testing**
- no false positives due to running code nature, automation possible
- tests, **CURRP**
	- correctness
	- utility
	- reliability
	- robustness
	- performance

**black box testing technique**
1. equivalence partioning
	- input values split between valid and invalid partitions
	- come up with as few partitions as possible
	- eg only 50 - 60 is valid
	- so we will have 3 partitions, <=49, 50 to 60, 61<=.
	- random values are chosen in these 3 to be the test values
2. boundary value analysis
	- similar to equivalnce, but test values are at the boundaries
	- choose values that are rejected as well, so if 50 to 90 is accepted, choose 49, 50, 51, 89, 90, 91
	- but choose only 1 value on each side, so choose 49, and either 50 or 51
	- 3 value BVA needs all 3, so 49, 50, 51. 2 value BVA takes 49 and 50
3. decision table
	- list all causes, eg. email and password both have values or true or false
	- find total combis, formula = c1_value x c2_value eg. 2 x 2 = 4
	- fill all possible combinations, and add expected outcomes
	- reduce test combinations based on combinations that are indifferent
	- disadv: not suited for system with alot of causes
	- A login checks four Y/N conditions in order: account exists, password correct, account not locked, 2FA code correct. The first failed check denies access immediately and the rest are not checked. Access is granted only if all four pass. After reduction, how many rules remain?
		- 4 Y/N
		- first rule is account exists: N, the rest dont matter
		- 2nd rule is account: Y, password: N, rest dont matter
		- 3rd rule is account:Y, password:Y, not locked: N, rest dont matter
		- 4th rule account:Y, pass:Y, notlocked:Y, 2FA: N
		- 5th rule all Y. so total 5 rules

**blackbox considers input/output, whitebox looks at the code**
**both complement each other**
- **whitebox cannot reveal errors due to missing paths**
- **black box cannot reveal due to missing implementation details**

**white box testing technique**
- coverage testing
	- whether each statement in code executes at least once
	- does not guarantee coverage of all possible paths
	- cyclomatic complexity for control flow graphs
	- $$M = E - N + 2P$$
	- E = edges, N = nodes, P = 1 
	- or: Number of decisions + 1 = 4
		- decisions include, If, elif, while, for, foreach, do, switch(1 per case, not including the switch itself and default)
```python
string Grade(int score) {
  string g;                   // +1 node, +1 edge (flows into first 'if')
  if (score >= 80)            // +1 node, +2 edges
    g = "A";                  // +1 node, +1 edge (jumps to 'return g')
  else if (score >= 65)       // +1 node, +2 edges 
    g = "B";                  // +1 node, +1 edge (jumps to 'return g')
  else if (score >= 50)       // +1 node, +2 edges (True -> "C", False -> 'else')
    g = "C";                  // +1 node, +1 edge (jumps to 'return g')
  else                        // +0 nodes, +0 edges
    g = "F";                  // +1 node, +1 edge (flows into 'return g')
  return g;                   // +1 node, +0 edges (exit has no outgoing edges)
}
// 11 - 9 + 2 = 4
// 3 decisions + 1 = 4 // same results
// while loops also count as 2 edges, for loops as well
```
```python
double Fee(char type) {
  double fee; // 1 edge, 1 node
  switch (type) { // 4 edge(1 for each case/default) , 1 node
    case 'S': fee = 10; break; // 1 edge towards return fee , 1 node
    case 'F': fee = 5;  break; // 1 edge towards return fee , 1 node
    case 'A': fee = 0;  break; // 1 edge towards return fee , 1 node
    default:  fee = 20; // 1 edge towards return fee , 1 node
  }
  return fee; // 1 node
}
```
- branch coverage
	- attempts to explore all possible branches (true false of all decision)
	- $$\text{Branch Coverage (\%)} = \frac{\text{Number of branches executed and tested}}{\text{Total number of branches}} \times 100\%$$
```python
void CheckSavings(int income, int expenses) {
  int result = income - expenses;
  if (result > 500)
    Console.WriteLine("Not bad");
  else if (result < 500)
    Console.WriteLine("Reduce those expenses");
}
// two if statements, both true or false, = 4 branches
// test 1: income 1000, expense 200, takes first if to be true and ends. so 1 branch covered
// test 2: income 400, expense 400, takes first if to be false and 2nd if to be true, so 2 more branches covered. 2nd if's false is not covered.
// therefore 75% coverage
```
- path coverage: covers all paths, expensive, impractical
- control flow testing
	- test all possible paths in control flow graph
	- guarantees every statement and transition is executed once 
	- basis paths: results in 100% statement cov and branch cov
		- size of paths is cyclomatic complexity (M is upper bound)
		1. pick baseline path (have as many decisions)
		2. back-track to nearest decision and flip decision
		3. repeat until all decisions covered

version management terminologies
version: configuration item at one point in development
- revision: config item ordered in time
- variant: functionally equivalent versions, but for different settings

**maintenance type CAPP**
corrective: fixing errors
adaptive: adapting to new stuff
perfective: optimize
preventive: prevent before occur

**refactoring: internal change to make it easier to understand without changing behaviour**

bad smell code patterns 
- cloning
	- **duplicate code**
		- **refactor tech: extract method** - new method that encaps dupe code
- complex structs
	- long methods
	- **large class**
		- **refactor: extract class** - create new class
- variables and params
	- long parameter list
	- **feature envy: method placed in wrong class, a class using another class's methods more than itself**
		- **refactor: move method** - move class with feature envy to class containing the method
	- data clumps
	- primitive obsession
	- temporary field
- control structs and polymorphism
	- **switch statements: cases contain logic for different types of instances**
		- **refactor: extract subclass and extract method** - create new subclass
	- refused bequest
	- alternative class with different interface

### Limits of Refactoring

- Changes can affect:
    - **Non-functional properties** (e.g., performance)
    - The **Application Programming Interface (API)** of a system
- Refactorings that alter APIs happen when:
    - A class member is moved / renamed
    - List of parameters to an operation is changed