# EmptyTest

## Objective:
- Smell rule: The test has no runnable statements. Thus tehe test always shows up as passing.
- Code fix: Add a NotImplementedException at the end of the test to make sure it shows up as failing when runnning tests.

# Code 
## [EmptyTestAnalyzer](EmptyTest/EmptyTest/EmptyTestAnalyzer.cs):

### Definitions and Rule
The class must inherit from `DiagnosticAnalyzer`. We also specifiy that it's an analyzer for C# (since Roslyn also supports VB analyzers).
```
[DiagnosticAnalyzer(LanguageNames.CSharp)]
    public class EmptyTestAnalyzer : DiagnosticAnalyzer
```

The first lines of the class define strings specific to the smell diagnostic:

```cs
public const string DiagnosticId = "EmptyTest";

//Defining localized names and info for the diagnostic
private static readonly LocalizableString Title = new LocalizableResourceString(nameof(Resources.AnalyzerTitle), Resources.ResourceManager, typeof(Resources));
private static readonly LocalizableString MessageFormat = new LocalizableResourceString(nameof(Resources.AnalyzerMessageFormat), Resources.ResourceManager, typeof(Resources));
private static readonly LocalizableString Description = new LocalizableResourceString(nameof(Resources.AnalyzerDescription), Resources.ResourceManager, typeof(Resources));
private const string Category = "Test Smells";

private static readonly DiagnosticDescriptor Rule = new DiagnosticDescriptor(DiagnosticId, Title, MessageFormat, Category, DiagnosticSeverity.Warning, isEnabledByDefault: true, description: Description);

public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics { get { return ImmutableArray.Create(Rule); } }

```
`Title`, `MessageFormat` and `Description`'s values are read from the [Resources](EmptyTest/EmptyTest/Resources.resx) file. We then create a `DiagnosticDescriptor` out of these values. This descriptor gets passed later to the emitted diagnostic, alongside the location of the smell and any strings needed for formatting `MessageFormat` (in this case, "Test Method '{0}' is empty", which expects the name of the smelly method).

### Initialize

In the initialize method, we set some configuration options (no analyzing of generated code and enabling concurrent execution) and then register our first callback, `FindTestingClass`, to go over the [compilation](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-semantics#compilation) of the program, which houses all of the available classes and assemblies.

```cs
public override void Initialize(AnalysisContext context)
{
    // Controls analysis of generated code (ex. EntityFramework Migration) None means generated code is not analyzed
    context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);

    context.EnableConcurrentExecution();

    //Registers callback to start analysis
    context.RegisterCompilationStartAction(FindTestingClass);
}
```

### FindTestingClass

This method is tasked with finding the `TestClass` and `TestMethod` classes in the compilation. Then, it registers a callback for a lambda functions over method symbols, which actually does the searching for test classes, pasing the found classes to it.

(the lambda function is omitted here to explain it in detail later)
```cs
private static void FindTestingClass(CompilationStartAnalysisContext context)
        {

            // Get the attribute object from the compilation
            var testClassAttr = context.Compilation.GetTypeByMetadataName("Microsoft.VisualStudio.TestTools.UnitTesting.TestClassAttribute");
            if (testClassAttr is null) { return; }
            var testMethodAttr = context.Compilation.GetTypeByMetadataName("Microsoft.VisualStudio.TestTools.UnitTesting.TestMethodAttribute");
            if (testMethodAttr is null) { return; }

            

            // We register a Symbol Start Action to filter all test classes and their test methods
            context.RegisterSymbolStartAction(...lambda... , SymbolKind.Method);
        }

```

### lambda function

The lambda function checks all method [symbols](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-semantics#symbols). First, it looks at the parent class of the symbol, and checks if the class has the `[TestClass]` attribute we found before in the compilation. If it doesnt, it skips over the method and stops analyzing it. It does a similar thing
```cs
(ctx) =>
    {
        var methodSymbol = (IMethodSymbol)ctx.Symbol;


        //Check if the container class is [TestClass]
        var container = methodSymbol.ContainingSymbol;
        if (container is null) { return; }
        var isTestClass = false;
        foreach (var attr in container.GetAttributes())
        {
            if (attr.AttributeClass.Name.Equals("TestClassAttribute"))
            {
                isTestClass = true; break;
            }
        }
        if (!isTestClass) { return; }

        //Check if method is [TestMethod]
        foreach (var attr in methodSymbol.GetAttributes())
        {
            if (SymbolEqualityComparer.Default.Equals(attr.AttributeClass, testMethodAttr))
            {
                // If it's a test method in a test class, we check it internally to see if it has no statements
                ctx.RegisterOperationBlockAction(AnalyzeMethodBlockIOperation);
                break;
            }
        }
    }
```

