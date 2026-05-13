# Member C — Abstract Syntax Tree (AST) & Visitors

> This document is a deep dive into the role of **Member C** on the Boolean Rule Language team. Every term that is normally abbreviated is **spelled out the first time it appears** and then used in either form.
>
> Members A, B, D, and E have their own areas (scanning, parsing, semantics/runtime, engine/tooling). Member C sits **between the parser and the rest of the compiler**: their work is what the parser produces and what the type checker, interpreter, and printers consume.

---

## 1. The role in one paragraph

Member C owns the **Abstract Syntax Tree (AST)** — the in-memory tree of objects that represents a parsed program — and the **Visitor pattern** that lets other phases run algorithms over that tree. In compiler terms, Member C is in charge of the project’s **Intermediate Representation (IR)**: the canonical data structure that flows between the front-end (lexer + parser) and the back-end (type checker + interpreter + future passes).

---

## 2. Vocabulary you must be comfortable with

| Term (full form, then short form) | Meaning in this project |
|------------------------------------|--------------------------|
| **Abstract Syntax Tree (AST)** | A tree of plain Java objects (instances of `Node` subclasses) that represents the structural meaning of the program, without the surface details like whitespace, comments, or parentheses. |
| **Intermediate Representation (IR)** | The data structure that sits between the front-end (lexer/parser) and the back-end (analyzers/interpreters/code generators). Here, our IR **is** the AST. |
| **Visitor pattern** | A design pattern for adding new operations to a fixed set of object types without modifying those types. Each operation is implemented as a separate visitor class. |
| **Double dispatch** | A technique by which a method call is dispatched on the runtime types of **two** objects (the node and the visitor), not just one. Java only natively does single dispatch, so we implement double dispatch by hand using `Node.accept(visitor)`. |
| **Domain-Specific Language (DSL)** | A small programming language designed for a narrow problem area — in our case, boolean rules over numbers and booleans. |
| **Depth-First Search (DFS)** | A way to walk a tree where each child is fully visited before moving on to the next sibling. **Pre-order DFS** writes the parent before its children; **post-order DFS** writes the parent after them. |
| **JavaScript Object Notation (JSON)** | A text format used by `ASTJsonPrinter` to serialize the AST for debugging. |
| **Single Source of Truth** | A principle: when there is only one place in the codebase that defines a thing, every other component must read from that place. Member C is the single source of truth for AST node shapes and the visitor contract. |

---

## 3. What Member C owns in the source tree

```
src/main/java/com/booleanrulelang/
├── domain/
│   ├── Node.java                 ← abstract root of the AST
│   │
│   ├── ProgramNode.java          ← (jointly with Member B — see §8)
│   ├── AssignNode.java           ← (jointly with Member B)
│   ├── PrintNode.java            ← (jointly with Member B)
│   │
│   ├── ArithmeticOpNode.java     ← binary +, -, *, /
│   ├── ComparisonOpNode.java     ← binary <, >, <=, >=, ==, !=
│   ├── LogicalOpNode.java        ← binary and, or
│   ├── NotNode.java              ← unary not
│   ├── NegationNode.java         ← unary arithmetic minus
│   │
│   ├── NumberNode.java           ← numeric literal leaf
│   ├── BoolNode.java             ← boolean literal leaf
│   └── IdentifierNode.java       ← variable reference leaf
│
└── visitor/
    ├── ASTVisitor.java           ← the visitor contract
    ├── ASTJsonPrinter.java       ← serializes the AST as JSON
    └── ASTTraversalPrinter.java  ← prints the AST as indented pre-order text
```

The three statement nodes (`ProgramNode`, `AssignNode`, `PrintNode`) **straddle Member B and Member C**, because the *shape* of the statement is a parser decision, but the *class definition* is part of the AST. Treat these as “joint files”: Member B may modify them when they change the grammar; Member C reviews to make sure new fields fit the existing visitor pattern.

---

## 4. Background — why the Abstract Syntax Tree matters

A compiler typically goes through these phases:

1. **Lexical analysis** (scanner) — group characters into tokens.
2. **Syntactic analysis** (parser) — verify the token sequence respects the grammar.
3. **Build an Intermediate Representation** — usually the Abstract Syntax Tree.
4. **Semantic analysis** — typing rules, name resolution.
5. **Code generation or interpretation** — produce machine code, bytecode, or directly evaluate.

The AST is what makes phases 4 and 5 possible **independently of the original source text**. Once the tree is built, the type checker and interpreter never need to look at strings or tokens; they look at typed Java objects. This separation:

- Lets each phase reason in its own language (operators are Java classes, not characters).
- Lets us add new analyses (constant folding, dead-code detection) **without touching the parser**.
- Lets us swap the front-end (e.g. a parser for a different syntax) without rewriting the back-end.

