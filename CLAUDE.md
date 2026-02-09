# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Tai-e is a static analysis framework for Java that features powerful pointer analysis, configurable security analysis, control/data-flow analysis, and bug detection. It uses Soot as its bytecode frontend and provides an extensible plugin system for custom analyses.

**Key Paper:** Tian Tan and Yue Li. "Tai-e: A Developer-Friendly Static Analysis Framework for Java by Harnessing the Good Designs of Classics." ISSTA 2023.

## Build Commands

### Building the Project
```bash
# Build runnable fat JAR (includes all dependencies)
./gradlew fatJar
# Output: build/tai-e-all-0.5.4-SNAPSHOT.jar

# Build regular JAR
./gradlew jar
# Output: build/tai-e-0.5.4-SNAPSHOT.jar

# Clean build
./gradlew clean build
```

### Running Tests
```bash
# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests "pascal.taie.analysis.pta.PointerAnalysisTest"

# Run tests with specific pattern
./gradlew test --tests "*PointerAnalysis*"

# Run tests in parallel (configured to use half CPU cores)
# Already configured in build.gradle.kts
```

### Code Quality
```bash
# Run checkstyle
./gradlew checkstyleMain checkstyleTest

# Generate Javadocs
./gradlew javadoc
# Output: build/docs/javadoc/
```

### Running Tai-e
```bash
# Basic usage with fat JAR
java -jar build/tai-e-all-0.5.4-SNAPSHOT.jar \
  -cp <classpath> \
  -m <main-class> \
  -a "pta=cs:2-type"

# With custom JDK for Java 9+ (use correct JDK version to match -java flag)
/usr/lib/jvm/java-17-openjdk-amd64/bin/java -jar build/tai-e-all-0.5.4-SNAPSHOT.jar \
  -cp <classpath> \
  -m <main-class> \
  -java 17 \
  --soot-class-path /usr/lib/jvm/java-17-openjdk-amd64/jmods \
  -a "pta=cs:2-type"

# Generate call graph
-a "pta=cs:2-type" -a "cg=dump:true;dump-methods:true;dump-call-edges:true"
```

## Architecture Overview

### Execution Flow

1. **Main Entry** (`pascal.taie.Main`):
   - Parses command-line options via `Options.parse()`
   - Loads `tai-e-analyses.yml` containing analysis definitions
   - Uses `AnalysisPlanner` to resolve analysis dependencies and create execution plan
   - Calls `buildWorld()` to construct program representation
   - `AnalysisManager` executes analyses in dependency order

2. **World Building** (`pascal.taie.WorldBuilder`):
   - Default implementation: `SootWorldBuilder` (uses Soot for bytecode parsing)
   - Constructs `ClassHierarchy`, `TypeSystem`, and `IRBuilder`
   - Handles Java library selection (JRE 6, 8, 9+)
   - Sets `World.get()` as global singleton for program-wide data

3. **Analysis Execution** (`pascal.taie.analysis.AnalysisManager`):
   - Instantiates analyses via reflection (requires `AnalysisConfig` constructor)
   - Dispatches to appropriate executor based on type:
     - `ProgramAnalysis<R>`: Whole-program analysis
     - `ClassAnalysis<R>`: Per-class analysis
     - `MethodAnalysis<R>`: Per-method (intra-procedural) analysis
   - Stores results in `World` or entity-specific holders
   - Clears unused results based on dependency graph

### Core Components

#### World (`pascal.taie.World`)
Global singleton holding:
- `TypeSystem`: Type information and queries
- `ClassHierarchy`: Class hierarchy and method resolution
- `IRBuilder`: IR construction for methods
- `NativeModel`: Native method behavior modeling
- `JMethod mainMethod`: Program entry point
- Analysis results storage via `AbstractResultHolder`

#### Configuration System
- **Options.java**: Command-line parsing (uses picocli)
  - `-cp, --class-path`: Application classpath
  - `-acp, --app-class-path`: Source code location
  - `-m, --main-class`: Entry point
  - `--soot-class-path`: Custom JDK libraries (supports Java 9+ jmods via VIRTUAL_FS_FOR_JDK)
  - `-a, --analysis`: Analyses to run with options (format: `id=option:value`)

- **tai-e-analyses.yml**: Analysis definitions
  - Each analysis has: id, analysisClass, description, requires, options
  - Dependencies resolved via `AnalysisPlanner` with topological sort
  - Conditional requires: `requires: [A(x=y)]` only requires A if option x=y

#### Language Model
- **JClass** (`pascal.taie.language.classes.JClass`): Represents Java classes
- **JMethod** (`pascal.taie.language.classes.JMethod`): Represents methods, provides `getIR()`
- **JField** (`pascal.taie.language.classes.JField`): Represents fields
- **TypeSystem** (`pascal.taie.language.type.TypeSystem`): Type queries and management
- **ClassHierarchy** (`pascal.taie.language.classes.ClassHierarchy`): Superclass/interface relationships, method resolution

