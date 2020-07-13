## 翻訳元：[sebastienros/jint](https://github.com/sebastienros/jint)


# Jint
Jint は __ECMA 5.1__ に完全に準拠し、あらゆる .NET プラットフォームで動作する .NET 用の __JavaScript インタプリタ__ です。.NET バイトコードを生成したり DLR(動的言語ランタイム) を使用したりしないので、比較的小さなスクリプトを高速に実行することができます。Nuget の PCL(ポータブルクラスライブラリ)として https://www.nuget.org/packages/Jint から入手できます。

# 特徴

- ECMAScript 5.1 をフルサポート - http://www.ecma-international.org/ecma-262/5.1/
- .NET のポータブルクラスライブラリ - http://msdn.microsoft.com/en-us/library/gg597391(v=vs.110).aspx
- .NET との相互運用性

> ECMAScript 6.0 は現在実装が進行中です。https://github.com/sebastienros/jint/issues/343 を参照してください。

# 議論

[Gitter](https://gitter.im/sebastienros/jint)のチャットに参加するか、[stackoverflow](http://stackoverflow.com/questions/tagged/jint)の `jint` タグで質問を投稿してください。

# 例

この例では、`Console.WriteLine` を指す `log` という新しい値を定義し、`log('Hello World!')` を呼び出すスクリプトを実行します。
```c#
    var engine = new Engine()
        .SetValue("log", new Action<object>(Console.WriteLine))
        ;

    engine.Execute(@"
      function hello() {
        log('Hello World');
      };

      hello();
    ");
```
ここでは、変数 `x` を `3` に設定して `x * x` を JavaScript で実行しています。結果は .NET に直接返されますが、この場合は二倍の値 `9` として返されます。
```c#
    var square = new Engine()
        .SetValue("x", 3)     // 新しい値を定義
        .Execute("x * x")     // 処理を実行
        .GetCompletionValue() // 最後のステートメントが完了した時の値を取得
        .ToObject()           // .NET 用の値に変換
        ;
```
移植可能なコンポーネントや匿名オブジェクトを直接渡して JavaScript から利用することもできます。この例では、新しい `Person` インスタンスを JavaScriptから操作しています。
```c#
    var p = new Person {
        Name = "Mickey Mouse"
    };

    var engine = new Engine()
        .SetValue("p", p)
        .Execute("p.Name = 'Minnie'")
        ;
    Assert.AreEqual("Minnie", p.Name);
```
JavaScript の関数の参照を呼び出すことができます。
```c#
    var add = new Engine()
        .Execute("function add(a, b) { return a + b; }")
        .GetValue("add")
        ;

    add.Invoke(1, 2); // -> 3
```
また、関数名を直接呼び出すこともできます。
```c#
    var engine = new Engine()
        .Execute("function add(a, b) { return a + b; }")
        ;

    engine.Invoke("add", 1, 2); // -> 3
```
## .NET アセンブリやクラスへのアクセス

このようにエンジンのインスタンスを構成することで、エンジン内で任意の .NET クラスにアクセスできるようにすることができます。
```c#
    var engine = new Engine(cfg => cfg.AllowClr());
```
上記のようにすることで `System` 名前空間にグローバル値としてアクセスできるようになります。コマンドラインユーティリティのコンテキストでの使用方法を以下に示します。
```javascript
    jint> var file = new System.IO.StreamWriter('log.txt');
    jint> file.WriteLine('Hello World !');
    jint> file.Dispose();
```
また、一般的な .NET メソッドへのショートカットを作成することもできます。
```javascript
    jint> var log = System.Console.WriteLine;
    jint> log('Hello World !');
    => "Hello World !"
```
CLR(共通言語ランタイム)を許可している場合、オプションで特定の型をロードするためにカスタムアセンブリを渡すことができます。
```c#
    var engine = new Engine(cfg => cfg
        .AllowClr(typeof(Bar).Assembly)
    );
```
そして、`System` と同じようにローカルの名前空間を割り当てるには `importNamespace` を使用します。
```javascript
    jint> var Foo = importNamespace('Foo');
    jint> var bar = new Foo.Bar();
    jint> log(bar.ToString());
```
特定の CLR 型の参照を追加するには、次のようにします。
```csharp
engine.SetValue("TheType", TypeReference.CreateTypeReference(engine, typeof(TheType)))
```
そしてこのように使用します。
```javascript
    jint> var o = new TheType();
```
ジェネリック型もサポートされています。ここでは、`List<string>` の宣言、インスタンス化、使用方法を説明します。
```javascript
    jint> var ListOfString = System.Collections.Generic.List(System.String);
    jint> var list = new ListOfString();
    jint> list.Add('foo');
    jint> list.Add(1); // 自動的に String に変換される
    jint> list.Count;  // 2
```

## Internationalization

You can enforce what Time Zone or Culture the engine should use when locale JavaScript methods are used if you don't want to use the computer's default values.

This example forces the Time Zone to Pacific Standard Time.
```c#
    var PST = TimeZoneInfo.FindSystemTimeZoneById("Pacific Standard Time");
    var engine = new Engine(cfg => cfg.LocalTimeZone(PST));
    
    engine.Execute("new Date().toString()"); // Wed Dec 31 1969 16:00:00 GMT-08:00
```

This example is using French as the default culture.
```c#
    var FR = CultureInfo.GetCultureInfo("fr-FR");
    var engine = new Engine(cfg => cfg.Culture(FR));
    
    engine.Execute("new Number(1.23).toString()"); // 1.23
    engine.Execute("new Number(1.23).toLocaleString()"); // 1,23
```

## Constraints 

Constraints are used during script execution to ensure that requirements around resource consumption are met, for example:

* Scripts should not use more than X memory.
* Scripts should only run for a maximum amount of time.

You can configure them via the options:

```c#
var engine = new Engine(options => {

    // Limit memory allocations to MB
    options.LimitMemory(4_000_000);

    // Set a timeout to 4 seconds.
    options.TimeoutInterval(TimeSpan.FromSeconds(4));

    // Set limit of 1000 executed statements.
    options.MaxStatements(1000);

    // Use a cancellation token.
    options.CancellationToken(cancellationToken);
}
```

You can also write a custom constraint by implementing the `IConstraint` interface:

```c#
public interface IConstraint
{
    /// Called before a script is executed and useful when you us an engine object for multiple executions.
    void Reset();

    // Called before each statement to check if your requirements are met.
    void Check();
}
```

For example we can write a constraint that stops scripts when the CPU usage gets too high:

```c#
class MyCPUConstraint : IConstraint
{
    public void Reset()
    {
    }

    public void Check()
    {
        var cpuUsage = GetCPUUsage();

        if (cpuUsage > 0.8) // 80%
        {
            throw new OperationCancelledException();
        }
    }
}

var engine = new Engine(options =>
{
    options.Constraint(new MyCPUConstraint());
});
```

When you reuse the engine you want to use cancellation tokens you have to reset the token before each call of `Execute`:

```c#
var constraint = new CancellationConstraint();

var engine = new Engine(options =>
{
    options.Constraint(constraint);
});

for (var i = 0; i < 10; i++) 
{
    using (var tcs = new CancellationTokenSource(TimeSpan.FromSeconds(10)))
    {
        constraint.Reset(tcs.Token);

        engine.SetValue("a", 1);
        engine.Execute("a++");
    }
}
```

## Implemented features:

### ECMAScript 5.1

- Complete implementation
  - ECMAScript 5.1 test suite (http://test262.ecmascript.org/) 

### ECMAScript 6.0

ES6 features which are being implemented:
- [x] [arrows](https://github.com/lukehoban/es6features/blob/master/README.md#arrows)
- [ ] [classes](https://github.com/lukehoban/es6features/blob/master/README.md#classes)
- [x] [enhanced object literals](https://github.com/lukehoban/es6features/blob/master/README.md#enhanced-object-literals)
- [x] [template strings](https://github.com/lukehoban/es6features/blob/master/README.md#template-strings)
- [x] [destructuring](https://github.com/lukehoban/es6features/blob/master/README.md#destructuring)
- [x] [default + rest + spread](https://github.com/lukehoban/es6features/blob/master/README.md#default--rest--spread)
- [x] [let + const](https://github.com/lukehoban/es6features/blob/master/README.md#let--const)
- [x] [iterators + for..of](https://github.com/lukehoban/es6features/blob/master/README.md#iterators--forof)
- [ ] [generators](https://github.com/lukehoban/es6features/blob/master/README.md#generators)
- [ ] [unicode](https://github.com/lukehoban/es6features/blob/master/README.md#unicode)
- [ ] [modules](https://github.com/lukehoban/es6features/blob/master/README.md#modules)
- [ ] [module loaders](https://github.com/lukehoban/es6features/blob/master/README.md#module-loaders)
- [x] [map + set](https://github.com/lukehoban/es6features/blob/master/README.md#map--set--weakmap--weakset)
- [ ] [weakmap + weakset](https://github.com/lukehoban/es6features/blob/master/README.md#map--set--weakmap--weakset)
- [x] [proxies](https://github.com/lukehoban/es6features/blob/master/README.md#proxies)
- [x] [symbols](https://github.com/lukehoban/es6features/blob/master/README.md#symbols)
- [ ] [subclassable built-ins](https://github.com/lukehoban/es6features/blob/master/README.md#subclassable-built-ins)
- [ ] [promises](https://github.com/lukehoban/es6features/blob/master/README.md#promises)
- [x] [math APIs](https://github.com/lukehoban/es6features/blob/master/README.md#math--number--string--array--object-apis)
- [x] [number APIs](https://github.com/lukehoban/es6features/blob/master/README.md#math--number--string--array--object-apis)
- [x] [string APIs](https://github.com/lukehoban/es6features/blob/master/README.md#math--number--string--array--object-apis)
- [x] [array APIs](https://github.com/lukehoban/es6features/blob/master/README.md#math--number--string--array--object-apis)
- [x] [object APIs](https://github.com/lukehoban/es6features/blob/master/README.md#math--number--string--array--object-apis)
- [x] [binary and octal literals](https://github.com/lukehoban/es6features/blob/master/README.md#binary-and-octal-literals)
- [x] [reflect api](https://github.com/lukehoban/es6features/blob/master/README.md#reflect-api)
- [ ] [tail calls](https://github.com/lukehoban/es6features/blob/master/README.md#tail-calls)

### .NET Interoperability

- Manipulate CLR objects from JavaScript, including:
  - Single values
  - Objects
    - Properties
    - Methods
  - Delegates
  - Anonymous objects
- Convert JavaScript values to CLR objects
  - Primitive values
  - Object -> expando objects (`IDictionary<string, object>` and dynamic)
  - Array -> object[]
  - Date -> DateTime
  - number -> double
  - string -> string
  - boolean -> bool
  - Regex -> RegExp
  - Function -> Delegate

### Security

The following features provide you with a secure, sand-boxed environment to run user scripts.

- Define memory limits, to prevent allocations from depleting the memory.
- Enable/disable usage of BCL to prevent scripts from invoking .NET code.
- Limit number of statements to prevent infinite loops.
- Limit depth of calls to prevent deep recursion calls.
- Define a timeout, to prevent scripts from taking too long to finish.

Continuous Integration kindly provided by  [AppVeyor](https://www.appveyor.com)

### Branches and releases

- The recommended branch is __dev__, any PR should target this branch
- The __dev__ branch is automatically built and published on [Myget](https://www.myget.org/feed/Packages/jint)
- The __dev__ branch is occasionally merged to __master__ and published on [NuGet](https://www.nuget.org/packages/jint)
- The 3.x releases have more features (from es6) and is faster than the 2.x ones. They run the same test suite so they are as reliable. For instance [RavenDB](https://github.com/ravendb/ravendb) is using the 3.x version.
- The 3.x versions are marked as _beta_ as they might get breaking changes while es6 features are added. 