Member C is therefore responsible for the **interface between halves of the compiler**. If this interface is well-designed, the team scales: A and B can iterate on the front-end while D iterates on the back-end.

---

## 5. Why the Visitor pattern is used here

There are roughly three ways to add a new operation over a tree of typed nodes:

1. **Add a method to every node class.** Works, but every new operation (typing, evaluation, pretty-printing, optimization) requires editing **every node class**. This concentrates change across many files.
2. **Pattern-match on the node type at the call site.** Works in Java with `instanceof`, but you lose the compiler’s help when a new node class is added — you have to remember to update every call site.
3. **Visitor pattern.** Each operation is a class that implements an interface listing **one method per node type**. Adding a new operation = adding one class. Adding a new node type = adding one method to the visitor interface, and Java forces every existing visitor to implement it.

We use option 3 because it gives us **compile-time safety when the set of node types changes**, and Member C is the gatekeeper for that interface.

### How the double dispatch works in this project

When `Interpreter` is given a generic `Node` reference, plain Java would dispatch based only on the static type (`Node`). We need the dispatch to choose by the actual runtime type (for instance `ArithmeticOpNode` vs `LogicalOpNode`). The trick is:

1. We call `node.accept(thisInterpreter)`.
2. Java dispatches `accept` on `node`’s runtime class — that picks the right `accept` body.
3. Inside `accept` we call `visitor.visitArithmeticOp(this)` (or whichever method matches that class).
4. Java now dispatches on the **visitor’s** runtime class — picking the interpreter’s, the printer’s, or the type checker’s implementation.

Two dispatches, one per object. That is the double dispatch.

---

## 6. The classes — what each one is and why it exists

### 6.1 `domain/Node.java` — abstract root of the AST

```text
public abstract class Node {
    public abstract <T> T accept(ASTVisitor<T> visitor);
}
```

- **What it is:** The abstract base class that every AST node extends.
- **Why it exists:** Lets us refer to any node with a single Java type and lets every node participate in the visitor pattern by implementing `accept`.
- **Member C’s responsibility:** Keep the `accept` contract simple, generic, and total — every concrete node must implement it by calling the matching `visit…` method.

### 6.2 Statement nodes (joint with Member B)

#### `ProgramNode.java`

- A list of statements: `List<Node> statements`. Conceptually the **root** of the parsed program.
- Member C makes sure the field stays `List<Node>` rather than a more specific type so that future statement kinds (`if`, `while`) can join without breaking visitors.

#### `AssignNode.java`

- Represents `name := value;`. Holds the variable name and the right-hand expression.
- Member C makes sure adding a new statement field (e.g. a source location) does not break the visitor’s `visitAssign` signature.

#### `PrintNode.java`

- Represents `print expression;`. Holds the expression to be evaluated.

### 6.3 Binary operator nodes — three **separate** classes on purpose

A central design decision in this project (see `docs/decision.md`) is that **arithmetic, comparison, and logical** operations must be **distinguishable** at the AST level. We did not merge them under one `BinaryOpNode` with an operator string because:

- The type checker’s rules differ per family (numbers in, numbers out vs booleans in, booleans out vs numbers in, boolean out).
- Future tooling (printers, optimizers, formal verifiers) can dispatch by Java type, which is faster and safer than string comparison.

#### `ArithmeticOpNode.java`

- Fields: `String op` (the textual operator: `"+"`, `"-"`, `"*"`, `"/"`), `Node left`, `Node right`.
- Built by the parser inside `parseArithmetic` and `parseTerm`.
- Consumed by `TypeChecker` (must be `NUMBER × NUMBER → NUMBER`) and `Interpreter` (does the actual arithmetic; division by zero throws `EvaluationException`).

#### `ComparisonOpNode.java`

- Fields: same shape as `ArithmeticOpNode`, but the operator is one of `<`, `>`, `<=`, `>=`, `==`, `!=`.
- Type rules differ between relational (`<` family — both must be numbers) and equality (`==`, `!=` — same type on both sides, returns boolean).

#### `LogicalOpNode.java`

- Fields: same shape, operator is `"and"` or `"or"`.
- Both operands must be boolean; result is boolean. Member D’s interpreter naturally short-circuits because Java’s `&&` and `||` do.

### 6.4 Unary operator nodes — split for the same reason

#### `NotNode.java`

- Fields: `Node operand`. No operator string — the class itself encodes the operator.
- Operand must be boolean; result boolean.

#### `NegationNode.java`

- Fields: `Node operand`. Represents the unary arithmetic minus (`-(1 + 2)`).
- Operand must be number; result number.

### 6.5 Leaf nodes

#### `NumberNode.java`

- Holds a single `double value`.
- Created by the parser when it sees a `NUMBER` token (via `Double.parseDouble`).
- Both printers special-case integer-valued doubles for cleaner output (`5` rather than `5.0`).

