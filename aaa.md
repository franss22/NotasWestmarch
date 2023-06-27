# EmptyTest

## Objective:
- Smell rule: The test has no runnable statements. Thus tehe test always shows up as passing.
- Code fix: Add a NotImplementedException at the end of the test to make sure it shows up as failing when runnning tests.

# Code 
## [EmptyTestAnalyzer](EmptyTest/EmptyTest/EmptyTestAnalyzer.cs):
This class is tasked with defining an analysis to be run on some piece of code (on this example, each method), which determines if a diagnostic must be reported, and then reports it accordingly.

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

The lambda function checks all method [symbols](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-semantics#symbols). First, it looks at the parent class of the symbol, and checks if the class has the `[TestClass]` attribute we found before in the compilation. If it doesnt, it skips over the method and stops analyzing it. It does the same to check if the method we're analyzing has the `[TestMethod]` attribute. If it does, it register a callback to `AnalyzeMethodBlockIOperation`, to check the operations of the method.
```cs
(ctx) =>
{
    var methodSymbol = (IMethodSymbol)ctx.Symbol;

    //Check if the container class is [TestClass], skip if it's not
    var containerClass = methodSymbol.ContainingSymbol;
    if (containerClass is null) { return; }
    if (!FindAttributeInSymbol(testClassAttr, containerClass)) { return; }

    //Check if method is [TestMethod], register mthod analysis if it is
    if (FindAttributeInSymbol(testMethodAttr, methodSymbol)) { ctx.RegisterOperationBlockAction(AnalyzeMethodBlockIOperation); }
}
```

### AnalyzeMethodBlockIOperation

This method analyzes the method's operations. We get the method's body, which must be a `OperationKind.Block` descended from the method. If the body block does not have any descendants, it means it has no statements (comments, whitespace, etc, which are called trivia, do not count as operations nor symbols). Thus we get the method symbol again, and we emit a diagnostic with the decriptor (`Rule`), the diagnostic's location (the method identifier), and the method's name for the diagnostic message formatting.

```cs
private static void AnalyzeMethodBlockIOperation(OperationBlockAnalysisContext context)
{
    foreach (var block in context.OperationBlocks)//we look for the method body
    {
        if (block.Kind != OperationKind.Block) { continue; }
        if (block.Descendants().Count() == 0)//if the method body has no operations, it is empty
        {
            var methodSymbol = context.OwningSymbol;
            var diagnostic = Diagnostic.Create(Rule, methodSymbol.Locations.First(), methodSymbol.Name);
            context.ReportDiagnostic(diagnostic);
        }
    }
}
```

## [EmptyTestCodeFixProvider](EmptyTest/EmptyTest.CodeFixes/EmptyTestCodeFixProvider.cs)
This class is tasked with giving a reformat solution to a reported diagnostic. Specifically, for Empty Tests we want to add raise a NotImplementedException at the end of the method definition.


## Initial Definitions
First, we define which diagnostics trigger this specific code fix: Some fixes could be applied to many similar diagnostics. In this case, we only care about Empty Test diagnostics.

```cs
public sealed override ImmutableArray<string> FixableDiagnosticIds
{
    get { return ImmutableArray.Create(EmptyTestAnalyzer.DiagnosticId); }
}
```

We also leave this method as is. It allows the codefix to be applied wuickly to many diagnostics (refactor all).
```cs
public sealed override FixAllProvider GetFixAllProvider()
{
    // See https://github.com/dotnet/roslyn/blob/main/docs/analyzers/FixAllProvider.md for more information on Fix All Providers
    return WellKnownFixAllProviders.BatchFixer;
}
```

## Registering

The `RegisterCodeFixesAsync` method is tasked with finding the position of the diagnostic and could also, in more complex cases, decide between multiple different fixes. Each code fix registered then shows up when checking the refactoring drop down menu on the editor.
```cs
public sealed override async Task RegisterCodeFixesAsync(CodeFixContext context)
{
    var root = await context.Document.GetSyntaxRootAsync(context.CancellationToken).ConfigureAwait(false);

    var diagnostic = context.Diagnostics.First();
    var diagnosticSpan = diagnostic.Location.SourceSpan;

    // Find the method declaration identified by the diagnostic.
    var methodDeclaration = root.FindToken(diagnosticSpan.Start).Parent.AncestorsAndSelf().OfType<MethodDeclarationSyntax>().First();

    // Register a code action that will invoke the fix.
    context.RegisterCodeFix(
        CodeAction.Create(
            title: CodeFixResources.CodeFixTitle,
            createChangedDocument: c => AddNotImplementedException(context.Document, methodDeclaration, c),
            equivalenceKey: nameof(CodeFixResources.CodeFixTitle)),
        diagnostic);
}
```

## The Code Fix

The code fix must generate a "throw new NotImplementedException();" statement, anbd place it at the end of the method declaration, after every comment in the block. We should also be careful to include "using System;" at the top of the file if it's needed, or the exception will throw out import errors.

The full fix is as follow:

```cs
private async Task<Document> AddNotImplementedException(Document document, MethodDeclarationSyntax methodDeclaration, CancellationToken cancellationToken)
{
    //NotImplementedException class
    var semanticModel = await document.GetSemanticModelAsync(cancellationToken);
    var notImplementedExceptionType = semanticModel.Compilation.GetTypeByMetadataName(SystemNotImplementedExceptionTypeName);

    //creating the new body with the added "raise new NotImplementedException();" at the end.
    //Method statements
    var bodyBlockSyntax = methodDeclaration.Body;
    var bodyStatements = bodyBlockSyntax.Statements;
    var endBrace = bodyBlockSyntax.CloseBraceToken;

    //We generate "raise new NotImplementedException();"
    var generator = SyntaxGenerator.GetGenerator(document);
    var throwStatement = (StatementSyntax)generator.ThrowStatement(generator.ObjectCreationExpression(
        generator.TypeExpression(notImplementedExceptionType))).WithLeadingTrivia(endBrace.LeadingTrivia).WithAdditionalAnnotations(Simplifier.AddImportsAnnotation, Formatter.Annotation);

    //We add to the start of the statement block
    var newBlockStatements = bodyStatements.Insert(0, throwStatement);
    var newBodyBlockSyntax = bodyBlockSyntax.WithCloseBraceToken(endBrace.WithLeadingTrivia()).WithStatements(newBlockStatements);

    //Editing the document
    var root = await document.GetSyntaxRootAsync(cancellationToken);
    var newDocument = document.WithSyntaxRoot(root.ReplaceNode(bodyBlockSyntax, newBodyBlockSyntax));

    return newDocument;
}
```

Let's dissect it step by step:

Let's see a simple example smelly program to explain how it is fixed:
```cs
using Microsoft.VisualStudio.TestTools.UnitTesting;

namespace Corpus
{
    [TestClass]
    public class UnitTest
    {
        [TestMethod]
        public void TestMethod()
        {
            //Comment
        }
    }
}
```

After the codefix, it should like like this:

```cs
using Microsoft.VisualStudio.TestTools.UnitTesting;
using System;
namespace Corpus
{
    [TestClass]
    public class UnitTest
    {
        [TestMethod]
        public void TestMethod()
        {
            //Comment
            throw new NotImplementedException();
        }
    }
}
```

First, we get the class object for NotImplementedException. We'll use this later to generate the throw statement.
```cs
//NotImplementedException class
var semanticModel = await document.GetSemanticModelAsync(cancellationToken);
var notImplementedExceptionType = semanticModel.Compilation.GetTypeByMetadataName(SystemNotImplementedExceptionTypeName);
```

We then get the method's list of statements (which is empty) to be able to add the new statement at the end of it. We also get the method's ending brace ("}"). This is because every comment, whitespace, line end, etc in the method's body (except those between the opening brace and the first line ending) are stored as the ending brace's leading trivia. We'll need both the trivia and the brace later.

```cs
var bodyBlockSyntax = methodDeclaration.Body;
var bodyStatements = bodyBlockSyntax.Statements;
var endBrace = bodyBlockSyntax.CloseBraceToken;
```

To illustrate, the closing brace's leading trivia is the whitespace and comments between the ||. 
```cs
        {
|        //comment
        |}
```

We then generate our throw statement. We create generator for the document, then generate our statement.
`ThrowStatement(generator.ObjectCreationExpression(generator.TypeExpression(notImplementedExceptionType)))` becomes "throw new NotImplementedException();".
Then `.WithLeadingTrivia(endBrace.LeadingTrivia)` adds the leading trivia from the closing bracket to the throw statement, so it now includes the whitespace and comments before it.
Finally `.WithAdditionalAnnotations(Simplifier.AddImportsAnnotation, Formatter.Annotation);` makes sure the document has the necessary assemblies for the `NotImplementedException` class and adds it if it doesn't already exist with `Simplifier.AddImportsAnnotation`, and tidies up the formatting with ` Formatter.Annotation`.
```cs
var generator = SyntaxGenerator.GetGenerator(document);
var throwStatement = (StatementSyntax)generator
                        .ThrowStatement(generator.ObjectCreationExpression(generator.TypeExpression(notImplementedExceptionType)))
                        .WithLeadingTrivia(endBrace.LeadingTrivia)
                        .WithAdditionalAnnotations(Simplifier.AddImportsAnnotation, Formatter.Annotation);
```

After that, we make a new block for the method. Roslyn syntax trees are immutable, so every time we want to make a change we make copies of it. We make a copy of the original block, with some changes to it's original clsoing brace, and changing it's list of statements. Note that we use `.WithCloseBraceToken(endBrace.WithLeadingTrivia())` without passing any arguments to `WithLeadingTrivia()`. This means we substitute the block's closing brace with a copy of the original brace, which has no leading trivia. If we didnt do this, the leading trivia would be duplicated (since we copied it into the throw statement).
```
var newBlockStatements = bodyStatements.Insert(0, throwStatement);
var newBodyBlockSyntax = bodyBlockSyntax.WithCloseBraceToken(endBrace.WithLeadingTrivia()).WithStatements(newBlockStatements);
```


Finally, we make a copy of the whole syntax tree, substituting the original method block with our new fixed one, and return.

```cs
//Editing the document
var root = await document.GetSyntaxRootAsync(cancellationToken);
var newDocument = document.WithSyntaxRoot(root.ReplaceNode(bodyBlockSyntax, newBodyBlockSyntax));

return newDocument;
```


## [Tests](EmptyTest/EmptyTest.Test/EmptyTestUnitTests.cs)

We have a way to build unit tests for our analyzer and code fixer. It has many options, but for now we'll check those that are used in this project.

First we must declare a tester for our sepcific analyzer and fixer.
```cs
using VerifyCS = EmptyTest.Test.CSharpCodeFixVerifier<
    EmptyTest.EmptyTestAnalyzer,
    EmptyTest.EmptyTestCodeFixProvider>;
```

We also define an assembly, so that we can add specific dependencies into the tests.
```cs
private readonly ReferenceAssemblies UnitTestingAssembly = ReferenceAssemblies.NetFramework.Net48.Default.AddPackages(ImmutableArray.Create(new PackageIdentity("Microsoft.VisualStudio.UnitTesting", "11.0.50727.1"))).AddAssemblies(ImmutableArray.Create("Microsoft.VisualStudio.UnitTesting"));
```

Then we can start defining test methods.
The general idea of a test is as follows:
- Define a string with the code we want to analyze
- Define the diagnostic (or lack thereof) we expect to get. We must specify which the expected diagnostic id, the position (the start and ending line and character of the code that will be highlighted when shown on the editor) and any aditional parameters the formatted message string would have (in our empty Test analyzer, that would be the moethod's name.)
- (Optionally) Define a string for the code we expect to get after running the code fixer
- Pass all of that, and the extra assemblies, into `VeryfyCS.Test`.

Normally, the code string is defined on the method itself. However, to make it easier to visualize, we built and use `TestReader`, which let's us import code from a corpus folder in the same same directory as the test class as strings. That way, we can make a `testcode.cs` file for each test case and then import it using it's name only.

A typical test looks like this:
```cs
[TestMethod]
public async Task EmptyTestWithCommentFixed()
{
    var testFile = @"Corpus\EmptyTestWithComments.cs";
    var fixedFile = @"Corpus\EmptyTestWithCommentsFixed.cs";

    var expected = VerifyCS.Diagnostic("EmptyTest").WithSpan(9, 21, 9, 32).WithArguments("TestMethod1");
    await new VerifyCS.Test
    {
        TestCode = testReader.ReadTest(testFile),
        FixedCode = testReader.ReadTest(fixedFile),
        ExpectedDiagnostics = { expected },
        ReferenceAssemblies = UnitTestingAssembly
    }.RunAsync();
}
```
A final note on tests: since the files in the corpus fodler are `.cs` files, they are picked up by intellisense, grammar highlighted, etc (which is the point). However, since we are testing test classes, this can interfere with running the actual tests. For example, making 2 files with the class and same method names (for example, for a file with a smell, and then a file with the smell fixed) will pop up a build error (since we are defining the same method twice). Each testing class and method also shows up in the test explorer (and generally fails), which isn't ideal.

To (not very elegantly) fix this, I excluded the corpus file from the Visual Studio Project. That way it doesn't get checked for error nor appears in the test explorer. It also doesnt show up in the solution explorer, which isn't the best either, but it can be opened in another editor (VSCode in my case) as a workaround.

