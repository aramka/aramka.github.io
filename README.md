# Minimal .NET Interceptor Source Generator
After looking at a few articles about .NET Interceptors I got a minimal project working. Quick and dirty.<br>
[The code is here](https://github.com/aramka/MinimalDotNetInterceptorSourceGenerator)
## Key Bits
Its an experimental feature as of 2026-04-25. See [here](https://github.com/dotnet/roslyn/blob/main/docs/features/interceptors.md)<br>
The csproj, the one that has Program.cs, that will make use of the generator must:

1. Target .NET 8 or better
2. Set the `<InterceptorsNamespaces>MinimalInterceptorSourceGenerator</InterceptorsNamespaces>`. This is the namespace where the interceptor generator will emit its code.
3. Make ref to the generator, either as a package or in my case the csproj that contains the generator and set ``ReferenceOutputAssembly="false" OutputItemType="Analyzer" `` . See below


<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net8.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
        <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
        <CompilerGeneratedFilesOutputPath>.GeneratedFiles</CompilerGeneratedFilesOutputPath>
        <InterceptorsNamespaces>MinimalInterceptorSourceGenerator</InterceptorsNamespaces>
    </PropertyGroup>

    <ItemGroup>
        <ProjectReference Include="..\MinimalInterceptorSourceGenerator\MinimalInterceptorSourceGenerator.csproj" ReferenceOutputAssembly="false" OutputItemType="Analyzer" />
    </ItemGroup>
    </Project>


The generator csproj must target .net standard and there are other bits...Well, I know at the very least you must include MS CodeAnalysis bits, this is how MS keeps source generators inline. You cannot use the entirety of .NET inside an [``IIncrementalGenerator``](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.iincrementalgenerator?view=roslyn-dotnet-5.0.0) and the MS analyzers requirements enforce this. AS for the other bits in the generator's csproj... well toggle them and see ;)

In the generator [``public void Initialize(IncrementalGeneratorInitializationContext context``)](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.iincrementalgenerator.initialize?view=roslyn-dotnet-5.0.0) I target particular code parts via [context.SyntaxProvider.CreateSyntaxProvider](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.syntaxvalueprovider.createsyntaxprovider?view=roslyn-dotnet-5.0.0) function by passing a prediate function, Func<[SyntaxNode](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.syntaxnode?view=roslyn-dotnet-5.0.0),CancellationToken> and a Func<[GeneratorSyntaxContext](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.generatorsyntaxcontext?view=roslyn-dotnet-5.0.0), CancellationToken, T> transform function. 

There are syntax node types for every single piece of code in C#. In the case of generating an interceptor I need to find the syntaxnode that represents the location of the invocation, function call, I want to intercept. [InvocationExpressionSyntax](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntax.invocationexpressionsyntax?view=roslyn-dotnet-5.0.0) is the syntaxnode type and I must also check if that invocation is of whatever the name is of the method or function Im targetting, in my case MyMethod.

In the transform function I return [InterceptableLocation](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.interceptablelocation?view=roslyn-dotnet-5.0.0) which I what I will use to generate the interceptor.

This part is funny to me and seems akward. The return value from [context.SyntaxProvider.CreateSyntaxProvider](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.syntaxvalueprovider.createsyntaxprovider?view=roslyn-dotnet-5.0.0) call above gets passed to another method on the context, [context.RegisterSourceOutput](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.incrementalgeneratorinitializationcontext.registersourceoutput?view=roslyn-dotnet-5.0.0) where I pass a third function that will be passed these [InterceptableLocation s](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.interceptablelocation?view=roslyn-dotnet-5.0.0)  that match the predicate and where Im allowed to generate the source.

Seems like a clumsy programming model. I feel like in the transform function that I passed, or maybe just call it the generate code function, I should be able to generate the code from there. Or maybe, [``IIncrementalGenerator``](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.iincrementalgenerator?view=roslyn-dotnet-5.0.0) itself should have three functions each targeting these concerns. But then again, I dont have to keep up with compiler code. But then again, compilers have end users too and if Im offering compiler features then my end users must be able to use them easily.

Hmm, I oft wonder if the things I build are good because they work, but then once in a while I get to talk with app support directly where I find that end users are doing all sorts of things to work around my shortcomings.

I digress..

Now, finally in the function I passed to [context.RegisterSourceOutput](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.incrementalgeneratorinitializationcontext.registersourceoutput?view=roslyn-dotnet-5.0.0) I get a [SourceProductionContext](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.sourceproductioncontext?view=roslyn-dotnet-5.0.0) and I can add some code via its [AddSource](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.sourceproductioncontext.addsource?view=roslyn-dotnet-5.0.0) function. Also, note that I get passed in my locations that I targeted in the above predicate function.

I must add the magic attribute

        sealed file class InterceptsLocationAttribute : global::System.Attribute
        {
            public InterceptsLocationAttribute(int version, string data)
            {
                _ = version;
                _ = data;
            }
        }

Some kind of magic, easter egg, the cs compiler will look for I suppose. Note, that I cannot find this attribute in the docs or compiler code base. Im not sure how the compiler looks for it. But certainly it must look for it somehow.

Then I must add the extension method class that will hold my interceptor extension method. The signature of this extension must match that of the method that im intercepting with the exception of the first parameter, this MyClass mc, that is required for extension methods.

Also, the interceptor extension method must be attributed with the magic attribute easter egg from above and pass in the [InterceptableLocation.Data](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.interceptablelocation.data?view=roslyn-dotnet-5.0.0) and the [InterceptableLocation.Version](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.interceptablelocation.version?view=roslyn-dotnet-5.0.0).

Interceptor done. Run the app and see it work.

The fact that the intercetpor is an extension method says, to me at least, there is no support for intercepting static or private and likely many other calls. At least I couldnt figure it.

Also, since many .NET things are not available for use in the interceptor debugging was hard for me. I know people are building unit tests around these and there is support for debugging source generators in unit tests. Hmmm, maybe Ill give it a go next. But, for me to see into the generator code I created a separate project that would log things to file. It worked too. But I had to shake it sometimes as it would intermittently not generate the log or jam the genrator somehow and I would see the some errors in the compile output. Still, it was helpful, in particular in seeing the SyntaxNodes I wanted to target. It turns out that ToString on these will emit the code they encapsulate. Handy.

See the code. I hope it helps. Message om github and I answer to the best of my avail.