#### `BoolNode.java`

- Holds a single `boolean value`. Used for the literals `true` and `false`.

#### `IdentifierNode.java`

- Holds a single `String name`. Resolution happens later — the AST records *what name the user wrote*, and the type checker / interpreter look it up in their environments.

### 6.6 `visitor/ASTVisitor.java` — the visitor contract

```text
public interface ASTVisitor<T> {
    T visitProgram(ProgramNode node);
    T visitAssign(AssignNode node);
    T visitPrint(PrintNode node);

    T visitArithmeticOp(ArithmeticOpNode node);
    T visitComparisonOp(ComparisonOpNode node);
    T visitLogicalOp(LogicalOpNode node);

    T visitNot(NotNode node);
    T visitNegation(NegationNode node);

    T visitNumber(NumberNode node);
    T visitIdentifier(IdentifierNode node);
    T visitBool(BoolNode node);
}
```

- **Generic in `T`** so visitors can return any type (the interpreter returns `RuntimeValue`, the JSON printer returns `JSONObject`, the traversal printer returns `Void` because it only has side effects).
- Member C is the **only person who may change the method names or signatures here**. Anyone touching this file must coordinate with Members A (if a new token type is added that yields a new node), C (interface owner), and D (typing/eval rule for the new node).

### 6.7 `visitor/ASTJsonPrinter.java` — serializer

- Implements `ASTVisitor<JSONObject>`. Every visit method builds a `JSONObject` describing the node, then attaches its children’s objects under `left`, `right`, or `operand`.
- **Why it exists:** an inspectable, machine-readable dump of the AST, useful for debugging passes Member C might add later.
- Not on the default printout, but kept available so adding `--ast-json` to the engine (Member E) is one line.

### 6.8 `visitor/ASTTraversalPrinter.java` — pre-order pretty-printer

- Implements the **depth-first pre-order traversal** described in `docs/ASTTraversalPrinter-Explained.md`.
- Uses a private inner class (`PreOrderWalker`) so each call to `print(...)` gets fresh `StringBuilder` and `depth` fields and the public class stays a clean Spring bean.
- Produces the `== AST (pre-order traversal) ==` section of the engine’s output.

---

## 7. Day-to-day responsibilities

1. **Be the gatekeeper of `ASTVisitor`.** Any pull request that adds, renames, or removes a `visit…` method must be reviewed by Member C. If the change is good, Member C is also responsible for **updating every existing visitor** (`ASTJsonPrinter`, `ASTTraversalPrinter`, plus the visitors owned by Member D — `TypeChecker`, `Interpreter`) in the same pull request so the build never breaks on the main branch.
2. **Be the gatekeeper of `Node` and its subclasses.** Reviews to make sure new fields stay immutable where possible, that `accept` dispatches to the correct method, that no business logic creeps into node classes (the rule is: **nodes are data**, visitors do the work).
3. **Keep the operator-family split intact.** If anyone proposes “let’s just collapse `ArithmeticOpNode` and `ComparisonOpNode` again,” point them at the decision document.
4. **Write and maintain the AST documentation** in `docs/` — the project leans on these docs heavily during the discussion.
5. **Provide example trees** for hand-traced exercises during the discussion (the traversal pretty-printer makes this easy).

---

## 8. Coordination protocol with the other members

| Counterpart | Shared interface | What Member C does |
|--|--|--|
| **Member A — Scanner & Tokens** | If A adds a new token category (for instance for comments or a new operator), C decides whether that token results in a new AST node or fits an existing one. |
| **Member B — Parser & Grammar** | B is the main writer of `Parser.java`. C reviews any change that constructs nodes to ensure the right class is used (for instance, `or` must create a `LogicalOpNode`, not a generic binary node). |
| **Member D — Semantics & Runtime** | D writes the most “real” visitors (`TypeChecker`, `Interpreter`). When C changes the visitor interface, D’s code must be updated in the same pull request to keep `mvn test` green. |
| **Member E — Engine & Tooling** | E owns `CompilerEngine`. When C adds a new visitor (for instance a constant folder), E decides where to plug it into the pipeline. |

A typical pull request that adds a new statement kind (say `if`/`else`) touches the parser (B), introduces an `IfNode` (C), adds a `visitIf` method to `ASTVisitor` (C), implements the new method in all visitors (C and D in the same pull request), and is wired into the engine if a new printer section is needed (E). Member C is the natural lead for that pull request because they touch the most files.

---

## 9. Workflow — adding a new AST node from scratch

This is the most common task in Member C’s queue. Step by step:

