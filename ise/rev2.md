---
title: INF2001 Intro to Sweng - Revision 2
course: INF2001
tags:
  - INF2001
  - software-engineering
  - revision
  - quiz-prep
quiz_format: 45 mins (Closed-book) | MCQ, Short Answers, Multi-Select
date: 2026-10-22
---

# INF2001 Intro to Sweng — Revision 2

> [!info] Quiz Format
> - **Duration:** 45 minutes (Closed-book)
> - **Question Types:** MCQ, Short Answers, and Multi-Select

## Table of Contents
- [[#Component Diagrams]]
- [[#Software Engineering Design]]
- [[#Overview of Architecture]]
- [[#C4 Model]]
- [[#4+1 View of Architecture]]
- [[#Architectural Patterns]]
- [[#Design Patterns (Gang of Four)]]
- [[#Non-Execution-Based Testing]]
- [[#Execution-Based Testing]]
- [[#Blackbox Testing]]
- [[#Whitebox Testing]]
- [[#User Acceptance Testing (UAT)]]
- [[#Application Packaging]]
- [[#Software Configuration Management (SCM)]]
- [[#Version Management]]
- [[#System Building]]
- [[#Release Management]]
- [[#Continuous Integration & CI/CD]]
- [[#Software Maintenance]]
- [[#Refactoring]]

---

## Component Diagrams

- **Components** are independent units within a system that encapsulate a collection of data and functions.
- Components are **higher-level abstractions than objects** and encapsulate their internal structure.
- Components collaborate with each other through **well-defined interfaces**.
- Components can be a **collection of classes, a web service, or a resource**.
- Components are to be 
  - **modular
  - **portable**
  - **replaceable
  - **reusable**.
- Makes discussion easier at higher levels for clients.

### Interfaces, Nesting, Ports & Delegation
- **Provided Interface (Lollipop):** Component A is providing an interface `I` to others to collaborate with it; it defines how it accepts the collaboration.
- **Required Interface (Socket):** Component B requires interface `I`, denoted by a connector (socket).
- **Nested Components:** Part of component A's functionality is provided by sub-component B (e.g., component A contains sub-components B and C).
- **Ports and `<<delegate>>`:**
  - Component A can delegate responsibility for interface `I` to sub-component C, shown by a `<<delegate>>` dependency from component C to a shaded box on A's boundary. The box is called a **port**.
  - Sub-component D delegates the required interface on B's port.
  - Components use **ports** and **delegates** to model how components interact with other components.

> [!note] Verification Note (Delegation Direction)
> Technically, the outer component (A or B) delegates the provided/required interface at its boundary port to its internal sub-component (C or D) via a `<<delegate>>` connector.

- A component diagram breaks down the actual system into various high levels of functionality.
- Each component is responsible for **one clear aim** and only interacts with others on a **need-to-know basis**.

> [!example] Sample Component Diagram — `ClientCollectionSystem`
> - `ClientCollectionSystem` has 3 subcomponents: `ClientManager`, `SensorManager`, `VideoManager`.
> - `ClientCollectionSystem` needs an interface called `ISchedule` (via Port 2), used by `ClientManager`, so a required connector (`<<delegate>>`) is drawn.
> - The system provides `IClientData` (via Port 1), provided by `ClientManager`, so a provider (`<<delegate>>`) is drawn.
> - Internally, `ClientManager` interacts with `SensorManager` via `ISensorControl` and `VideoManager` via `IVideoControl`.
> ![[Pasted image 20261120211746.png]]

> [!example] Example 3 — BlogApp Site (Class Diagram to Component Diagram)
> 3 components can be created from the class diagram:
> 1. **Notification:** `EmailNotification`, `SMSNotification`
> 2. **AccountManagement:** `User`, `UserRegistration`
> 3. **BlogPosts:** `Post`, `Comment`, `Paginate`, `PostManagement`, `CommentManagement`
>
> *Components can be any encapsulated unit in a system.*
![[Pasted image 20261120211800.png]]
### Project Guideline to Creating Components
1. What are the **main responsibilities** of the system?
2. Which **classes belong together because they serve the same responsibility?**
3. What does each **component need from another** component?
4. **Who provides that service and who requires it?**
5. Does my **component diagram still align with my class diagram** and the rest of my system diagram?
6. **New interface classes** are probably needed to be modelled in the component diagram.
7. Class diagram may not have a **database inside**, but classes may need to interact with it.

### Common Mistakes
- Components are higher-level abstractions than objects and encapsulate internal structure; **do not provide too much detail**.
- **no dangling lollipops or sockets** with no connections unless it is to external components.
- **Do not share lines** in the notation.
- Use a **port** when you want to represent a point of interaction
- **Component needs both lollipop and socket**
- User interface / dashboard **do not need to be a separate component**.
- A component **can have both** a required interface and a provided interface.
- A **lollipop can be shared** by more than one component.

---

## Software Engineering Design

### Definition
- The process of **identifying, evaluating, validating, and specifying** the architectural, detailed, and construction models required to build software that meets its intended **functional and non-functional requirements**, AND the result of such process.

### Importance of Software Design (Product Development Perspective)
- **Requirements are mapped to conceptual models** of software:
  - Provide models that represent structure and behavior.
  - Main components and interfaces are identified.
- **Quality of future systems** is there:
  - Provide means to evaluate quality attributes like usability.
- Design forms the **foundation** for all other development activities such as construction, testing, maintenance.
- Solutions can be **reused** in other projects.
- Design helps **improve project management**:
  - Planning, organization, staffing, and tracking.
- Increases **efficiency in human resource allocation**:
  - Designs decompose systems so teams can work on different parts in parallel.
  - Decomposing can allow teams to work in a distributed fashion.
- **Shields the project from having "one guy" own the whole system**:
  - Multiple members own multiple components; if someone leaves, it is easier to replace.
- Good software design **minimizes effects of requirements volatility**:
  - Minimize impact on schedule and cost.

### Software Design Challenges
- Requirements volatility
- Inconsistent development process
- Fast ever-changing tech
- Ethical and professional practices
- Managing design influences

### Software Design Process
- A set of activities and controls that specify how resources work together for the production of software design.

#### AI in Design Process
| Step | AI Suggests | Engineer Checks |
| :--- | :--- | :--- |
| **1. Clarify needs** | Questions, missing cases | Real needs and constraints |
| **2. Explore options** | Components and interfaces | Fit and trade-offs |
| **3. Evaluate designs** | Risks and test ideas | Evidence of system quality |
| **4. Refine design** | Design revisions and prototypes | Changes and final decisions |

### Software Design Activities
1. **Software Architecture:**
   - **Macrodesign** approach for creating models that depict quality and function of software system.
   - Provides **black-box models** used to evaluate system's capabilities and lay the foundation for all subsequent work in the life cycle.
   - Focus on **NFR** (Non-Functional Requirements).
2. **Detailed Design:**
   - Builds on software architecture to provide **white-box** design elements of the structure and behaviour of system.
   - Activity that deals with refining software architecture to reach a point where software design (including architecture and detailed design) is deemed sufficient for construction to begin.
   - Focus on addressing **FRs** (Functional Requirements) with interfaces and component design.
3. **Construction Design:**
   - Further elaborates detailed design.
   - To support system's quality attributes.
4. **Human Computer Interface:**
   - General principles are applied to optimize the interface between humans and computers.
5. **Software Design Documentation:**
   - Include necessary info that properly captures the design of the system.
6. **Software Design Management:**
   - Activities required to create and implement quality design artefacts, within schedule and budget.

> [!tip] Key Distinction: Software Architecture vs. Detailed Design
> - **Software Architecture:**
>   - *Structural:* provides major structural components and interfaces of the system.
>   - *Communication:* serves as important communication, reasoning, and analysis tool that supports the dev and growth of system.
>   - *Quality:* focuses on quality aspects of the system before detailed design or construction begins (**NFR**).
>   - *Foundation:* for all subsequent work in the software engineering lifecycle (**Black-box**).
> - **Detailed Design:**
>   - *Internal design:* focuses on internals of those components and interfaces.
>   - *White box:* builds on software architecture to provide white-box approach to design the structure and behavior of system.
>   - *Functional:* focuses on **FR** whereas architecture focuses on **NFR**.
>   - *Follows architecture:* refines the architecture so software design is complete for construction to begin.
> - *Note:* In practice, design activities are **not** carried in sequence; different iterations occur between them.

### Software Design Fundamentals — Design Principles
- **Modularization:** Process of continuous decomposition of system until fine-grained components are created. Abstraction and encapsulation help with this.
- **Abstraction**
- **Encapsulation**
- **Coupling and Cohesion:**
  - **Cohesion:** defined as all elements of a module, class, or component working together as a unit (High cohesion = good).
  - **Coupling:** degree of interdependence between two or more classes, modules, or components. **Tight coupling = bad** (components highly dependent on others) and vice versa (loose coupling = good).
- **Separation of Interface and Implementation:** Specifying a public interface (known to client) that is separate from details of the component.
- **Sufficiency:** Measures how well designed units are at providing services sufficient to achieve intent (**no more**).
- **Completeness:** How well designed units provide required services to achieve the intent (**no less**).

---

## Overview of Architecture

- **Stakeholder concerns** serve as the driving force behind architectural decisions.
- When deciding the architecture:
  - Functional and Non-functional requirements dictate what to do.
  - Manage trade-offs (e.g., weight vs speed).
  - Adhere to **software quality requirements**:
    - **Usability:** complexity when learning/using system.
    - **Maintainability:** complexity in fixing issues or making changes without affecting current working components in system.
    - **Security:** ability to protect information/system.
    - **Performance:** capacity to accomplish useful work under resource constraints.
    - **Reliability:** system's failure rate.
    - **Portability:** complexity when adapting system to other software/hardware.
    - **Testability:** complexity when verifying and validating system functions.
    - **Availability:** system's uptime.
    - **Interoperability:** system's ability to collaborate with other hardware/software.

> [!example] Architecture Decision Example (Blog Application)
> | Mobile Web | Native App |
> | :--- | :--- |
> | Works on any mobile browser | Need an app for each platform |
> | Needs to be Online | Can be Offline |
> | Slower, lagging interface | Faster, smooth interface |
> | Small size | Could take up space |
> | Easier to update web app | Pushing updates to all versions |
> | Cheaper | More expensive to develop and maintain |
> | Little client hardware used | Use hardware capabilities of device (like sensors) |

---

## C4 Model

**C4 = Context, Container, Component, Code**

- A starting point for abstraction.
- Simplified version of UML and 4+1 Model.
- Architectural documentation standard for software systems that breaks a system into **4 levels**.
- Makes it easier for developers to describe how a system works.
- Minimizes gap between software architecture model and source code.
- **System Context and Container diagrams** are usually sufficient for most teams.

### The 4 Levels (Internet Banking System Example)
1. **Level 1: System Context Diagram**
   - Shows the software system in context with users (e.g., Personal Banking Customer) and external software systems (e.g., E-mail System, Mainframe Banking System).
2. **Level 2: Container Diagram** *(zoom in of a Context)*
   - Shows high-level technical building blocks / containers inside the system (e.g., Web Application, Single-Page Application, Mobile App, API Application, Database) and protocols (HTTPS, JSON/HTTPS, JDBC, SMTP, XML/HTTPS).
3. **Level 3: Component Diagram** *(zoom in of a Container)*
   - Shows components inside a specific container (e.g., inside API Application: Sign In Controller, Reset Password Controller, Accounts Summary Controller, Security Component, E-mail Component, Mainframe Banking System Facade).
4. **Level 4: Code** *(zoom in of a Component)*
   - Shows UML class diagrams / code-level details of a component (e.g., `MainframeBankingSystemFacadeImpl`, `GetBalanceRequest`, `BankingSystemConnection`, etc.).

- From this, we can see that **component diagrams** and **class (code) diagrams** are also part of software architecture; it also includes **sequence diagrams**.
- Different levels and views to the architecture suit all stakeholders.
- A good architecture diagram clearly communicates what it needs to, to who it needs to communicate to.
- **Zoom-in hierarchy:** `Code` $\subset$ `Component` $\subset$ `Container` $\subset$ `Context`.

---

## 4+1 View of Architecture

- Model for documentation of software architecture.
- Decisions made can be organized around **4 views** (Logical, Development, Physical, Process), then illustrated by a few selected **use cases (fifth view)**.
- Communicates architecture to different stakeholders.

| View | Focus / Perspective | UML Diagrams Used |
| :--- | :--- | :--- |
| **Logical view** | Functionality that the system provides to end users | Class, state, component diagrams |
| **Process view** | Dynamic aspects of the system, system processes and how they communicate; focuses on the run-time behaviour of the system | Sequence, communication, activity diagrams |
| **Development / Implementation view** | System from a programmer's perspective; concerned with software management | Package diagram and source code |
| **Physical / Deployment view** | System from a system engineer's POV; concerned with topology of software components on the physical layer as well as physical connections between these components | Deployment diagram |
| **Use case / Scenarios view (+1)** | Describes sequences of interactions between objects and between processes. Used to identify architectural elements, illustrate and validate architecture design, and serve as a starting point for tests of an architecture prototype | Use case diagrams / scenarios |

### Key Tasks in Architectural Design
- Identify:
  - Stakeholders' concerns
  - Appropriate architectural views
  - Architectural styles and patterns
  - Influences of architectural decisions in orgs
  - System major components and interfaces
- Evaluate and validate architecture
- Establish policies for ensuring architectural design synchronicity

---

## Architectural Patterns

- A **solution to a recurring problem in a context**.
- In software, we see similar problems like: access to a DB, CRUD operations in DB, creating objects, etc.
- Use existing solutions and ideas; patterns provide generic, reusable solutions that are easily understood.

### Common System Types & Suitable Patterns
| System Type | Description | Example | Suitable Patterns |
| :--- | :--- | :--- | :--- |
| **Data centered** | Systems that serve as a centralized repository for data, while allowing clients to access and perform work on the data | IDEs | Blackboard pattern |
| **Data flow** | Systems oriented around the transport and transformation of a stream of data | Sensor Data processing system | Pipe and Filter pattern |
| **Distributed** | Systems that primarily involve interactions between several independent processing units connected via a network | Email system | Client-Server, Peer to peer, Broker |
| **Interactive** | Systems that serve users or user-centric systems | e-Shopping website | Model-View-Controller (MVC) |
| **Hierarchical** | Systems where components can be structured as a hierarchy to reflect different levels of abstraction | Android OS | Main program and subroutine, Layered Pattern |

### 1. Hierarchical — Layered Software Architecture Pattern
- Most common pattern.
- Components or classes are organized into **horizontal layers**; each layer performs a specific role within the system.
- **Layers:**
  - **Presentation layer:** Logic here focuses on user interface concerns, devoid of business logic for easier tests.
  - **Business / Application layer:** Holds implementation of business logic of app, including business rules, validations, and calculation logic. Business entities for application's domain are placed here.
  - **Data / Persistence layer:** Provides functionality to access and manage data. Layer contains a data store for persistent storage. Provides services for business layer.
- Each layer is a level of abstraction around a role and is not concerned with how other layers work.
- **Layers are closed:** a request has to move layer to layer sequentially (no skipping).
- **Layers of isolation:** changes made in one layer usually do not impact components in other layers.
- **Disadvantages:**
  - More code necessary for layered apps (to provide interfaces and other logic necessary for comms).
  - Development teams have to be diligent about placing code in the correct layer to not leak logic to others.
  - Inefficiencies in having requests go through multiple layers; moving layer to layer may require data representations to transform.

### 2. Distributed — Client-Server Pattern
- Popular pattern in today's modern systems.
- Decomposes software systems into two main components: **Client** and **Server** (2-Tier Architecture).
  - Client sends `<<Request>>` (requests resources or calls some service).
  - Server sends `<<Response>>` (responds to the requests of the client).
- There may be multiple clients connected to a single server.

> [!example] Application — Email System
> - Two components: `EmailClient`, `EmailServer`.
> - `EmailServer` provides two separate services to client, implemented by two subcomponents (delegated via ports):
>   - **Retrieval:** provides functions for client to access and download emails stored on server via **IMAP**.
>   - **Transport:** allows client to submit emails for delivery to other users via **SMTP**.

### 3. Interactive — Model-View-Controller (MVC)
- Used in interactive apps that require flexible incorporation of human computer interfaces.
- Decomposes apps into three components that handle system's input, processing, and output:
  - **Model:** Contains domain-specific knowledge. Often linked to a database. One model can link to different views. Multiple layers can be used at the model side. *(Sends state changes to View)*
  - **View:** Presents data to the user. Allows user interaction. **Does no processing**. *(Sends user actions to Controller)*
  - **Controller:** Controls the flow of information between the model and the view. Receives messages from the view (where events come from) and sends messages to the model (tells what data to display / updates model / selects view).
    - *Note:* Controllers only coordinate actions and messages, unlike control classes which have application logic.

> [!example] Application of MVC — Real Estate Housing Market System
> - Allows users to find properties for sale.
> - Composed of two views (`SmartPhoneView`, `PCView`), each with their own controller (`SmartPhoneController` via `ISpeechController`, `PCController` via `IMouseController`), and one model (`RealEstateModel` via `IModel`) that contains information.
> - Both views are designed uniquely for usability on PC (interactive buttons/menus) and smartphone (smaller, supports speech recognition).

---

## Design Patterns (Gang of Four)

### 1. Creational Patterns — *for instantiating objects*
- **Factory:**
  - *Problem:* Abstract class has many concrete subclasses; abstract class cannot be instantiated; need to create and store many instances for the abstract class based on a concrete class; need to add new concrete subclasses.
  - *Solution:* A factory class responsible for instantiating other classes.
  - *Intent:* To provide a method by which objects can be created, allowing subclasses to decide which concrete class should be used.
  - *Applicability:*
    - When the concrete type of an object must be determined at run time.
    - When more concrete types might develop over time.
    - To encapsulate the logic of choosing the appropriate concrete type in a single function.
  - *How it works:*
    - `Factory.CreateProduct()` method, usually static.
    - Client invokes `Factory.CreateProduct()` with parameters: `IProduct prod = Factory.CreateProduct("A")`.
    - `Factory.CreateProduct()` decides which subclass to instantiate: `switch (productType) { case "A": return new ProductA(); ... }`
- **Abstract Factory:** Create families of related objects that belong together.
  - *Ex:* An app offers light and dark themes; selecting a theme produces matching buttons, menus, and dialogs.
- **Builder:** Assemble a complex object step by step.
  - *Ex:* A travel booking lets users choose flights, seats, baggage, and meals before producing one booking.
- **Prototype:** Make new objects by copying an existing one.
  - *Ex:* A slide editor duplicates a formatted slide so the user can change its content without rebuilding.
- **Singleton:** Ensure only one instance of a class exists.
  - *Applicability:* Providing access to unique resources like the console; grouping global variables.
  - *When to use:* School app needs one config manager so every screen uses the same settings.

### 2. Structural Patterns — *for managing relationships between objects*
- **Adapter:** Make two incompatible interfaces work together.
  - *Ex:* A payment system expects one standard interface, but a new payment system uses different method names.
- **Bridge:** Separate what something does from how it is implemented.
  - *Ex:* A remote control should work with TVs without needing a separate remote class for each brand.
- **Composite:** Re-use existing classes in new context and build complex structure from simpler components; treat one item and a group of items in the same way.
  - *Ex:* In file explorer, can delete or copy a file or entire folder using the same action.
- **Decorator:** Add optional features without changing original class.
  - *Ex:* Coffee ordering system adds milk, etc. and updates price for the extra.
- **Facade:** Offer one simple entry point to a complicated process.
  - *Ex:* "Place order" button handles payment, inventory checks, delivery booking behind the scenes.
- **Flyweight:** Share reusable data instead of storing many duplicates.
  - *Ex:* Game displays thousands of trees while sharing each tree type's texture and shape.
- **Proxy:** Place a stand-in in front of another object to control access.
  - *Ex:* A photo gallery shows thumbnails first and loads full photos only when opened.

### 3. Behavioural Patterns — *for implementing common behaviours*
- **Chain of Responsibility:** Pass a request along handlers until one handles it.
  - *Ex:* An IT helpdesk routes a ticket from a chatbot to support staff to specialist.
- **Command:** Turn an action into an object that can be stored or reversed.
  - *Ex:* Drawing app records "draw line" or "delete shape" actions so users can undo them.
- **Interpreter:** Define rules for understanding a small language.
  - *Ex:* A search tool understands simple filters such as `"price < 50 AND category = books"`.
- **Iterator:** Visit items one by one without exposing how it is stored.
  - *Ex:* A music player goes through songs using Next, whether songs are in a list or other structure.
- **Mediator:** Use a central coordinator instead of many direct connections.
  - *Ex:* In a chatroom, users send messages to the chat server rather than connecting to everyone.
- **Memento:** Save an object's state so it can be restored later.
  - *Ex:* Text editor saves checkpoints so a user can return to an earlier version of a document.
- **Observer:** A class that generates events maintains a list of observers that will be notified when an event arrives.
  - *Ex:* Online shop changes an order to shipped, the app and email notifier update automatically.
- **State:** Change object's behaviors when its current state changes.
  - *Ex:* Vending machine responds differently when it is idle, received payment, or dispensing.
- **Strategy:** Swap between different ways of doing the same task.
  - *Ex:* Maps app can calculate quickest, shortest, or toll-free route based on choice.
- **Template Method:** Keep common workflow steps fixed, but let subclasses customise.
  - *Ex:* Different report types follow `load data > analyse > export`, but `analyse` works differently for different reports.
- **Visitor:** Add new operation across different object types without changing each type.
  - *Ex:* Shopping cart calculates tax for books, groceries, and electronics using a separate tax calculator.

---

## Non-Execution-Based Testing

Checking for quality **without running test cases** (static analysis).

- **Underlying principles:**
  - We should **not review our own work**.
  - **Group synergy**.

### Techniques
1. **Walkthrough:**
   - A static analysis technique in which a designer or programmer leads members of the development team through a software product, and participants ask questions and make comments about possible errors, violations, etc.
   - *Objectives:* find anomalies, improve the software product, consider alternatives, evaluate conformance to standards and specs.
   - **Two-step informal process:** `preparation` $\rightarrow$ `analysis`.
2. **Inspection:**
   - To detect and identify software product anomalies.
   - Systematic examination that verifies:
     - The software product satisfies its specs, specified quality attributes, and conforms to applicable regulations, standards, guidelines, plans, procedures.
     - Identifies deviations from standards and specs.
     - Collects software engineering data (optional).
     - Uses collected data to improve inspection process (optional).
   - *Metrics:*
     - Inspection rate (e.g., design pages inspected per hour).
     - Fault density (e.g., faults per KLOC inspected).
     - Fault detection rate / efficiency.
     - *Discussion question:* Does a 50% increase in fault detection rate mean quality has decreased, or inspection is more efficient? *(Ambiguous without context!)*
   - **Five-step formal process:** `overview` $\rightarrow$ `prep` $\rightarrow$ `inspection` $\rightarrow$ `rework` $\rightarrow$ `follow up`.
3. **Technical Review / Peer Review**

---

## Execution-Based Testing

- Testing via **running code**.
- **No false positives** — testing directly observes actual behavior of code.
- Can be automated.
- **Definition:** Process of inferring certain behavioral properties of the product based on results of executing product in known environment with selected inputs.

### What to Test
- **Correctness:** A product is correct if it satisfies its specs.
  - *Note:* Specs may be incorrect, or correct but incomplete, allowing for incorrect implementations to meet specs.
- **Utility:** Extent to which the product meets user needs (ease of use, useful functions).
- **Reliability:** Measure of frequency and criticality of failure (mean time between failures, time and cost to repair the results of failure).
- **Robustness:** Function of the range of operating conditions and the effect of invalid input.
- **Performance:** Extent to which space and time constraints are met (may affect correctness, robustness, reliability).

---

## Blackbox Testing

- Also known as **functional testing** or **behavioural testing**.
- Tests the observable functionality of an application; validates conformity to the specifications.
- Introduces a series of inputs to a system and compares the outputs against pre-defined test specifications.
- Applicable to **all forms of testing**: unit, integration, system, acceptance.
- Tests are **architecture independent**.
- Given a chosen input, we do not care how the output is produced, only whether that output is the desired and expected output.
- Tests can be functional or non-functional.
- Designing the tests requires **no knowledge of underlying system** and does **not require a software engineer or programming skills**.

### Blackbox Testing Techniques
*(Different techniques provide different guarantees/assurance regarding the system under test.)*

#### 1. Equivalence Partitioning
- Divide input values into **valid and invalid partitions** and select representative values from each partition to test.
- Inputs are divided into different equivalent data classes.
- Each partition is tested **once** as all inputs within it are considered equivalent.
- **How many partitions?** Depends on range of all possible input values and what function is tested; goal is to come up with **as few partitions as possible** that cover the whole range of possible inputs.
- **How to choose specific test value?** Randomly as long as test value is within the partition (different from BVA where you must choose boundary values or near boundary).

> [!example] Equivalence Partitioning — Percentage Form (Accepts 50–90%)
> - Invalid partition 1: $< 50$ $\rightarrow$ choose `30` (reject)
> - Valid partition: $50\text{--}90$ $\rightarrow$ choose `60` (accept)
> - Invalid partition 2: $> 90$ $\rightarrow$ choose `100` (reject)

#### 2. Boundary Value Analysis (BVA)
- Similar to equivalence partitioning, but test values are at the **boundaries** (and near boundaries).
- More application errors occur at boundaries.

> [!example] BVA — Percentage Form (Accepts 50–90%)
> - Choose: `49` (reject), `50` (accept), `51` (accept), `89` (accept), `90` (accept), `91` (reject).

#### 3. Decision Table
- A tabular form that presents a set of **causes** and their corresponding **effects**.
- Consists of a number of columns (**rules**) that comprise all test situations.
- **Steps:**
  1. List all causes in the decision table.
  2. Calculate the number of possible combinations: $\text{values}(\text{cause}_1) \times \text{values}(\text{cause}_2) \times \dots \times \text{values}(\text{cause}_n)$.
  3. Fill columns with all possible combinations (inputs).
  4. Add effects to the table (expected outputs).
  5. Reduce test combinations (based on indifferent combinations).
  6. Check covered combinations.

> [!example] Decision Table 1 — Login (Email & Password)
> - Causes: Email (Y, N), Password (Y, N) $\rightarrow$ $2 \times 2 = 4$ combinations.
> - Reduced from **4 test cases to 3** based on indifferent combination (if no email `N`, state of password doesn't matter `-`).
>
> | Causes | Description | Values | 1–2 | 3 | 4 |
> | :--- | :--- | :--- | :---: | :---: | :---: |
> | **C1** | Email | Y, N | N | Y | Y |
> | **C2** | Password | Y, N | - | N | Y |
> | **Effects** | | | | | |
> | **E1** | Accept | | | | X |
> | **E2** | Reject | | X | X | |

> [!example] Decision Table 2 — Account Withdrawal
> - Causes: A/C Exists (Y/N), Pin Correct (Y/N), Balance Available (Y/N) $\rightarrow$ $2 \times 2 \times 2 = 8$ combinations.
> - Reduced from **8 test cases to 4**:
>
> | Causes | Description | Values | 1–4 | 5–6 | 7 | 8 |
> | :--- | :--- | :--- | :---: | :---: | :---: | :---: |
> | **C1** | A/C Exists | Y/N | N | Y | Y | Y |
> | **C2** | Pin Correct | Y/N | - | N | Y | Y |
> | **C3** | Balance Available | Y/N | - | - | N | Y |
> | **Effects** | | | | | | |
> | **E1** | Withdrawal | | | | | X |
> | **E2** | Denied | | X | X | X | |

- Once we reduce the decision table, come up with specific test cases to be passed to software testing engineer (tester does not need to be a programmer, just needs to follow a test case).
- **Limitation:** Decision table is **not suited to systems that have a large number of causes** (combinatorial explosion).
- Actual result and pass/fail columns are filled after running these tests.

---

## Whitebox Testing

- Blackbox considers only inputs and outputs; **whitebox testing looks inside the box — at the code**.
- Both are **execution-based** tests.
- **Other names:** structural, logic-driven, clear/open/glass box testing.
- **Complementary techniques:**
  - Blackbox **cannot** reveal errors due to implementation details.
  - Whitebox **cannot** reveal errors due to missing paths.
- **Goal:**
  - Uncover as many bugs in a small, carefully selected subset of the input domain.
  - Process every single character/statement in code and test for errors (usually impractical to cover entire input domain).
  - Execute a faulty statement to reveal that it is faulty.

### Levels of Testing
- **Unit testing:** testing individual modules.
- **Integration testing:** tests between units, interactions among modules.
- **Regression testing:** tests performed every time we change our system to ensure changes behave as expected and unchanged code doesn't behave differently.
- **Acceptance testing**
- **Load testing**
- **Alpha / Beta testing**

### Whitebox Technique — Coverage Testing
- **Coverage criteria:** standards used to measure how thoroughly the code has been tested; provides systematic way to design test cases; finds gaps in testing and improves software reliability.
- **Test requirements:** what are the interesting cases we can identify in the code, based on what we want to cover.
- Results in:
  - **Test specifications:** what must our inputs be to get these interesting cases to be executed.
  - **Test case:** concrete values applied based on our test specification logic, with expected output.
- *Important:* Achieving **100% coverage does not guarantee the absence of bugs**, but increases confidence.

#### 1. Statement Coverage
- Focuses on verifying whether **each statement in the code executes at least once**.
- Does **not** guarantee coverage of all possible execution paths, as it doesn't test different combinations of conditions.

> [!example] Statement Coverage Example
> ```python
> def example_function(x, y):
>     if x > 0:
>         print("X is positive")
>     if y > 0:
>         print("Y is positive")
>     if x > 0 and y > 0:
>         print("Both X and Y are positive")
> ```
> - You can cover **all statements** with a single test case: $x = 1$ and $y = 1$.
> - However, there are **4 paths**:
>   1. $x > 0$ and $y > 0$
>   2. $x > 0$ and $y \le 0$
>   3. $x \le 0$ and $y > 0$
>   4. $x \le 0$ and $y \le 0$

#### Control Flow Graphs (CFG)
- Helps visualize both statements and flow of control in a module.
- **Nodes:** represent statements.
- **Edges:** represent flow of control within the code.
- **Branch:** when a node has more than one outgoing edge, each edge is referred to as a branch.
- **Path:** routes we can take from start to finish.
- **Structured Programming Constructs:** Sequence, If-Then, If-Then-Else, While Loop, For Loop, Select Case.

#### Cyclomatic Complexity ($M$)
- Software metric used to determine the complexity of a program.
- Number of linearly independent paths through a CFG:
$$M = E - N + 2P$$
  - $E =$ number of edges
  - $N =$ number of nodes
  - $P =$ number of connected components (just $1$ for simplicity)

> [!warning] Verification Note (Cyclomatic Complexity in Notes)
> 1. In the `CheckSavings` example (`if (result >= 500) ... else if (result < 500) ...`), your notes state $7 - 7 + 2(1) = 2$. Strictly speaking, because of the `else if`, the CFG has **8 edges** (including the False edge from node 5 to exit node 7) and 7 nodes, giving $M = 8 - 7 + 2(1) = 3$ (though the False branch of `result < 500` is logically impossible, leaving 2 feasible paths). If it were a plain `else`, it would be $E=7, N=7 \implies M=2$.
> 2. On page 21, the formula had a typo `$M = E - N + 2PS$` — the correct formula is `$M = E - N + 2P$`.

#### 2. Branch Coverage
- Statement testing may not reveal all possible paths that the code may take.
- Branch coverage attempts to **explore all possible branches** (True/False of every decision).
$$\text{Branch Coverage (\%)} = \frac{\text{Number of branches executed and tested}}{\text{Total number of branches}} \times 100\%$$

#### 3. Path Coverage
- Generate coverage such that **all paths** in the program are covered.
- Expensive — any non-trivial program may have a virtually infinite number of paths.

### Whitebox Technique — Control Flow Testing
- Tests all possible independent paths in the control flow graph of a program; best executed for procedures.
- Guarantees every statement of the program has been executed at least once and every transition is tested.
- Helps figure out **how many test cases are needed** and **how to construct each test** (what to test).
- **Basis Paths:**
  - A set of paths such that any other path can be constructed out of a combination of basis paths.
  - Designed to test all possible paths at least once.
  - Results in **100% statement coverage** and **100% branch coverage** (includes all edges of the graph).
  - Size of the set is related to Cyclomatic Complexity ($M$) — **$M$ is the upper bound** of the number of paths in the set of basis paths.

#### How to Determine a Set of Basis Paths
1. Pick a **baseline path** that corresponds to normal execution (aim to have as many decisions as possible).
2. To get succeeding basis paths, **back-track** the baseline until you reach a decision node. **"Flip the decision"** (take the alternative path) and continue as much of the baseline as possible.
3. Repeat until all decisions have been flipped. When you reach $M$ (Cyclomatic Complexity) basis paths, you're done!

> [!example] Basis Path Examples
> - **Graph 1:** $M = E - N + 2P = 9 - 8 + 2(1) = 3$. Minimum tests needed for nodes/edges: 2, complete basis path test requires 3 paths (e.g., `ABDEFH`, `ABDEGH`, `ACDEFH`). *(Note: original PDF truncated the 3rd path as `ACDE`.)*
> - **Graph 2:** $M = 14 - 12 + 2(1) = 4$.
>   - Path 1: `A-B-C-E-F-G-J-K-L`
>   - Path 2: `A-B-C-E-F-H-J-K-L`
>   - Path 3: `A-B-C-E-F-I-J-K-L`
>   - Path 4: `A-B-C-D-K-L`

- **Test Case (IEEE Definition):** A set of test inputs, execution conditions, and expected results developed for a particular objective, such as to exercise a particular program path or to verify compliance with a specific requirement. Practically, a set of instructions for running a specific test.

### Coverage Types Comparison
| Coverage Type | What it Requires | What it Ensures | Difficulty Level |
| :--- | :--- | :--- | :--- |
| **Statement Coverage** | Execute every statement at least once | Code lines are tested | Easy |
| **Branch Coverage** | Execute each branch (True/False) of every decision | Decision logic is tested | Medium |
| **Path Coverage** | Execute all possible paths through the code | Full logic combinations tested | Hard (often impractical) |

### Whitebox Testing: Advantages & Disadvantages
- **Advantages:**
  - Attempts to test all functionalities.
  - Identifies errors in hidden code and makes debugging process swift.
  - Removes extra lines of code which are not required, optimizing and increasing efficiency of program.
  - Helps in evaluating all loops and paths.
  - Provides stability and usability of test cases.
  - Thoroughness achieved in whitebox testing is far more than blackbox.
- **Disadvantages:**
  - Testers must have knowledge of programming, so it is expensive.
  - 100% coverage is rarely possible; bugs can be missed.
  - Changes in code may require all scenarios to be tested again.
  - Developing test cases takes time.
  - If application is large, complete testing is not feasible.

---

## User Acceptance Testing (UAT)

- **UAT is NOT system testing.**
- System testing (unit, functional, integration) should be completed by this phase — UAT is **not looking for major errors**.
- *System testing* ensures software does not crash and meets requirements (blackbox inputs/outputs, whitebox thoroughness, resource consumption & load times).
- **What's left BEFORE release is UAT.**
- **Definition:**
  - UAT is the **last phase** of the software testing process.
  - During UAT, **actual software users** test the software to make sure it can handle required tasks in **real-world scenarios**, according to specifications.
  - UAT is one of the final and critical software project procedures that must occur before newly developed software is rolled out.

---

## Application Packaging

Process of developing a final deliverable to:
- Ensure consistent, stable, and reliable standard environment.
- Increase the efficiency of software management by streamlining software deployment along with any customization needed.
- Mitigate security issues.
- Decrease risks for business disruption.
- Help reduce ongoing administration and support costs.

**In simple terms:**
- Document process of installation (`README`).
- Ensure process of installation of system is documented for all supporting tech and devices.
- All prerequisites included as part of install package.
- Auto installer included (one click to install).
- Any required supporting documents, manuals during installation.
- Any licensing agreements visible to user.

---

## Software Configuration Management (SCM)

### Software & Change
- **Moving target problem:** Change in requirements while software is being developed.
- **Change is inevitable:**
  - New requirements emerge when software is used.
  - Business environment changes.
  - Errors must be repaired.
  - New equipment must be accommodated.
  - Performance or reliability may have to be improved.
- Managing software changes is more challenging if dev effort is distributed between devs in different countries and time zones.

### Uncontrolled Change
- *Scenario:* Two employees work on the same project simultaneously on local copies and copy changes back to central server. Person A adds on to `function A`, while Person B removes `function A`. Person B pushes first, then Person A pushes — code fails to compile since `function A` doesn't exist.
- **Causes of uncontrolled change:**
  - Two or more devs working on parallel dev effort.
  - Team permits uncontrolled change to source code master.
  - No agreed system of preserving previous working versions.
  - Team did not have a procedure in place for notifying others of a change.

### SCM Definition
- Also known as **change management**: a set of activities designed to manage change by **identifying work products likely to change**, **establishing relationships among them**, **defining mechanisms for managing different versions** of these work products, **controlling the changes imposed**, and **auditing and reporting** on the changes made.
- SCM is the control of evolution of complex systems, for the purpose of contributing to satisfying quality and delay constraints.

### Configuration Item (CI)
- An **approved and accepted deliverable** associated with a software project:
  - Problem definition and requirements specification
  - Software design and implementation
  - Test plan and harness
  - User documentation
  - System dependencies
  - Associated development and management tools
- **What NOT to track:** personal config files, auto-generated files, compiled binaries, large asset files.
- **When to track:** From the **beginning of the project until the end**.

### Change Control
- Intended to ensure that system evolution is a managed process.
- Priority is given to the most urgent and cost-effective changes.
- **Concerned with:** analyzing costs and benefits of proposed changes, approving worthwhile changes, tracking which components changed.
- **Process:**
  1. Submission of change request (CR).
  2. Decision: technical and business evaluation and impact analysis $\rightarrow$ approval of Change Control Board (CCB).
  3. Engineering change order is generated stating: commitment and evaluation, configuration items check out, changes made and reviewed, configuration items checked in.
- **Factors of Change Analysis:**
  - Consequences of not making the change
  - Benefits of the change
  - Number of users affected by the change
  - Cost of making the change
  - Product release cycle

#### Change Control Form & Workflow
1. **Section A:** Change requester completes all boxes (Project, Controlled item, Aspect to be changed, Change details/urgency, Requester name/date) and passes to Project Manager.
   - *Flow:* Identify items for change $\rightarrow$ Submit CR $\rightarrow$ CR Complete? (No $\rightarrow$ Inform Requester; Yes $\rightarrow$ Change Analysis).
2. **Section B & C:** Project Manager arranges investigation of request. Depending on outcome, request is rejected or given priority and costs, and with investigator completes **Section B** (Investigator, Impact, Investigation Outcome, Suggested Priority) and **Section C** (Implementor, Date Scheduled).
   - *Flow:* CR Feasible? (No $\rightarrow$ Inform Requester; Yes $\rightarrow$ Assign Developer $\rightarrow$ Implement Change).
3. Form is retained in project files.
4. **Section D:** Once change is implemented and verified OK (if No $\rightarrow$ back to Assign Developer), **Section D** (Implementor & Project Manager signature/date) is signed off $\rightarrow$ **BASELINE**.

#### Change Control When AI Contributes Code
- **Source of change:** Treat AI-generated code as a proposed change that must address requirements.
- **Impact analysis and review:** AI can summarise pull requests and suggest risks; devs verify suggestions and inspect changes.
- **Approval:** Require **human review and approval** before merge.
- **Traceability:** Link changes to its request and record AI assistance, review decisions, and verification evidence.

---

## Version Management

- Process of keeping track of different versions of configuration items.
- Involves ensuring changes made by different devs do not interfere with each other, resolving conflicts, and enabling collaboration.

### Terminologies
| Term | Definition |
| :--- | :--- |
| **Version** | A configuration item at one point in its development; differs in some way from itself at another point. A version has **revisions** and **variants**. |
| **Revision** | A config item linked to another via a `revision-of` relationship, and **ordered in time** (e.g., 1.2 $\rightarrow$ 1.3 $\rightarrow$ 1.4). |
| **Variant** | **Functionally equivalent** versions, but designed for **different settings** (e.g., hardware and software). |
| **Repository** | Shared database of versions of configuration items and meta-information about changes to items. |
| **Workspace** | Isolated, private environment where a dev can work without interfering with and affecting other devs. |
| **Check out** | Action of copying items from **repository to workspace**. |
| **Check in** | Action of copying items from **workspace to repository**. |
| **Branching** | Creation of a new workflow from an existing version; new flow developed independently. |
| **Merging** | Combining changes to a file simultaneously made by different users. |

### Version Management in AI-Assisted Development
- **Create a checkpoint:** Commit the current working version before asking AI to make changes.
- **Keep changes manageable:** Use a separate branch and organise AI-generated edits into small, meaningful commits.
- **Track supporting artifacts:** Version source code, tests, dependency lockfiles, and shared AI instruction files.
- **Check conflicting changes:** Changes can merge successfully but still make incompatible assumptions about system behavior.

---

## System Building

- Process of creating a **complete, executable system** by **compiling and linking** the system components, external libraries, and configuration files.

### In AI-Assisted Development
- **Build configuration:** AI can draft build scripts and config files; check that they use the correct commands, paths, and runtime version.
- **Dependencies:** AI-generated code may introduce new packages; verify that they exist, are suitable, and have compatible versions recorded in dependency files.
- **Build failure diagnosis:** AI can explain error logs and suggest fixes; verify the cause and rebuild after applying fix.
- **Consistent builds:** Build from clean checkout using declared dependencies and documented config.

---

## Release Management

- Formal distribution of an approved version — an instance of a system which is **distributed** *(fixed from "distrusted")* to users outside of the dev team.
- **Two types of release:**
  - **Major release:** Deliver significant new functionality.
  - **Minor release:** Repair bugs and fix problems reported.
- **A release includes:**
  - Executable code
  - Configuration files defining how the release should be configured for particular installations
  - Data files (such as files of error messages) needed for successful system operation
  - Installation program used to help install system on hardware
  - Electronic and paper documentation describing the system
  - Packaging and associated publicity designed for that release

### AI Assistance in Release Management
- **Identify the release:** Assign release version and link it to the source commit, build, and required config.
- **Verify readiness:** Check AI-assisted changes meet requirements, have been reviewed, and have appropriate test evidence.
- **Document the release:** AI can draft release notes; verify the changes.
- **Plan recovery:** Keep previous working release and define how to restore service; check DB if changes affect rollback.

---

## Continuous Integration & CI/CD

- **Integration hell:** Situation where a dev spends more time reintegrating altered source code into the software system than they did creating the new feature.
- **Continuous Integration (CI):** Software development practice where members of a team integrate their work frequently (daily). Integration is verified by an automated build.

### SDLC with CI/CD
- **Idea:** CI/CD automates build, test, and release for faster, safer delivery. Failed tests go back to devs for fixing.
- **CI vs CD:**
  - **CI (Continuous Integration):** Integrate frequently and automatically build and test pushed changes.
  - **CD (Continuous Delivery):** Keep validated software ready for release, with a manual/explicit production release decision.
  - **Continuous Deployment:** Automatically deploy changes that pass the required checks.
- **Pipeline:** `Commit code to source control` $\rightarrow$ `CI server builds and runs tests` $\rightarrow$ `Report results to developer` $\rightarrow$ `Deploy to staging, then production`.

### CI/CD in AI
| Stage | Developer Action | AI Role |
| :--- | :--- | :--- |
| **Commit and push** | Record small meaningful changes | Draft code and commit descriptions |
| **Build** | Use declared dependencies and configuration | Explain errors and suggest fixes |
| **Test** | Run tests and check expected behavior | Suggest tests and explain failures |
| **Review and merge** | Inspect the diff, review evidence before approval | Summarize PR and highlight risks |
| **Release and deploy** | Identify the release, deploy and monitor | Draft release notes and help analyze errors |

### Four Steps to Continuous Integration
1. **Repository setup:** Keeps track of every change in your code base. If you generate something, don't store it in repo. Commit as much as possible.
2. **Building automation:** System building (compiling and linking system components, external libraries, config files).
3. **Testing automation**
4. **CI server setup:** Many products, same functionalities. Define trigger (e.g., check-in, 12 am everyday). Keep developers informed of build result.

*Continuous integration practices minimize the disruption caused by rapid, concurrent changes to software systems.*

---

## Software Maintenance

- **Definition:** The modification of a software product **after delivery** to correct faults, to improve performance, or other attributes.

### Maintenance Activities
| Type | Purpose / Examples |
| :--- | :--- |
| **Corrective** | Fixing logical errors, exceptions, missing functions |
| **Adaptive** | Adapting to new OS, new compiler |
| **Perfective** | Faster search, optimizing |
| **Preventive** | Exception handlers, backup routines — prevent problems before they occur |

---

## Refactoring

- **Definition:** A change made to the **internal structure** of software to make it easier to understand and cheaper to modify **without changing its observable behaviour**.
- Restructuring code in a series of small, **semantics-preserving** transformations in order to make the code easier to maintain and modify.
- **NOT the same as:**
  - Adding features
  - Debugging code
  - Rewriting code

### Process of Refactoring
1. Identify refactoring opportunity
2. Create test case
3. Plan refactoring
4. Apply refactoring
5. Run test $\rightarrow$ if **success**, commit; **else**, go back to plan

### Bad Smell Code Patterns & Refactoring Techniques
- **Cloning (Duplicate code):**
  - *Refactoring technique:* **Extract Method** — create new method that encapsulates duplicate code; replace code clones with method call.
- **Complex structures:**
  - *Long methods*
  - *Large classes:* Small classes can get bloated over time as program grows. Problems: responsibility confusion, class becomes non-realistic model.
    - *Refactoring technique:* **Extract Class** — create new class; move the fields and methods responsible for the relevant functionality into the new class.
- **Variables and parameters:**
  - *Long parameter list*
  - *Feature envy:* Indication of a method that was incorrectly placed in the wrong class (indicates abstraction fault).
    - *Refactoring technique:* **Move Method** — move the method with feature envy to the class containing most frequently used methods/data.
  - *Data clumps*
  - *Primitive obsession*
  - *Temporary field*
- **Control structures and polymorphism:**
  - *Switch statements:* Cases in a switch contain logic for different types of instances of the same class; in OOP, indicates a new subclass should be created. Same switch-case structure appears in many places.
    - *Refactoring technique:* **Extract Subclass and Extract Method** — create new subclass, extract method to move case block logic into methods on new subclass, use **polymorphism** to substitute switch block.
  - *Refused bequest*
  - *Alternative classes with different interfaces*

### Limits of Refactoring
- Changes can affect:
  - **Non-functional properties** (e.g., performance)
  - The **Application Programming Interface (API)** of a system
- Refactorings that alter APIs happen when:
  - A class member is moved / renamed
  - List of parameters to an operation is changed