#### Intermediate Representation (IR)
- **IR** (`pascal.taie.ir.IR`): Per-method representation
  - Contains variables (this, parameters, locals)
  - Contains statements (instructions)
  - Exception entries (try-catch blocks)
  - Accessed via `JMethod.getIR()`

- **Statements** (`pascal.taie.ir.stmt`): Assignment, Invoke, New, Return, If, Goto, Switch, etc.
- **Expressions** (`pascal.taie.ir.exp`): Var, Constant, FieldAccess, ArrayAccess, etc.

### Pointer Analysis (PTA)

The centerpiece of Tai-e's analysis capabilities.

#### Architecture (`pascal.taie.analysis.pta`)

**PointerAnalysis** orchestrates:
1. Creates `HeapModel` (e.g., `AllocationSiteBasedModel`)
2. Selects `ContextSelector` for context sensitivity via `ContextSelectorFactory`
3. Instantiates `Solver` with heap model and context selector
4. Registers plugins (reflection, exception, lambdas, taint, etc.)
5. Calls `solver.solve()` to run analysis

**Solver** (`pascal.taie.analysis.pta.core.solver.Solver`):
- Work-list based fixed-point computation
- Manages pointer flow graph (PFG)
- Builds call graph on-the-fly
- Provides queries: `getPointsToSetOf()`, `getCallGraph()`
- Plugin integration via event callbacks

#### Context Sensitivity (`cs` option)
- `ci`: Context-insensitive (0-CFA)
- `k-obj`: k-limiting object-sensitive (k=1,2,3...)
- `k-type`: k-limiting type-sensitive
- `k-call`: k-limiting call-site-sensitive
- `-k'h`: Specify heap context limit (e.g., `2-obj-1h`)