1. **Justify the node.** Is the new construct really distinct, or can an existing node represent it (for example with a new operator string)? If the type rules or runtime behavior differ from any existing node, create a new class.
2. **Create the class under `domain/`** following the existing patterns (`@Component`, `@RequiredArgsConstructor`, `@AllArgsConstructor`, fields, and a single `accept` method).
3. **Add a `visitXxx(...)` method to `ASTVisitor`.** Pick a clear, unambiguous name. Do not abbreviate.
4. **Implement the new method in every existing visitor.** That includes `ASTJsonPrinter`, `ASTTraversalPrinter`, `TypeChecker`, and `Interpreter`. The compiler will refuse to build until all four are updated, which is the point.
5. **Update the parser** so it actually constructs the new node where appropriate.
6. **Add at least one passing and one failing test** under `examples/tests/pass/` and `examples/tests/fail/`.
7. **Document the change** in `docs/team-and-source-tour.md` and, if user-facing, in `docs/Boolean-Rule-Language-Guide.md`.

---

## 10. Workflow — adding a new visitor (a new pass over the AST)

For example, “constant folding” (collapsing `2 + 3` to `5` at compile time).

1. **Define what the visitor returns.** A folder might return a new `Node` (the folded subtree). A complexity analyzer might return an `int`. Pick `T` accordingly.
2. **Create the class under `visitor/`,** declaring `implements ASTVisitor<T>`. The Java compiler will list **every** method you must implement — there is no way to forget a node type.
3. **Implement each method.** For nodes that are unchanged, return a copy or the original; only the relevant nodes change.
4. **Decide where to plug it in.** This is a conversation with Member E. Often it goes between the type checker and the interpreter: `engine.compile` would now call `program = constantFolder.fold(program);` before handing to the interpreter.
5. **Write tests.** Folding tests are usually golden-file (input file → expected printed AST).
6. **Document the visitor’s purpose and contract** in `docs/`.

---

## 11. Conventions and standards Member C enforces

- **Nodes are data, visitors are behavior.** No `eval()` or `check()` methods on `Node` subclasses; the only logic in a node is the `accept` call.
- **Prefer immutability.** Lombok’s `@AllArgsConstructor` plus `public final` fields are fine; only mutate during construction.
- **Operator strings live on binary-operator nodes only.** Leaves (`NumberNode`, `BoolNode`, `IdentifierNode`) and named unary nodes (`NotNode`, `NegationNode`) do not need an operator field.
- **Visitor methods follow `visit<NodeShortName>`.** For instance `visitArithmeticOp`, not `visitArith` or `visitAOP`.
- **Generic visitors over result type `T` only.** Do not introduce additional generic parameters; if you need extra state, pass it through the visitor’s constructor (as `ASTTraversalPrinter` does with the `StringBuilder`).

---

## 12. Testing duties

While Member E will eventually own the automated test harness, Member C should still:

- Hand-trace one program per kind of node and verify it appears in both the JSON dump and the pre-order traversal correctly.
- Write a regression note when the visitor interface changes (in `docs/decision-checklist.md`).
- Provide example inputs for the discussion that show each node kind being created (`examples/tests/pass/07-scanner-coverage.txt` is already a good example — Member C may want a similar “AST coverage” example).

---

## 13. Deliverables Member C should present at the discussion

1. **A live demo** of `examples/tests/pass/06-assignment-and-use.txt` showing how the pre-order traversal mirrors the source program’s structure.
2. **A diagram or hand-drawn tree** for `(1 + 2) * 3 >= 9 and not false`, mapping every part of the expression to its node class. The visitor’s output for `examples/tests/pass/07-scanner-coverage.txt` is the printed form of exactly that tree.
3. **A short slide on why arithmetic, comparison, and logical operations are separate Java classes** — this answers the third bullet of `docs/decision.md` directly.
4. **A short slide explaining double dispatch** in plain language, using the snippet from §5 above.

---

## 14. Suggested feature roadmap for Member C (in order)

1. **Source locations on every AST node** — add a `Position` record `(line, column)` and thread it from the parser. Touches every node class but with a tightly-defined contract. Greatly improves diagnostics from the type checker.
2. **Pretty-printer visitor** — emits valid source code from the AST. Great for the discussion (round-trip demo: source → tokens → AST → source).
3. **Graphviz visitor** — emits the AST as a `.dot` file that renders to a real tree diagram.
4. **Constant folding visitor** — first real optimization pass over the AST.
5. **Post-order and level-order traversal printers** — pedagogical, but useful in slides.

Each of these is one new class under `visitor/` plus minor documentation; none of them require touching `Parser.java` or `Lexer.java`, which keeps cross-team conflicts low.

---

## 15. Summary

Member C is the **architect of the project’s internal data model**. Everything Member A produces (tokens) becomes something Member B turns into Member C’s structures (AST nodes), which Member D consumes (typing and execution) and which Member E displays (engine output sections). When the AST and the visitor contract are clean and stable, every other team member moves faster. When they drift, every other team member is blocked. That is why this role is, in effect, the structural core of the compiler.
