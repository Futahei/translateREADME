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

## 国際化

`TimeZone` や `Culture` に関して、コンピューターの既定値を使いたくない場合、自身で設定することができます。

この例では、`TimeZone` を太平洋標準時に変更しています。
```c#
    var PST = TimeZoneInfo.FindSystemTimeZoneById("Pacific Standard Time");
    var engine = new Engine(cfg => cfg.LocalTimeZone(PST));

    engine.Execute("new Date().toString()"); // Wed Dec 31 1969 16:00:00 GMT-08:00
```

この例では、`Culture` をフランスのものに変更しています。
```c#
    var FR = CultureInfo.GetCultureInfo("fr-FR");
    var engine = new Engine(cfg => cfg.Culture(FR));

    engine.Execute("new Number(1.23).toString()"); // 1.23
    engine.Execute("new Number(1.23).toLocaleString()"); // 1,23
```

## 制約

制約は、スクリプトの実行中に使用され、リソース消費に関する要件を確実に満たすために使用されます。例えば、

* スクリプトは設定した以上のメモリを使用してはいけません。
* スクリプトは設定した最大の時間以上実行されません。

これらはオプションから設定することができます。

```c#
var engine = new Engine(options => {

    // メモリの割り当てを4MBに制限します
    options.LimitMemory(4_000_000);

    // タイムアウトを4秒に設定します
    options.TimeoutInterval(TimeSpan.FromSeconds(4));

    // 最大ステートメントを1000に設定します
    options.MaxStatements(1000);

    // キャンセルトークンを使用します
    options.CancellationToken(cancellationToken);
}
```

`IConstraint` インターフェースを実装することで、カスタム制約を使用することができます。

```c#
public interface IConstraint
{
    /// スクリプトが実行される前に呼び出されます。エンジンで複数回実行をする場合便利です。
    void Reset();

    // 各ステートメントの前に呼び出され、要件が満たされているかどうかを確認します。
    void Check();
}
```

例えば、CPU使用率が高くなりすぎたときにスクリプトを停止するような制約を書くことができます。

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

キャンセルトークンを利用したエンジンを再利用する場合は、`Execute` を呼び出す前にトークンをリセットする必要があります。

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

## 実装されている機能

### ECMAScript 5.1

- 全ての機能が実装されています。
  - ECMAScript 5.1 test suite (http://test262.ecmascript.org/)

### ECMAScript 6.0

- 実装されている ES6 の機能
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

### .NETとの相互作用

- JavaScript から以下のような CLR オブジェクトを操作します。
  - Single values
  - Objects
    - Properties
    - Methods
  - Delegates
  - Anonymous objects
- JavaScript の値を CLR オブジェクトに変換します。
  - Primitive values
  - Object -> expando objects (`IDictionary<string, object>` and dynamic)
  - Array -> object[]
  - Date -> DateTime
  - number -> double
  - string -> string
  - boolean -> bool
  - Regex -> RegExp
  - Function -> Delegate

### セキュリティ

以下の機能により、ユーザースクリプトを実行するための安全なサンドボックス環境を提供します。

- メモリ制限を定義して、割り当てられたメモリが枯渇しないようにします。
- スクリプトが .NET コードを呼び出すのを防ぐために、BCL (基本クラスライブラリ) の使用を有効または無効にします。
- 無限ループを防ぐためにステートメントの数を制限します。
- 呼び出しの深さを制限して、深い再帰呼び出しを防止します。
- タイムアウトを定義し、スクリプトの終了に時間がかかりすぎないようにします。

継続的インテグレーションは [AppVeyor](https://www.appveyor.com) からのご厚意により提供されています。

### ブランチとリリース

- 推奨されるブランチは __dev__ です。
- __dev__ ブランチは自動的にビルドされて [Myget](https://www.myget.org/feed/Packages/jint) 上で公開されます。
- __dev__ ブランチは時々 __master__ にマージされ、[NuGet](https://www.nuget.org/packages/jint) で公開入れています。
- 3.x のリリースはより多くの機能(es6)を持ち、2.x のものよりも高速に動作します。これらのリリースは同じテストスイートを実行しているので、信頼性は同じです。例えば、[RavenDB](https://github.com/ravendb/ravendb) ではバージョン 3.x を使用しています。
- バージョン 3.x は、es6 の機能が順次追加される間に様々な変更が加えられる可能性があるため、_ベータ_ 版として公開されています。