Advanced techniques:
- `zipper`: Selective context sensitivity (OOPSLA'18)
- `scaler`: Scalable context sensitivity (FSE'18)
- `mahjong`: Customizable heap abstraction (PLDI'17)

#### Plugin System (`pascal.taie.analysis.pta.plugin.Plugin`)

Plugins receive callbacks during PTA:
- `onStart()`: Before analysis begins
- `onNewPointsToSet(CSVar, PointsToSet)`: When points-to set grows
- `onNewCallEdge(Edge)`: When new call edge discovered
- `onNewMethod(JMethod)`: When method becomes reachable
- `onNewStmt(Stmt, JMethod)`: When statement becomes reachable
- `onNewCSMethod(CSMethod)`: When context-sensitive method added
- `onUnresolvedCall(CSObj, Context, Invoke)`: For reflection/dynamic dispatch
- `onFinish()`: After analysis completes

**Built-in Plugins:**
- `EntryPointHandler`: Adds entry methods
- `ClassInitializer`: Class initialization modeling
- `ThreadHandler`: Thread behavior
- `NativeModeller`: Native method modeling
- `ExceptionAnalysis`: Exception flow
- `LambdaAnalysis`: Lambda expressions (Java 8+)
- `InvokeDynamicAnalysis`: invokedynamic (Java 7+)
- `ReflectionAnalysis`: Reflection resolution (string-constant or SOLAR)
- `TaintAnalysis`: Taint tracking for security analysis

**Custom Plugins:**
Load via: `-a pta=plugins:[my.package.MyPlugin]`

Must implement `Plugin` interface and have public constructor taking `Solver`.

### Call Graph Construction

**CallGraphBuilder** (`pascal.taie.analysis.graph.callgraph.CallGraphBuilder`):
- Requires PTA or CHA
- Algorithms (`algorithm` option):
  - `pta`: Precision-driven, built during pointer analysis
  - `cha`: Class Hierarchy Analysis (conservative, fast)
  - `cha=LIMIT`: CHA with depth limit

- **Dump options:**
  - `dump:true`: Generate DOT file (visualize with Graphviz)
  - `dump-methods:true`: List reachable methods
  - `dump-call-edges:true`: List all call edges

### Control and Data Flow Analysis

**CFG** (`pascal.taie.analysis.graph.cfg.CFGBuilder`):
- Per-method control-flow graph
- Includes exception flow (requires throw analysis)
- Option: `dump:true` to output CFG

**ICFG** (`pascal.taie.analysis.graph.icfg.ICFGBuilder`):
- Inter-procedural CFG
- Combines CFG with call graph
- Requires both CFG and call graph analyses

**Data-Flow Analyses** (`pascal.taie.analysis.dataflow`):
- Framework: `AbstractDataflowAnalysis`, `WorkListSolver`
- Built-in: `LiveVariable`, `ConstantPropagation`, `ReachingDefinition`, `AvailableExpression`
- Custom analyses extend `AbstractDataflowAnalysis`

### Soot Frontend Integration

**SootWorldBuilder** (`pascal.taie.frontend.soot.SootWorldBuilder`):
- Initializes Soot `Scene` with program classpath
- Converts Soot classes to Tai-e `JClass` via `SootClassLoader`
- **MethodIRBuilder** converts Soot bytecode to Tai-e IR
- Handles Java version differences (6, 8, 9+)

**Key Implementation Details:**
- When `--soot-class-path` contains "jmods", automatically uses `VIRTUAL_FS_FOR_JDK` for Java 9+
- Must run with matching JDK version (e.g., Java 17 JVM for `-java 17`)
- Soot phases: Resolution → Transformation → Extraction

## Key Packages

| Package | Purpose |
|---------|---------|
| `pascal.taie` | Main entry, World management |
| `pascal.taie.config` | Configuration parsing (Options, AnalysisConfig) |
| `pascal.taie.analysis` | Analysis framework base classes |
| `pascal.taie.analysis.pta` | Pointer analysis orchestration |
| `pascal.taie.analysis.pta.core` | PTA solver, context handling |
| `pascal.taie.analysis.pta.core.heap` | Heap abstraction models |
| `pascal.taie.analysis.pta.plugin` | Plugin interface and implementations |
| `pascal.taie.analysis.graph` | Call graph, CFG, ICFG |
| `pascal.taie.analysis.dataflow` | Data-flow analysis framework |
| `pascal.taie.ir` | Intermediate representation |
| `pascal.taie.language` | Language model (types, classes, methods) |
| `pascal.taie.frontend.soot` | Soot bytecode frontend |

## Development Patterns

### Creating a New Analysis

1. **Extend appropriate base class:**
   ```java
   public class MyAnalysis extends ProgramAnalysis<MyResult> {
       public static final String ID = "my-analysis";

       public MyAnalysis(AnalysisConfig config) {
           super(config);
       }

       @Override
       public MyResult analyze() {
           // Implementation
       }
   }
   ```

2. **Register in tai-e-analyses.yml:**
   ```yaml
   - id: my-analysis
     analysisClass: my.package.MyAnalysis
     description: "My custom analysis"
     requires: [pta]  # if depends on PTA
     options:
       my-option: default-value
   ```

3. **Run with:** `-a "my-analysis=my-option:value"`

### Creating a PTA Plugin

```java
public class MyPlugin implements Plugin {
    private Solver solver;

    @Override
    public void setSolver(Solver solver) {
        this.solver = solver;
    }

    @Override
    public void onNewPointsToSet(CSVar csVar, PointsToSet pts) {
        // React to points-to set changes
    }

    @Override
    public void onNewCallEdge(Edge<CSCallSite, CSMethod> edge) {
        // React to new call edges
    }

    // Implement other callbacks as needed
}
```

Load via: `-a pta=plugins:[my.package.MyPlugin]`

### Accessing Analysis Results

```java
// From World
MyResult result = World.get().getResult(MyAnalysis.ID);

// From JClass
ClassResult result = jclass.getResult(MyClassAnalysis.ID);

// From IR/JMethod
MethodResult result = ir.getResult(MyMethodAnalysis.ID);
```

## Testing

Tests are located in `src/test/java/` mirroring the main source structure.

**Test Structure:**
- Most tests use JUnit 5 (`@Test`, `@ParameterizedTest`)
- Test inputs in `src/test/resources/`
- PTA tests use `PTATestBase` with expected results files

**Running specific tests:**
```bash
# Single test class
./gradlew test --tests "pascal.taie.analysis.pta.PointerAnalysisTest"

# Test method
./gradlew test --tests "pascal.taie.analysis.pta.PointerAnalysisTest.testSimple"

# Pattern matching
./gradlew test --tests "*Pointer*"
```

## Important Files

- **tai-e-analyses.yml**: Analysis registry and configuration
- **Options.java**: Command-line option definitions
- **World.java**: Global program state
- **PointerAnalysis.java**: PTA orchestration
- **Solver.java**: PTA fixed-point computation
- **CallGraphBuilder.java**: Call graph construction
- **SootWorldBuilder.java**: Bytecode frontend integration

## Common Modifications

### Adding a new command-line option:
1. Add field to `Options.java` with `@Option` annotation
2. Add getter method
3. Option automatically available via picocli

### Modifying analysis dependencies:
Edit the `requires` field in `tai-e-analyses.yml`

### Changing context sensitivity:
Use `-a "pta=cs:<variant>"` where variant is `ci`, `k-obj`, `k-type`, `k-call`, etc.

### Custom heap abstraction:
Implement `HeapModel` interface and register in `PointerAnalysis`

## Performance Considerations

- PTA time limit: `-a "pta=time-limit:60"` (seconds)
- World caching: `-wc` (enables world cache mode)
- Pre-build IR: `--pre-build-ir` (builds IR for all methods upfront)
- Scope control: `-scope APP` (analyze only application code)
- Only keep needed results: `-kr <analysis-id>` (keeps only specified results)

## Documentation

- **Reference Docs**: https://tai-e.pascal-lab.net/docs/current/reference/en/index.html
- **API Docs**: https://tai-e.pascal-lab.net/docs/current/api/index.html
- **Local Docs**: `docs/en/` (AsciiDoc format)
- **Educational Version**: https://tai-e.pascal-lab.net/en/intro/overview.html (8 assignments)
