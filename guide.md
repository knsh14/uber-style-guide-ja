# 導入
コーディングスタイルは私達のコードを統治する規則です。
これらのスタイルは、gofmt がやってくれることから少しだけ発展したものです。

このガイドのゴールはUber社内でのGoのコードでやるべき、もしくはやるべからずを説明し、コードの複雑さを管理することです。
これらのルールはコードを管理しやすくし、かつエンジニアがGoの言語機能をより生産的に利用できるようにします。

このガイドは元々同僚がGoを使ってより開発しやすくするために[Prashant Varanasi]( https://github.com/prashantv )と[Simon Newton]( https://github.com/nomis52 )によって作成されました。
長年にわたって多くのフィードバックを受けて修正されています。

このドキュメントはUber社内で使われる規則を文書化したものです。
多くは以下のリソースでも見ることができるような一般的なものです。

1. Effective Go
2. The Go common mistakes guide

全てのコードは `golint` や `go vet` を通してエラーが出ない状態にするべきです。
エディタに以下の設定を導入することを推奨しています。

1. 保存するごとに `goimports` を実行する
2. `golint` と `go vet` を実行してエラーがないかチェックする

Goのエディタのサポートについては以下の資料を参考にしてください。
https://github.com/golang/go/wiki/IDEsAndTextEditorPlugins

# ガイドライン
## Pointers to Interfaces
インタフェースをポインタとして渡す必要はほぼありません。
インタフェースは値として渡すべきです。
ただインタフェースを実装している要素はポインタでも大丈夫です。

インタフェースには2種類あります。

1. 型付けされた情報へのポインタ。これは type と考えることができます。
2. データポインタ。格納されたデータがポインタならそのまま使えます。格納されたデータが値ならその値のポインタになります。

もしインタフェースのメソッドがそのインタフェースを満たした型のデータをいじりたいなら、インタフェースの裏側の型はポインタである必要があります。

## Verify Interface Compliance
コンパイル時にインタフェースが適切に実装されているかチェックしましょう。
これは以下のことを指します。

* 公開された型がAPIとして適切に要求されたインタフェースを実装しているか
* 公開されてるかどうかに関わらず、ある型の集合が同じインタフェースを実装しているか
* その他にインタフェースを実装しなければ利用できなくなるケース

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Handler struct {
  // ...
}



func (h *Handler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  ...
}
```

</td><td>

```go
type Handler struct {
  // ...
}

var _ http.Handler = (*Handler)(nil)

func (h *Handler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  // ...
}
```

</td></tr>
</tbody></table>

`var _ http.Handler = (*Handler)(nil)` という式は `*Handler` 型が `http.Handler` インタフェースを実装していなければ、コンパイルエラーになります。

代入式の右辺はゼロ値にするべきです。
ポインタ型やスライス、マップなどは `nil` ですし、構造体ならその型の空の構造体にします。

```go
type LogHandler struct {
  h   http.Handler
  log *zap.Logger
}

var _ http.Handler = LogHandler{}

func (h LogHandler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  // ...
}
```

## Receivers and Interfaces
レシーバーが値のメソッドはレシーバーがポインタでも呼び出すことができますが、逆はできません。

```Go
type S struct {
  data string
}

func (s S) Read() string {
  return s.data
}

func (s *S) Write(str string) {
  s.data = str
}

sVals := map[int]S{1: {"A"}}

// You can only call Read using a value
sVals[1].Read()

// This will not compile:
//  sVals[1].Write("test")

sPtrs := map[int]*S{1: {"A"}}

// You can call both Read and Write using a pointer
sPtrs[1].Read()
sPtrs[1].Write("test")
```

同じように、メソッドのレシーバーが値型でも、ポインタがインタフェースを満たしているとみなされます。

```
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

s1Val := S1{}
s1Ptr := &S1{}
s2Val := S2{}
s2Ptr := &S2{}

var i F
i = s1Val
i = s1Ptr
i = s2Ptr

// The following doesn't compile, since s2Val is a value, and there is no value receiver for f.
//   i = s2Val
```

Effective Go の [Pointers vs Values]( https://golang.org/doc/effective_go.html#pointers_vs_values )を見るとよいでしょう。

## Zero-value Mutexes are Valid
`sync.Mutex` や `sync.RWMutex` はゼロ値でも有効です。ポインタで扱う必要はありません。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
mu := new(sync.Mutex)
mu.Lock()
```

</td><td>

```go
var mu sync.Mutex
mu.Lock()
```

</td></tr>
</tbody></table>

もし構造体のポインタを使う場合、mutexはポインタでないフィールドにする必要があります。
外部に公開されてない構造体なら、mutexを埋め込みで使うこともできます。


<table>
<tbody>
<tr><td>

```go
type smap struct {
  sync.Mutex // only for unexported types

  data map[string]string
}

func newSMap() *smap {
  return &smap{
    data: make(map[string]string),
  }
}

func (m *smap) Get(k string) string {
  m.Lock()
  defer m.Unlock()

  return m.data[k]
}
```

</td><td>

```go
type SMap struct {
  mu sync.Mutex

  data map[string]string
}

func NewSMap() *SMap {
  return &SMap{
    data: make(map[string]string),
  }
}

func (m *SMap) Get(k string) string {
  m.mu.Lock()
  defer m.mu.Unlock()

  return m.data[k]
}
```

</td></tr>

</tr>
<tr>
<td>インターナルな型やmutexのインタフェースを実装している必要がある場合には埋め込みを使う</td>
<td>公開されている型にはプライベートなフィールドを使う</td>
</tr>

</tbody></table>

## Copy Slices and Maps at Boundaries
スライスやマップは内部でデータへのポインタが含まれています。なのでコピーする際には注意してください。

### Receiving Slices and Maps
引数として受け取ってフィールドに保存したスライスは、他の箇所でデータが書き換わる可能性があることを覚えておいてください。

<table>
<thead><tr><th>Bad</th> <th>Good</th></tr></thead>
<tbody>
<tr>
<td>

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = trips
}

trips := ...
d1.SetTrips(trips)

// ここで値が変わると d1.trips[0] も変わる
trips[0] = ...
```

</td>
<td>

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}

trips := ...
d1.SetTrips(trips)

// d1.trips に変更が及ばない
trips[0] = ...
```

</td>
</tr>

</tbody>
</table>

### Returning Slices and Maps
同じように、公開せずに内部に保持しているスライスやマップが変更されることもあります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Stats struct {
  mu sync.Mutex
  counters map[string]int
}

// Snapshot returns the current stats.
func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  return s.counters
}

// snapshot は mutex で守られない
// レースコンディションが起きる
snapshot := stats.Snapshot()
```

</td><td>

```go
type Stats struct {
  mu sync.Mutex
  counters map[string]int
}

func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  result := make(map[string]int, len(s.counters))
  for k, v := range s.counters {
    result[k] = v
  }
  return result
}

// snapshot はただのコピーなので変更しても影響はない
snapshot := stats.Snapshot()
```

</td></tr>
</tbody></table>

## Defer to Clean Up
ファイルや mutex のロックなどをクリーンアップするために defer を使おう

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
p.Lock()
if p.count < 10 {
  p.Unlock()
  return p.count
}

p.count++
newCount := p.count
p.Unlock()

return newCount

// easy to miss unlocks due to multiple returns
```

</td><td>

```go
p.Lock()
defer p.Unlock()

if p.count < 10 {
  return p.count
}

p.count++
return p.count

// more readable
```

</td></tr>
</tbody></table>

defer のオーバーヘッドは非常に小さいです。
関数の実行時間がナノ秒のオーダーである場合には避ける必要があります。
defer を使ってほんの少しの実行コストを払えば可読性がとてもあがります。
これはシンプルなメモリアクセス以上の計算が必要な大きなメソッドに特に当てはまります。

## Channel Size is One or None
channel のサイズは普段は1もしくはバッファなしのものにするべきです。
デフォルトでは channel はバッファなしでサイズが0になっています。
それより大きいサイズにする場合はよく考える必要があります。
どのようにしてサイズを決定するのか、チャネルがいっぱいになり処理がブロックされたときにどのような挙動をするかよく考える必要があります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// Ought to be enough for anybody!
c := make(chan int, 64)
```

</td><td>

```go
// Size of one
c := make(chan int, 1) // or
// Unbuffered channel, size of zero
c := make(chan int)
```

</td></tr>
</tbody></table>

## Start Enums at One
Go で enum を導入するときの標準的な方法は、型を定義して `const` のグループを作り、初期値を `iota` にすることです。
変数のデフォルト値はゼロ値です。なので通常はゼロ値ではない値から enum を始めるべきでしょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Operation int

const (
  Add Operation = iota
  Subtract
  Multiply
)

// Add=0, Subtract=1, Multiply=2
```

</td><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)

// Add=1, Subtract=2, Multiply=3
```

</td></tr>
</tbody></table>

ただゼロ値を使うことに意味があるケースもあります。
例えばゼロ値をデフォルトの挙動として扱いたい場合です。

```go
type LogOutput int

const (
  LogToStdout LogOutput = iota
  LogToFile
  LogToRemote
)

// LogToStdout=0, LogToFile=1, LogToRemote=2
```

## Use `"time"` to handle time
時間を正しく扱うのは非常に困難です。
時間に対する誤解には次のようなものがあります。

1. 1日は24時間である
2. 1時間は60分である
3. 1週間は7日である
4. 1年は365日である
5. [などなど]( https://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time )

例えば、1番について考えると、単純に24時間を足すだけでは正しくカレンダー上次の日になるとは限りません。

そのため、時間を扱う場合は常に[time]( https://pkg.go.dev/time?tab=doc )パッケージを使いましょう。
なぜならこのパッケージで前述の誤解を安全に処理することができるからです。

### Use `time.Time` for instants of time
時刻を扱うときは[time.Time]( https://pkg.go.dev/time?tab=doc#Time )型を使いましょう。
また、時刻を比較したり、足し引きする際にも[time.Time]( https://pkg.go.dev/time?tab=doc#Time )型のメソッドを使いましょう

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func isActive(now, start, stop int) bool {
  return start <= now && now < stop
}
```

</td><td>

```go
func isActive(now, start, stop time.Time) bool {
  return (start.Before(now) || start.Equal(now)) && now.Before(stop)
}
```

</td></tr>
</tbody></table>

### Use `time.Duration` for periods of time
期間を扱うときには[time.Duration]( https://pkg.go.dev/time?tab=doc#Duration )型を使いましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func poll(delay int) {
  for {
    // ...
    time.Sleep(time.Duration(delay) * time.Millisecond)
  }
}

poll(10) // was it seconds or milliseconds?
```

</td><td>

```go
func poll(delay time.Duration) {
  for {
    // ...
    time.Sleep(delay)
  }
}

poll(10*time.Second)
```

</td></tr>
</tbody></table>

時刻に24時間を足す例に戻ります。
もしカレンダー上で次の日の同じ時刻にしたい場合は [`time.AddDate`]( https://pkg.go.dev/time?tab=doc#Time.AddDate )メソッドを使います。
もしその時刻から正確に24時間後にしたい場合は[`time.Add`]( https://pkg.go.dev/time?tab=doc#Time.Add )メソッドを使います。

```go
newDay := t.AddDate(0 /* years */, 0, /* months */, 1 /* days */)
maybeNewDay := t.Add(24 * time.Hour)
```

### Use `time.Time` and `time.Duration` with external systems

できるなら外部システムとのやり取りにも `time.Time` 型や `time.Duration` 型を使うようにしましょう。

* Command-line flags: [`flag`]( https://golang.org/pkg/flag/ ) パッケージは[`time.ParseDuration`]( https://golang.org/pkg/time/#ParseDuration )を使うことで `time.Duration` 型をサポートできます
* JSON: [`encoding/json`]( https://golang.org/pkg/encoding/json/ )パッケージは[`Unmarshal` メソッド]( https://golang.org/pkg/time/#Time.UnmarshalJSON )によって[RFC 3339]( https://tools.ietf.org/html/rfc3339 )フォーマットの時刻を `time.Time` 型にエンコーディングできます
* SQL: [`database/sql`]( https://golang.org/pkg/database/sql/ )パッケージではもしドライバーがサポートしていれば `DATETIME` や `TIMESTAMP` 型のカラムを `time.Time` 型にすることができます
* YAML: [`gopkg.in/yaml.v2`]( https://godoc.org/gopkg.in/yaml.v2 )パッケージは[`time.ParseDuration`]( https://golang.org/pkg/time/#ParseDuration )によって[RFC 3339]( https://tools.ietf.org/html/rfc3339 )フォーマットの時刻を `time.Time` 型にエンコーディングできます

もし `time.Duration` が使えないなら、`int` 型や `float64` 型を使ってフィールド名に単位をもたせるようにしましょう。
次の表のようにします。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// {"interval": 2}
type Config struct {
  Interval int `json:"interval"`
}
```

</td><td>

```go
// {"intervalMillis": 2000}
type Config struct {
  IntervalMillis int `json:"intervalMillis"`
}
```

</td></tr>
</tbody></table>

もしこれらのインタラクションで `time.Time` を使えない場合には、[RFC 3339]( https://tools.ietf.org/html/rfc3339 )フォーマットの `string` 型を使うようにしましょう。
このフォーマットは[time.UnmarshalText]( https://golang.org/pkg/time/#Time.UnmarshalText )メソッドの中でも使われますし、`time.Parse` や `time.Format` 関数でも [`time.RFC3339`]( https://pkg.go.dev/time?tab=doc#RFC3339 )と組み合わせて使えます。

## Errors

### Error Types
エラーを定義する方法にはいくつかの種類があります。
ユースケースに合った最適なものを選ぶために以下のことを考慮しましょう。

* 呼び出し側は自身でエラーをハンドリングするためにエラーを検知する必要がありますか？
  その場合、上位のエラー変数または自前の型を定義することで [`errors.Is`] と [`errors.As`] 関数を利用できるようにサポートしなければなりません。
* エラーメッセージは静的な文字列ですか？またはコンテキストを持つ情報が必要な動的な文字列ですか？
  前者ならば [`errors.New`] が利用できます。後者ならば [`fmt.Errorf`] または自前のエラー型を利用しなければなりません。
* 下流のエラーを更に上流に返していますか？もしそうならば[Error Wrappingのセクション](#error-wrapping)を参照してください。

[`errors.Is`]: https://golang.org/pkg/errors/#Is
[`errors.As`]: https://golang.org/pkg/errors/#As

[`errors.New`]: https://golang.org/pkg/errors/#New
[`fmt.Errorf`]: https://golang.org/pkg/fmt/#Errorf

| エラーを検知? | エラーメッセージ | アドバイス                          |
|-----------------|---------------|-----------------------------------|
| いいえ           | 静的        | [`errors.New`]                      |
| いいえ           | 動的       | [`fmt.Errorf`]                      |
| はい             | 静的        | [`errors.New`]を使ったパッケージ変数(`var`で定義) |
| はい             | 動的       |  自前のエラー型                 |

例えば、静的文字列のエラーならば [`errors.New`] を利用しましょう。
呼び出し側がエラーを検知しハンドリングする必要がある場合は、そのエラーをパッケージ変数とし`errors.Is`で検知できるようにしましょう。

<table>
<thead><tr><th>No error matching</th><th>Error matching</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open() error {
  return errors.New("could not open")
}

// package bar

if err := foo.Open(); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

</td><td>

```go
// package foo

var ErrCouldNotOpen = errors.New("could not open")

func Open() error {
  return ErrCouldNotOpen
}

// package bar

if err := foo.Open(); err != nil {
  if errors.Is(err, foo.ErrCouldNotOpen) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

動的文字列のエラーの場合、呼び出し側がエラー検知する必要がないならば [`fmt.Errorf`] を使い、検知する必要があるならば自前の`error`インターフェースを実装する型を使いましょう。

<table>
<thead><tr><th>No error matching</th><th>Error matching</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open(file string) error {
  return fmt.Errorf("file %q not found", file)
}

// package bar

if err := foo.Open("testfile.txt"); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

</td><td>

```go
// package foo

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

func Open(file string) error {
  return &NotFoundError{File: file}
}


// package bar

if err := foo.Open("testfile.txt"); err != nil {
  var notFound *NotFoundError
  if errors.As(err, &notFound) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

自前のエラー型を公開する場合、それもパッケージの公開APIの一部になることに留意しましょう。

```go
// package foo

type errNotFound struct {
  file string
}

func (e errNotFound) Error() string {
  return fmt.Sprintf("file %q not found", e.file)
}

func IsNotFoundError(err error) bool {
  _, ok := err.(errNotFound)
  return ok
}

func Open(file string) error {
  return errNotFound{file: file}
}

// package bar

if err := foo.Open("foo"); err != nil {
  if foo.IsNotFoundError(err) {
    // handle
  } else {
    panic("unknown error")
  }
}
```

### Error Wrapping
エラーを伝搬させるためには以下の3つの方法が主流です。

* 受けたエラーをそのまま返す。
* `fmt.Errorf` に `%w` を付けてコンテキストを追加する。
* `fmt.Errorf` に `%v` を付けてコンテキストを追加する。

追加するコンテキストが無いならばエラーをそのまま返しましょう。これによりオリジナルのエラー型とメッセージが保たれます。これは下流のエラーメッセージにエラーがどこから来たか追うための十分な情報がある場合に適しています。

別の方法として、"connection refused"のような曖昧なエラーではなく、"call service foo: connection refused"のようなより有益なエラーを得られるように、可能な限りエラーメッセージにコンテキストを追加することもできます。

エラーにコンテキストを追加するには`fmt.Errorf`を使いましょう。このとき、呼び出し側がエラー元の原因を抽出し検知できるようにするべきかどうかに基づき`%w`または`%v`を選ぶことになります。

* 呼び出し側が原因のエラー元を把握する必要がある場合は`%w`を使いましょう。これはほとんどのラップされたエラーにとって良いデフォルトの振る舞いになりますが、呼び出し側がそれに依存しだすかもしないことを考慮しましょう。そのため、ラップされたエラーが既知の変数(var)か型(type)であるケースでは、関数の責務としてその振る舞いのコードドキュメント記載とテストをしましょう。
* 原因のエラー元をあえて曖昧にする場合`%v`を使いましょう。呼び出し側はエラー検知をすることができなくなりますが、将来必要なときに`%w`を使うように変更できます。

返されたエラーにコンテキストを追加する場合、"failed to"のようなエラーがスタックに蓄積されるあたって明白な表現は避け、コンテキストを簡潔に保つようにしてください。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "failed to create new store: %w", err)
}
```

</td><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "new store: %w", err)
}
```

</td></tr><tr><td>

```
failed to x: failed to y: failed to create new store: the error
```

</td><td>

```
x: y: new store: the error
```

</td></tr>
</tbody></table>

しかし、エラーメッセージが他のシステムに送られる場合は"err"タグを付けたり"Failed"プレフィックスをつけたりすることでエラーメッセージであることを明確にする必要があります。

[Don't just check errors, handle them gracefully]の記事も参照してください。

  [Don't just check errors, handle them gracefully]: https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully

### Error Naming

グローバル変数として使われるエラー値においてそれがパブリックかプライベートかによって`Err`または`err`のプレフィックスを付けましょう。
この助言は[Prefix Unexported Globals with _](#prefix-unexported-globals-with-_)の助言よりも優先されます。

```go
var (
  // 以下の2つのエラーはパブリックであるため
  // このパッケージの利用者はこれらのエラーを
  // errors.Isで検知することができる。

  ErrBrokenLink = errors.New("link is broken")
  ErrCouldNotOpen = errors.New("could not open")

  // このエラーはパッケージのパブリックAPIに
  // させたくないのでプライベートにしている。
  // errors.Isでパッケージ内にてこのエラーを
  // 使うことができる。

  errNotFound = errors.New("not found")
)
```

カスタムエラー型の場合、`Error`を末尾に付けるようにしましょう。

```go
// 同様にこのエラーはパブリックであるため
// このパッケージの利用者はこれらのエラーを
// errors.Asで検知することができる。

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

// このエラーはパッケージのパブリックAPIに
// させたくないのでプライベートにしている。
// errors.Asでパッケージ内にてこのエラーを
// 使うことができる。

type resolveError struct {
  Path string
}

func (e *resolveError) Error() string {
  return fmt.Sprintf("resolve %q", e.Path)
}
```

### Handle Errors Once
呼び出し側が呼び出し先からエラーを受け取ったとき、エラーについてどれほど知っているかで様々な方法があります。

主にこれらですが、他にも色々あります。

- もし、呼び出し先が特定のエラーを定義しているなら、[`errors.Is`]や[`errors.As`]を使って処理を分岐させます。
- もし復帰可能なエラーなら、エラーをログに出力し、安全に処理を継続しましょう
- もしドメインで定義された条件を満たさないエラーなら、事前に定義したエラーを返しましょう
- エラーを返すときは、[Wrap]( #error-wrapping ) するか、一つ一つ忠実に返しましょう。

呼び出し元がどのようにエラーを扱うかに関わらず、通常はどのエラーも一度だけ処理するべきです。例えば、呼び出し元の更に呼び出し元も同様にエラーを処理するので、呼び出し元はエラーをログに記録してから返すべきではありません。

次の具体例を考えます。

<table>
<thead><tr><th>Description</th><th>Code</th></tr></thead>
<tbody>
<tr><td>

**悪い例**: エラーをログに書き出してから返す。

より上位の呼び出し先も同様にエラーをログに出力する可能性があるので、アプリケーションログに多くのノイズが混ざりこのログの価値が薄まります。

</td><td>

```go
u, err := getUser(id)
if err != nil {
  // BAD: See description
  log.Printf("Could not get user %q: %v", id, err)
  return err
}
```

</td></tr>
<tr><td>

**良い例**: エラーをラップして返す。

より上位の呼び出し側がエラーを処理します。 `%w` を使うと、[`errors.Is`]や[`errors.As`]を使ってエラーマッチさせることができます。

</td><td>

```go
u, err := getUser(id)
if err != nil {
  return fmt.Errorf("get user %q: %w", id, err)
}
```

</td></tr>
<tr><td>

**良い例**: エラーをログに出力し、処理を継続する。

もしその処理が絶対必要でないなら、品質は下がりますが復旧して処理を続けることができます。
</td><td>

```go
if err := emitMetrics(); err != nil {
  // メトリクスの書き出し失敗は処理を止めるほどではない
  log.Printf("Could not emit metrics: %v", err)
}

```

</td></tr>
<tr><td>

**良い例**: エラーマッチさせて処理を継続する。

もし呼び出し元がエラーを定義していて、そのエラーが復旧可能なら、エラーを確認して処理を継続しましょう。
その他のエラーだった場合はエラーをラップして返しましょう。

ラップされたエラーは上位の呼び出し元で処理させましょう。
</td><td>

```go
tz, err := getUserTimeZone(id)
if err != nil {
  if errors.Is(err, ErrUserNotFound) {
    // もしユーザーがみつからないなら UTC を使う
    tz = time.UTC
  } else {
    return fmt.Errorf("get user %q: %w", id, err)
  }
}
```

</td></tr>
</tbody></table>

## Handle Type Assertion Failures
[型アサーション]( https://golang.org/ref/spec#Type_assertions )で1つの戻り値を受け取る場合、その型でなかったらパニックを起こします。
型アサーションではその型に変換できたかを示すbool値も同時に返ってくるので、それで事前にチェックしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
t := i.(string)
```

</td><td>

```go
t, ok := i.(string)
if !ok {
  // 安全にエラーを処理する
}
```

</td></tr>
</tbody></table>

## Don't Panic
プロダクションで動くコードはパニックを避けなければいけません。
パニックは連鎖的障害の主な原因です。
もしエラーが起きた場合、関数はエラーを返して、呼び出し元がどのようにエラーをハンドリングするか決めさせる必要があります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func foo(bar string) {
  if len(bar) == 0 {
    panic("bar must not be empty")
  }
  // ...
}

func main() {
  if len(os.Args) != 2 {
    fmt.Println("USAGE: foo <bar>")
    os.Exit(1)
  }
  foo(os.Args[1])
}
```

</td><td>

```go
func foo(bar string) error {
  if len(bar) == 0 {
    return errors.New("bar must not be empty")
  }
  // ...
  return nil
}

func main() {
  if len(os.Args) != 2 {
    fmt.Println("USAGE: foo <bar>")
    os.Exit(1)
  }
  if err := foo(os.Args[1]); err != nil {
    panic(err)
  }
}
```

</td></tr>
</tbody></table>

`panic`と`recover`はエラーハンドリングではありません。
プログラムはnil参照などの回復不可能な状況が発生したとき以外は出すべきではありません。
ただプログラムの初期化時は例外です。
プログラムが開始するときに異常が起きた場合にはpanicを起こしてもよいでしょう。

```go
var _statusTemplate = template.Must(template.New("name").Parse("_statusHTML"))
```

またテストでは、テストが失敗したことを示すためには`panic`ではなくて `t.Fatal` や `t.FailNow` を使うようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  panic("failed to set up test")
}
```

</td><td>

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  t.Fatal("failed to set up test")
}
```

</td></tr>
</tbody></table>

## Use go.uber.org/atomic

[sync/atomic](https://golang.org/pkg/sync/atomic)パッケージによるアトミック操作は`int32`や`int64`といった基本的な型を対象としているため、アトミックに操作すべき変数に対する読み出し・変更操作にアトミック操作を用いるということ(つまりsync/atomicパッケージの関数を使うこと自体)を容易に忘却させます。例では`int32`の変数に普通の読み出し操作を行ってしまっていますが、これはコンパイラの型チェック機構を素通ししてしまっているため潜在的に競合条件のあるコードをコンパイルできてしまっています。

[go.uber.org/atomic](https://godoc.org/go.uber.org/atomic)は実際のデータの型を基底型として隠蔽することによりこれらのアトミック操作に対して型安全性を付与できます。これによって読み出し操作を行う方法はアトミックな操作に限定され、普通の読み出し操作はコンパイラの型チェックの機構によってコンパイル時にはじくことが可能となります。
また`sync/atomic`パッケージに加えて便利な`atomic.Bool`型も提供しています。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type foo struct {
  running int32  // アトミックな操作が必要な変数
}

func (f* foo) start() {
  if atomic.SwapInt32(&f.running, 1) == 1 {
     // すでに実行中
     return
  }
  // Fooを開始
}

func (f *foo) isRunning() bool {
  return f.running == 1  // 競合条件! --> 別スレッドから実行されたatomic.SwapInt32による値の更新が見えないことが起こりうる
}
```

</td><td>

```go
type foo struct {
  running atomic.Bool
}

func (f *foo) start() {
  if f.running.Swap(true) {
     // すでに実行中
     return
  }
  // Fooを開始
}

func (f *foo) isRunning() bool {
  return f.running.Load()  // 読み出し操作がアトミックなため決定論的な振る舞いにになる(実はこの振る舞いはGoのメモリバリア指定がデフォルトSeqCstであることに依存するが、本項では深く触れない)
}
```

</td></tr>
</tbody></table>

## Avoid Mutable Globals
グローバル変数を変更するのは避けましょう。
代わりに依存関係の注入を使って構造体に持たせるようにしましょう。
関数ポインタを他の値と同じように構造体にもたせます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// sign.go

var _timeNow = time.Now

func sign(msg string) string {
  now := _timeNow()
  return signWithTime(msg, now)
}
```

</td><td>

```go
// sign.go

type signer struct {
  now func() time.Time
}

func newSigner() *signer {
  return &signer{
    now: time.Now,
  }
}

func (s *signer) Sign(msg string) string {
  now := s.now()
  return signWithTime(msg, now)
}
```
</td></tr>
<tr><td>

```go
// sign_test.go

func TestSign(t *testing.T) {
  oldTimeNow := _timeNow
  _timeNow = func() time.Time {
    return someFixedTime
  }
  defer func() { _timeNow = oldTimeNow }()

  assert.Equal(t, want, sign(give))
}
```

</td><td>

```go
// sign_test.go

func TestSigner(t *testing.T) {
  s := newSigner()
  s.now = func() time.Time {
    return someFixedTime
  }

  assert.Equal(t, want, s.Sign(give))
}
```

</td></tr>
</tbody></table>


## Avoid Embedding Types in Public Structs
型の埋め込みは実装の詳細を漏らし、型のブラッシュアップを阻害し、ドキュメントが曖昧になってしまいます。

共通の `AbstractList` を使って様々なリスト型を実装すると仮定します。
この場合に `AbstractList` を埋め込むことでリスト操作を実装するのはやめましょう。
代わりにメソッドを再度定義してその中で `AbstractList` のメソッドを実装するようにしましょう。

```go
type AbstractList struct {}

// Add adds an entity to the list.
func (l *AbstractList) Add(e Entity) {
  // ...
}

// Remove removes an entity from the list.
func (l *AbstractList) Remove(e Entity) {
  // ...
}
```
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// ConcreteList is a list of entities.
type ConcreteList struct {
  *AbstractList
}
```

</td><td>

```go
// ConcreteList is a list of entities.
type ConcreteList struct {
  list *AbstractList
}

// Add adds an entity to the list.
func (l *ConcreteList) Add(e Entity) {
  return l.list.Add(e)
}

// Remove removes an entity from the list.
func (l *ConcreteList) Remove(e Entity) {
  return l.list.Remove(e)
}
```

</td></tr>
</tbody></table>

Go では継承が無い代わりに[埋め込み]( https://golang.org/doc/effective_go.html#embedding )を使えます。
外部の型は暗黙的に埋め込まれた型のメソッドを実装しています。
これらのメソッドはデフォルトでは埋め込まれた型のインスタンスのメソッドになります。

構造体は埋め込んだ型と同じ名前のフィールドを作成します。
なので埋め込んだ型が公開されていたら、そのフィールドも公開されます。
後方互換性を保つために、外側の型は埋め込んだ型を保持する必要があります。

埋め込みが必要な場面は殆どありません。
多くは面倒なメソッドの移譲を書かずに済ませるために使われます。

構造体の代わりに AbstractList インタフェースを埋め込むこともできます。
これだと開発者に将来的な自由度をもたせることができます。
しかし、抽象的な実装に依存して実装の詳細が漏れるという問題は解決されません。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// AbstractList is a generalized implementation
// for various kinds of lists of entities.
type AbstractList interface {
  Add(Entity)
  Remove(Entity)
}

// ConcreteList is a list of entities.
type ConcreteList struct {
  AbstractList
}
```

</td><td>

```go
// ConcreteList is a list of entities.
type ConcreteList struct {
  list *AbstractList
}

// Add adds an entity to the list.
func (l *ConcreteList) Add(e Entity) {
  return l.list.Add(e)
}

// Remove removes an entity from the list.
func (l *ConcreteList) Remove(e Entity) {
  return l.list.Remove(e)
}
```

</td></tr>
</tbody></table>

構造体の埋め込みでもインタフェースの埋め込みでも、将来的な型の変更に制限がかかります。

* 埋め込まれたインタフェースにメソッドを追加することは破壊的変更になります
* 埋め込まれた構造体からメソッドを削除すると破壊的変更になります
* 埋め込まれた型を削除することは破壊的変更になります
* 埋め込まれた型を同じインタフェースを実装した別の型に差し替える場合も破壊的変更になります

埋め込みの代わりに同じメソッドを書くのは面倒ですが、その分実装の詳細を外側から隠すことができます。
実装の詳細をそのメソッドが持つことで内部での変更がしやすくなります。
実装がすぐに見えるので、Listの詳細をさらに見に行く必要がなくなります。

## Avoid Using Built-in Names

Go の[言語仕様]( https://golang.org/ref/spec )ではいくつかのビルトイン、または[定義済み識別子]( https://golang.org/ref/spec#Predeclared_identifiers )があります。これらは Go のプログラム内で識別子として使うべきではありません。

状況にもよりますが、これらの識別子を再利用すると、元の識別子がレキシカルスコープ内で隠蔽されるか、元のコードを混乱させます。
コンパイラがエラーを出して気づく場合もありますが、最悪の場合は grep などでは発見困難な潜在的バグを起こす可能性があります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var error string
// `error` はビルトインの型を隠す

// or

func handleErrorMessage(error string) {
    // `error` はビルトインの型を隠す
}
```

</td><td>

```go
var errorMessage string
// `error` はビルトインの型のまま

// or

func handleErrorMessage(msg string) {
    // `error` はビルトインの型のまま
}
```

</td></tr>
<tr><td>

```go
type Foo struct {
    // これらのフィールドは
    // 技術的にはシャドウイングを引き起こしませんが、
    // `error` や `string` という文字列は曖昧です。
    error  error
    string string
}

func (f Foo) Error() error {
    // `error` と `f.error` は見た目が似ている
    return f.error
}

func (f Foo) String() string {
    // `string` と `f.string` は見た目が似ている
    return f.string
}
```

</td><td>

```go
type Foo struct {
    // `error` や `string` という文字列は明確に型名を指します
    err error
    str string
}

func (f Foo) Error() error {
    return f.err
}

func (f Foo) String() string {
    return f.str
}
```

</td></tr>
</tbody></table>

宣言済み識別子名をローカルの識別子に使ってもコンパイラはエラーを出さないことに注意してください。ですが `go vet` などのツールはこれらのシャドウイングを正しく見つけることができます。

## Avoid `init()`

できるなら `init()` を使うのは避けましょう。次のケースの場合は避けようがなかったり、推奨されます。

- 実行環境や呼び出し方に関係なく、決定的である場合
- 他の `init()` 関数の実行順や副作用に影響されない場合。`init()` の順序はよく知られていますが、コードが変更され、 `init()` 関数間の関係によってはコードが脆弱になり、エラーが発生しやすくなります
- グローバルな情報や、環境変数、ワーキングディレクトリ、プログラムの引数や入力にアクセスしたり、操作しない場合
- ファイルシステム、ネットワーク、システムコールの操作をしない場合

コードがこれらの要件を満たせない場合、 `main()` 関数か、プログラムのライフサイクルの一部でヘルパー関数として呼び出すか、`main()` 関数で直接呼び出す必要があります。
特にライブラリなど他のプログラムで使われることを想定したコードの場合は特に決定的であることに注意し、 "init magic" を引き起こさないように注意しましょう。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Foo struct {
    // ...
}

var _defaultFoo Foo

func init() {
    _defaultFoo = Foo{
        // ...
    }
}
```

</td><td>

```go
var _defaultFoo = Foo{
    // ...
}

// もしくはテスト可能性に配慮してこのようにします

var _defaultFoo = defaultFoo()

func defaultFoo() Foo {
    return Foo{
        // ...
    }
}
```

</td></tr>
<tr><td>

```go
type Config struct {
    // ...
}

var _config Config

func init() {
    // 悪い例: 現在のディレクトリに依存している
    cwd, _ := os.Getwd()

    // 悪い例: I/O
    raw, _ := os.ReadFile(
        path.Join(cwd, "config", "config.yaml"),
    )

    yaml.Unmarshal(raw, &_config)
}
```

</td><td>

```go
type Config struct {
    // ...
}

func loadConfig() Config {
    cwd, err := os.Getwd()
    // handle err

    raw, err := os.ReadFile(
        path.Join(cwd, "config", "config.yaml"),
    )
    // handle err

    var config Config
    yaml.Unmarshal(raw, &config)

    return config
}
```

</td></tr>
</tbody></table>

これらを考慮すると、次のような状況では `init()` が望ましかったり必要になる可能性があります。

- ただの代入では表現できない複雑な式
- `database/sql` の登録や、エンコーディングタイプの登録など、プラグイン的に使うフック
- [Google Cloud Functions]( https://cloud.google.com/functions/docs/bestpractices/tips#use_global_variables_to_reuse_objects_in_future_invocations ) などの決定的事前処理の最適化

## Exit in Main

Go のプログラムでは即時終了するために [`os.Exit`]( https://golang.org/pkg/os/#Exit ) や [`log.Fatal*`]( https://golang.org/pkg/log/#Fatal ) を使います。`panic()` を使うのは良い方法ではありません [don't panic](#dont-panic)を読んでください。
`os.Exit` や `log.Fatal*` を読んでいいのは `main()` 関数だけです。他の関数ではエラーを返して失敗を通知しましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func main() {
  body := readFile(path)
  fmt.Println(body)
}

func readFile(path string) string {
  f, err := os.Open(path)
  if err != nil {
    log.Fatal(err)
  }

  b, err := io.ReadAll(f)
  if err != nil {
    log.Fatal(err)
  }

  return string(b)
}
```

</td><td>

```go
func main() {
  body, err := readFile(path)
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println(body)
}

func readFile(path string) (string, error) {
  f, err := os.Open(path)
  if err != nil {
    return "", err
  }

  b, err := io.ReadAll(f)
  if err != nil {
    return "", err
  }

  return string(b), nil
}
```

</td></tr>
</tbody></table>

根拠: `exit` する関数が複数あるプログラムにはいくつかの問題があります。

- 明確でない制御フロー: どの関数もプログラムを強制終了できるので、制御フローを推論するのが難しくなります。
- テストが難しくなる: 強制終了するプログラムはテストで呼び出されたときも終了します。これはその関数をテストするのも難しくなりますし、`go test` でテストされるはずだった他のテストがスキップされる危険もあります。
- 後処理のスキップ: プログラムが強制終了されたとき、`defer` で終了時に実行する予定だった関数がスキップされます。これは重要な後処理をスキップする危険があります。

### Exit Once
可能なら、 `os.Exit` か `log.Fatal` を `main()` 関数で **一度だけ** 呼ぶのが好ましいです。もしプログラムを停止する失敗のシナリオがいくつかある場合、そのロジックは別の関数にしてエラーを返しましょう。

こうすると、 `main()` 関数を短くすることができますし、重要なビジネスロジックを分離してテストしやすくすることができます。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
package main

func main() {
  args := os.Args[1:]
  if len(args) != 1 {
    log.Fatal("missing file")
  }
  name := args[0]

  f, err := os.Open(name)
  if err != nil {
    log.Fatal(err)
  }
  defer f.Close()

  // もしここで log.Fatal を呼ぶと、
  // f.Close は実行されません。

  b, err := io.ReadAll(f)
  if err != nil {
    log.Fatal(err)
  }

  // ...
}
```

</td><td>

```go
package main

func main() {
  if err := run(); err != nil {
    log.Fatal(err)
  }
}

func run() error {
  args := os.Args[1:]
  if len(args) != 1 {
    return errors.New("missing file")
  }
  name := args[0]

  f, err := os.Open(name)
  if err != nil {
    return err
  }
  defer f.Close()

  b, err := io.ReadAll(f)
  if err != nil {
    return err
  }

  // ...
}
```

</td></tr>
</tbody></table>

## Use field tags in marshaled structs
JSON や YAML あるいは他のタグを使ったフィールド名をサポートするフォーマットに変換する場合、関連するタグを使ってアノテーションをつけましょう。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Stock struct {
  Price int
  Name  string
}

bytes, err := json.Marshal(Stock{
  Price: 137,
  Name:  "UBER",
})
```

</td><td>

```go
type Stock struct {
  Price int    `json:"price"`
  Name  string `json:"name"`
  // 安全に Name フィールドを name に変換できる
}

bytes, err := json.Marshal(Stock{
  Price: 137,
  Name:  "UBER",
})
```

</td></tr>
</tbody></table>

根拠: 構造体をシリアライズした形式は異なるシステムをつなぐ約束事です。
フィールド名を含む構造体のシリアライズした形式が変わってしまうと、この約束事が破れてしまいます。
タグを使ってフィールド名を指定するとこの約束事がより厳密になり、リファクタリングやフィールドのリネームで不意に壊れてしまうことを防ぐことができます。

## Don't fire-and-forget goroutines

ゴルーチンは軽量ですが、コストはかかります。少なくとも、スタックのメモリとスケジュールされたCPUを使います。
典型的な使い方をする限りコストは小さいですが、ライフタイムを考えずに大量に作り出すと大きなパフォーマンス問題を引き起こします。
ライフタイムが管理されてないゴルーチンは、ガベージコレクションの邪魔になったり、使用されなくなったリソースを保持し続けるなどの問題も引き起こす可能性があります。

なので、絶対に本番コードでゴルーチンをリークさせないようにしましょう。 [go.uber.org/goleak]( https://pkg.go.dev/go.uber.org/goleak ) でゴルーチンを使うところでリークさせてないかテストしましょう。

一般的に、全てのゴルーチンはどちらかの方法を持っている必要があります。

- 停止する時間が予測できる
- 停止すべきゴルーチンに通知する方法がある

どちらのケースでも、処理をブロックしてゴルーチンの終了を待つコードも無ければいけません。


例:
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
go func() {
  for {
    flush()
    time.Sleep(delay)
  }
}()
```

</td><td>

```go
var (
  stop = make(chan struct{}) // ゴルーチンに停止を伝える
  done = make(chan struct{}) // ゴルーチンが停止したことを伝える
)
go func() {
  defer close(done)

  ticker := time.NewTicker(delay)
  defer ticker.Stop()
  for {
    select {
    case <-ticker.C:
      flush()
    case <-stop:
      return
    }
  }
}()

// Elsewhere...
close(stop)  // ゴルーチンに停止シグナルを送る
<-done       // そしてゴルーチンが止まるのを待つ
```

</td></tr>
<tr><td>

ゴルーチンを止める方法は無い。プログラムが終了するまで残り続ける

</td><td>

ゴルーチンは `close(stop)` で停止できる。そして `<-done` で待つこともできる。

</td></tr>
</tbody></table>

### Wait for goroutines to exit

システムによって起動されたゴルーチンが与えられた場合、ゴルーチンの終了を待つ方法を用意する必要があります。次の2つがよく使われる方法です。

* `sync.WaitGroup` を使います。終了を待つべきゴルーチンが複数ある場合、こちらを使う

```go
var wg sync.WaitGroup
for i := 0; i < N; i++ {
  wg.Add(1)
  go func() {
    defer wg.Done()
    // ...
  }()
}

// 全ての終了を待つ
wg.Wait()
```

* 別の `chan struct{}` を作り、ゴルーチンが終了したとき `close` します。待つべきゴルーチンが1つだけのときはこちらを使いましょう

```go
done := make(chan struct{})
go func() {
  defer close(done)
  // ...
}()

// ここでゴルーチンの終了を待つ
<-done
```

### No goroutines in `init()`

`init()` 関数ではゴルーチンを起動するのを避けましょう。 [Avoid init](#avoid-init) も参照してください。

もしパッケージがバックグラウンドで動くゴルーチンを作る必要があるなら、ゴルーチンのライフタイムを管理するオブジェクトを作りそれを公開しましょう。
そのオブジェクトは `Close`、`Stop`、`Shutdown` など、バックグラウンドのゴルーチンを停止し、その終了を待つメソッドを提供しましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func init() {
  go doWork()
}

func doWork() {
  for {
    // ...
  }
}
```

</td><td>

```go
type Worker struct{ /* ... */ }

func NewWorker(...) *Worker {
  w := &Worker{
    stop: make(chan struct{}),
    done: make(chan struct{}),
    // ...
  }
  go w.doWork()
}

func (w *Worker) doWork() {
  defer close(w.done)
  for {
    // ...
    case <-w.stop:
      return
  }
}

// Shutdown はワーカーに終了を伝え、
// ワーカーが終了するのを待ちます。
func (w *Worker) Shutdown() {
  close(w.stop)
  <-w.done
}
```

</td></tr>
<tr><td>

ユーザーがこのパッケージを公開すると、ゴルーチンを無条件に作成し、止める方法はありません。

</td><td>

ユーザーがリクエストすると、ワーカーを起動します。`Shutdown` メソッドで、ワーカーを停止し使っているリソースを開放できる手段も提供しています。

[Wait for goroutines to exit]( #wait-for-gorutines-to-exit ) でも話したように、ワーカーが複数ゴルーチンを使う場合は `WaitGroup` を使いましょう。

</td></tr>
</tbody></table>


# Performance
パフォーマンスガイドラインは特によく実行される箇所にのみ適用されます。

## Prefer strconv over fmt
数字と文字列を単に変換する場合、`fmt` パッケージよりも `strconv` パッケージのほうが高速に実行されます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for i := 0; i < b.N; i++ {
  s := fmt.Sprint(rand.Int())
}
```

</td><td>

```go
for i := 0; i < b.N; i++ {
  s := strconv.Itoa(rand.Int())
}
```

</td></tr>
<tr><td>

```
BenchmarkFmtSprint-4    143 ns/op    2 allocs/op
```

</td><td>

```
BenchmarkStrconv-4    64.2 ns/op    1 allocs/op
```

</td></tr>
</tbody></table>

## Avoid string-to-byte conversion
固定の文字列からバイト列を何度も生成するのは避けましょう。
代わりに変数に格納してそれを使うようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for i := 0; i < b.N; i++ {
  w.Write([]byte("Hello world"))
}
```

</td><td>

```go
data := []byte("Hello world")
for i := 0; i < b.N; i++ {
  w.Write(data)
}
```

</tr>
<tr><td>

```
BenchmarkBad-4   50000000   22.2 ns/op
```

</td><td>

```
BenchmarkGood-4  500000000   3.25 ns/op
```

</td></tr>
</tbody></table>

## Prefer Specifying Map Capacity Hints
スライスやmapの容量のヒントが事前にある場合は初期化時に次のように設定しましょう。

```go
make(map[T1]T2, hint)
```

`make()` の引数にキャパシティを渡すと、初期化時に適切なサイズにしようとします。
なので要素をマップに追加する際にアロケーションの回数を減らすことができます。
ただし、キャパシティのヒントは必ずしも保証されるものではありません。
もし事前にキャパシティを渡していても、要素の追加時にアロケーションが発生する場合もあります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
m := make(map[string]os.FileInfo)
files, _ := ioutil.ReadDir("./files")
for _, f := range files {
    m[f.Name()] = f
}
```

</td><td>

```go
files, _ := ioutil.ReadDir("./files")
m := make(map[string]os.FileInfo, len(files))
for _, f := range files {
    m[f.Name()] = f
}
```

</td></tr>
<tr><td>

`m` は初期化時にサイズのヒントが与えられませんでした。
そのため余計にアロケーションが発生する可能性があります。

</td><td>

`m` にはサイズのヒントが与えられています。
要素の追加時にアロケーションの回数を押さえられます。

</td></tr>
</tbody></table>

# Style

## Avoid overly long lines
横にスクロールしたり、たくさん首をふるような長過ぎるコードは避けましょう。

横幅は99文字を推奨しています。書く側はこれを超えると改行したほうが良いですが、絶対ではありません。
コードが超えても問題ありません。

## Be Consistent
このガイドラインの一部は客観的に評価することができます。
ですが状況や文脈に依存する主観的なものもあります。

ですが何よりも重要なことは一貫性を保つことです。

一貫性のあるコードは保守しやすく、説明しやすく、読むときのオーバーヘッドも減らせます。
更に新しい規則やバグへの修正が非常に簡単になります。

逆に、1つのコードベース内に複数の異なったりバッティングしているスタイルがあると、メンテナンスのオーバーヘッドや、不確実なコード、認知的不協和が発生します。
これらの全てが開発速度の低下、苦痛なコードレビューを誘発し、更にバグを発生させます。

このガイドラインにある項目を自分たちのコードに適用する場合、パッケージもしくは更に大きな単位で適用することを勧めます。
サブパッケージレベルで適用することは同じコードベースに複数のスタイルを当てはめることになるため、先程述べた悪いパターンに当てはまっています。

## Group Similar Declarations
Go は似たような宣言をグループにまとめることができます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import "a"
import "b"
```

</td><td>

```go
import (
  "a"
  "b"
)
```

</td></tr>
</tbody></table>

これはパッケージ定数やパッケージ変数、型定義などにも利用できます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go

const a = 1
const b = 2



var a = 1
var b = 2



type Area float64
type Volume float64
```

</td><td>

```go
const (
  a = 1
  b = 2
)

var (
  a = 1
  b = 2
)

type (
  Area float64
  Volume float64
)
```

</td></tr>
</tbody></table>

関係が近いものだけをグループ化しましょう。
関係ないものまでグループ化するのは避けましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
  ENV_VAR = "MY_ENV"
)
```

</td><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)

const ENV_VAR = "MY_ENV"
```

</td></tr>
</tbody></table>

グループ化を使う場所に制限は無いので、関数内でも使うことができます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func f() string {
  var red = color.New(0xff0000)
  var green = color.New(0x00ff00)
  var blue = color.New(0x0000ff)

  ...
}
```

</td><td>

```go
func f() string {
  var (
    red   = color.New(0xff0000)
    green = color.New(0x00ff00)
    blue  = color.New(0x0000ff)
  )

  ...
}
```

</td></tr>
</tbody></table>

## Import Group Ordering
import のグループは次の2つに分けるべきです。

1. 標準パッケージ
2. それ以外

goimports がデフォルトで適用してくれます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import (
  "fmt"
  "os"
  "go.uber.org/atomic"
  "golang.org/x/sync/errgroup"
)
```

</td><td>

```go
import (
  "fmt"
  "os"

  "go.uber.org/atomic"
  "golang.org/x/sync/errgroup"
)
```

</td></tr>
</tbody></table>

## Package Names
パッケージ名をつける場合、以下のルールに従いましょう。

* 全て小文字で大文字やアンダースコアを使わない
* ほとんどの呼び出し側が名前付きインポートをする必要がないようにする
* 短く簡潔にすること。全ての呼び出し側で識別されることを意識してください
* 複数形にしないこと。`net/urls`ではなく`net/url`です
* "common"、"util"、"shared"、"lib"などを使わないこと。これらはなんの情報もない名前です。

[Package Names]( https://blog.golang.org/package-names )や[Style guideline for Go packages]( https://rakyll.org/style-packages/ )も参考にしてください。

## Function Names
関数名にはGoコミュニティの規則である[MixedCaps]( https://golang.org/doc/effective_go.html#mixed-caps )に従います。
例外はテスト関数です。
`TestMyFunction_WhatIsBeingTested`のようにテストの目的を示すためにアンダースコアを使って分割します。

## Import Aliasing
インポートエイリアスはパッケージ名とパッケージパスの末尾が一致していない場合に利用します。

```go
import (
  "net/http"

  client "example.com/client-go"
  trace "example.com/trace/v2"
)
```

他にもインポートするパッケージの名前がバッティングした場合には使います。
それ以外の場合は使わないようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import (
  "fmt"
  "os"


  nettrace "golang.net/x/trace"
)
```

</td><td>

```go
import (
  "fmt"
  "os"
  "runtime/trace"

  nettrace "golang.net/x/trace"
)
```

</td></tr>
</tbody></table>

## Function Grouping and Ordering

* 関数は呼び出される順番におおまかにソートされるべきです
* 関数はレシーバーごとにまとめられているべきです。

なので、`struct`、`const`、`var`の次にパッケージ外に公開されている関数が来るべきです。

`newXYZ()`や`NewXYZ()`は型が定義されたあと、他のメソッドの前に定義されている必要があります。

関数はレシーバーごとにまとめられているので、ユーティリティな関数はファイルの最後の方に出てくるはずです。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func (s *something) Cost() {
  return calcCost(s.weights)
}

type something struct{ ... }

func calcCost(n []int) int {...}

func (s *something) Stop() {...}

func newSomething() *something {
    return &something{}
}
```

</td><td>

```go
type something struct{ ... }

func newSomething() *something {
    return &something{}
}

func (s *something) Cost() {
  return calcCost(s.weights)
}

func (s *something) Stop() {...}

func calcCost(n []int) int {...}
```

</td></tr>
</tbody></table>

## Reduce Nesting
エラーや特殊ケースなどは早めにハンドリングして`return`したりループ内では`continue`や`break`してネストが浅いコードを目指しましょう。
ネストが深いコードを減らしていきましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for _, v := range data {
  if v.F1 == 1 {
    v = process(v)
    if err := v.Call(); err == nil {
      v.Send()
    } else {
      return err
    }
  } else {
    log.Printf("Invalid v: %v", v)
  }
}
```

</td><td>

```go
for _, v := range data {
  if v.F1 != 1 {
    log.Printf("Invalid v: %v", v)
    continue
  }

  v = process(v)
  if err := v.Call(); err != nil {
    return err
  }
  v.Send()
}
```

</td></tr>
</tbody></table>

## Unnecessary Else
if-else のどちらでも変数に代入する場合、条件に一致した場合に上書きするようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var a int
if b {
  a = 100
} else {
  a = 10
}
```

</td><td>

```go
a := 10
if b {
  a = 100
}
```

</td></tr>
</tbody></table>

## Top-level Variable Declarations
パッケージ変数で式と同じ型なら型名を指定しないようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var _s string = F()

func F() string { return "A" }
```

</td><td>

```go
var _s = F()
// Since F already states that it returns a string, we don't need to specify
// the type again.

func F() string { return "A" }
```

</td></tr>
</tbody></table>

式の型と合わない場合は明示するようにしましょう。

```go
type myError struct{}

func (myError) Error() string { return "error" }

func F() myError { return myError{} }

var _e error = F()
// F は myError 型を返すが私達は error 型が欲しい
```

## Prefix Unexported Globals with _
公開されてないtop-levelの`var`や`const`の名前には最初にアンダースコアをつけることでより内部向けてあることが明確になります。
ただ`err`で始まる変数名は例外です。
理由としてtop-levelの変数のスコープはそのパッケージ全体です。一般的な名前を使うと別のファイルで間違った値を使ってしまうことになります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// foo.go

const (
  defaultPort = 8080
  defaultUser = "user"
)

// bar.go

func Bar() {
  defaultPort := 9090
  ...
  fmt.Println("Default port", defaultPort)

  // We will not see a compile error if the first line of
  // Bar() is deleted.
}
```

</td><td>

```go
// foo.go

const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

</td></tr>
</tbody></table>

## Embedding in Structs

埋め込まれた型は構造体の定義の最初に置くべきです。
また、通常のフィールドと区別するために1行開ける必要があります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Client struct {
  version int
  http.Client
}
```

</td><td>

```go
type Client struct {
  http.Client

  version int
}
```

</td></tr>
</tbody></table>

埋め込みは適切な方法で機能を追加、拡張できる具体的なメリットがある場合に使いましょう。
ユーザーに悪影響を与えずに行う必要があります。[Avoid Embedding Types in Public Structs]( #avoid-embedding-types-in-public-structs ) も参照しましょう。

例外: Mutex は埋め込むべきではありません。公開されないフィールドとして使いましょう。[Zero-value Mutexes are Valid]( #zero-value-mutexes-are-valid ) も参照しましょう。

埋め込みは次のことをすべきではありません。:

- 見た目や、手軽さを重視して使うこと
- 埋め込まれた型が使いにくくなること
- 埋め込まれた型のゼロ値に影響すること。もし埋め込まれた型が便利なゼロ値を持っているなら、埋め込まれた後にもそれが維持されるようにしなければいけません。
- 埋め込まれた副作用として関係ないメソッドやフィールドが公開されてしまうこと
- 公開してない型を公開してしまうこと
- 埋め込まれた型のコピーに影響が出ること
- 埋め込まれた型の API や、意味上の型が変わってしまうこと
- 非標準の形式で埋め込むこと
- 埋め込まれる型の実装の詳細を公開すること
- 型の内部を操作できるようにすること
- ユーザーが意図しない方法で内部関数の挙動を変えること

簡単にまとめると、きちんと意識して埋め込みましょうということになります。
使うべきかチェックする簡単な方法は、「埋め込みたい型の公開されているメソッドやフィールドは全て埋め込まれる型に直接追加する必要があるか？」です。
答えが、「いくつかはある」あるいは「ない」の場合は埋め込みは使わず、フィールドを使いましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type A struct {
    // 悪い例:
    // A.Lock() と A.Unlock() が
    // 使えますが、メリットはありません。
    // さらに、A の内部を操作できる
    // ようになってしまいます。
    sync.Mutex
}
```

</td><td>

```go
type countingWriteCloser struct {
    // 良い例:
    // Write() は特定の目的のために
    // 外側のレイヤーに提供されます。
    // そして、内部の型の Write() メソッドに
    // 役割を移譲しています。
    io.WriteCloser

    count int
}

func (w *countingWriteCloser) Write(bs []byte) (int, error) {
    w.count += len(bs)
    return w.WriteCloser.Write(bs)
}
```

</td></tr>
<tr><td>

```go
type Book struct {
    // 悪い例:
    // ポインタなのでゼロ値の便利さを消してしまう
    io.ReadWriter

    // other fields
}

// later

var b Book
b.Read(...)  // panic: nil pointer
b.String()   // panic: nil pointer
b.Write(...) // panic: nil pointer
```

</td><td>

```go
type Book struct {
    // 良い例:
    // 便利なゼロ値を持っている
    bytes.Buffer

    // other fields
}

// later

var b Book
b.Read(...)  // ok
b.String()   // ok
b.Write(...) // ok
```

</td></tr>
<tr><td>

```go
type Client struct {
    sync.Mutex
    sync.WaitGroup
    bytes.Buffer
    url.URL
}
```

</td><td>

```go
type Client struct {
    mtx sync.Mutex
    wg  sync.WaitGroup
    buf bytes.Buffer
    url url.URL
}
```

</td></tr>
</tbody></table>

## Local Variable Declarations
変数が明示的に設定される場合、`:=` 演算子を利用しましょう。
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var s = "foo"
```

</td><td>

```go
s := "foo"
```

</td></tr>
</tbody></table>

しかし空のスライスを宣言する場合は`var`キーワードを利用したほうがよいでしょう。[参考資料: Declearing Empty Slices]( https://github.com/golang/go/wiki/CodeReviewComments#declaring-empty-slices )

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func f(list []int) {
  filtered := []int{}
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

</td><td>

```go
func f(list []int) {
  var filtered []int
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

</td></tr>
</tbody></table>

## nil is a valid slice
`nil` は長さ0のスライスとして有効です。
つまり以下のものが有効です。

* 長さ0のスライスを返す代わりに`nil`を返す
    <table>
    <thead><tr><th>Bad</th><th>Good</th></tr></thead>
    <tbody>
    <tr><td>

    ```go
    if x == "" {
    return []int{}
    }
    ```

    </td><td>

    ```go
    if x == "" {
    return nil
    }
    ```

    </td></tr>
    </tbody></table>

* スライスが空かチェックするためには`nil`かチェックするのではなく`len(s) == 0`でチェックする 
  <table>
  <thead><tr><th>Bad</th><th>Good</th></tr></thead>
  <tbody>
  <tr><td>

  ```go
  func isEmpty(s []string) bool {
    return s == nil
  }
  ```

  </td><td>

  ```go
  func isEmpty(s []string) bool {
    return len(s) == 0
  }
  ```

  </td></tr>
  </tbody></table>

* varで宣言しただけのゼロ値が有効
  <table>
  <thead><tr><th>Bad</th><th>Good</th></tr></thead>
  <tbody>
  <tr><td>

  ```go
  nums := []int{}
  // or, nums := make([]int)

  if add1 {
    nums = append(nums, 1)
  }

  if add2 {
    nums = append(nums, 2)
  }
  ```

  </td><td>

  ```go
  var nums []int

  if add1 {
    nums = append(nums, 1)
  }

  if add2 {
    nums = append(nums, 2)
  }
  ```

  </td></tr>
  </tbody></table>

## Reduce Scope of Variables
できる限り変数のスコープを減らしましょう。ただネストを浅くすることとバランスを考えてください。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
err := ioutil.WriteFile(name, data, 0644)
if err != nil {
 return err
}
```

</td><td>

```go
if err := ioutil.WriteFile(name, data, 0644); err != nil {
 return err
}
```

</td></tr>
</tbody></table>

もし関数の戻り値をifの外で利用する場合、あまりスコープを縮めようとしなくてもよいでしょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
if data, err := ioutil.ReadFile(name); err == nil {
  err = cfg.Decode(data)
  if err != nil {
    return err
  }

  fmt.Println(cfg)
  return nil
} else {
  return err
}
```

</td><td>

```go
data, err := ioutil.ReadFile(name)
if err != nil {
   return err
}

if err := cfg.Decode(data); err != nil {
  return err
}

fmt.Println(cfg)
return nil
```

</td></tr>
</tbody></table>

## Avoid Naked Parameters
値をそのまま関数の引数に入れることは可読性を損ないます。
もし分かりづらいならC言語スタイルのコメントで読みやすくしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true, true)
```

</td><td>

```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true /* isLocal */, true /* done */)
```

</td></tr>
</tbody></table>

よりよいのはただの`bool`を自作の型で置き換えることです。こうすると型安全ですし可読性も上がります。
更に将来的にtrue/false以外の状態も利用可能に修正することもできます。

```go
type Region int

const (
  UnknownRegion Region = iota
  Local
)

type Status int

const (
  StatusReady = iota + 1
  StatusDone
  // Maybe we will have a StatusInProgress in the future.
)

func printInfo(name string, region Region, status Status)
```

## Use Raw String Literals to Avoid Escaping
Goは複数行や引用符のために[`Raw string literal`]( https://golang.org/ref/spec#raw_string_lit )をサポートしています。
これらをうまく使って手動でエスケープした読みづらい文字列を避けてください。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
wantError := "unknown name:\"test\""
```

</td><td>

```go
wantError := `unknown error:"test"`
```

</td></tr>
</tbody></table>

## Initializing Structs

### Use Field Names to Initialize Structs

構造体を初期化する際にはフィールド名を書くようにしましょう。
[`go vet`]( https://golang.org/cmd/vet/ )でこのルールは指摘されます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
k := User{"John", "Doe", true}
```

</td><td>

```go
k := User{
    FirstName: "John",
    LastName: "Doe",
    Admin: true,
}
```

</td></tr>
</tbody></table>

例外としてフィールド数が3以下のテストケースなら省略してもよいです。

```go
tests := []struct{
  op Operation
  want string
}{
  {Add, "add"},
  {Subtract, "subtract"},
}
```

### Omit Zero Value Fields in Structs
フィールド名を使って構造体を初期化するときは、意味のあるコンテキストを提供しない場合はフィールド名を省略しましょう。Goが自動的に型に応じたゼロ値を設定してくれます

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
user := User{
  FirstName: "John",
  LastName: "Doe",
  MiddleName: "",
  Admin: false,
}
```

</td><td>

```go
user := User{
  FirstName: "John",
  LastName: "Doe",
}
```

</td></tr>
</tbody></table>

省略されたフィールドはデフォルトのゼロ値持つことは読み手の負荷を下げることができます。デフォルト値ではない値だけが指定されているからです。

ゼロ値をあえてセットする意味がある場合もあります。例えば、[Test Tables]( #test-tables ) で指定するテストケースではゼロ値でも設定することは役に立ちます。

### Use `var` for Zero Value Structs

全てのフィールドを省略して宣言するときは、 `var` を使って構造体の宣言をしましょう。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
user := User{}
```

</td><td>

```go
var user User
```

</td></tr>
</tbody></table>

[map initialization]( #initializing-maps )でも似たようなことをしていますが、この方法を使うと、ゼロ値だけの構造体の宣言と、フィールドに値を指定する宣言を区別することができます。そしてこの方法は[declare empty slices]( #declaring-empty-slices )と同じ理由でおすすめの方法です。

### Initializing Struct References
構造体の初期化と同じように構造体のポインタを初期化するときは`new(T)`ではなく、`&T{}`を使いましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
sval := T{Name: "foo"}

// inconsistent
sptr := new(T)
sptr.Name = "bar"
```

</td><td>

```go
sval := T{Name: "foo"}

sptr := &T{Name: "bar"}
```

</td></tr>
</tbody></table>

## Initializing Maps
空のマップを作る場合は `make(...)` を使い、コード内で実際にデータを入れます。
こうすることで、変数宣言と視覚的に区別でき、後でサイズヒントをつけやすくなります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var (
  // m1 is safe to read and write;
  // m2 will panic on writes.
  m1 = map[T1]T2{}
  m2 map[T1]T2
)
```

</td><td>

```go
var (
  // m1 is safe to read and write;
  // m2 will panic on writes.
  m1 = make(map[T1]T2)
  m2 map[T1]T2
)
```

</td></tr>
<tr><td>

変数宣言と初期化が視覚的に似ている

</td><td>

変数宣言と初期化が視覚的に区別しやすい

</td></tr>
</tbody></table>

可能なら `make()` でマップを初期化する際にキャパシティのヒントを渡しましょう。
詳細は[Prefer Specifying Map Capacity Hints]( #prefer-specifying-map-capacity-hints )を参照してください。

一方で、マップが予め決まった要素だけを保つ場合にはリテラルを使って初期化するほうがよいでしょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
m := make(map[T1]T2, 3)
m[k1] = v1
m[k2] = v2
m[k3] = v3
```

</td><td>

```go
m := map[T1]T2{
  k1: v1,
  k2: v2,
  k3: v3,
}
```

</td></tr>
</tbody></table>

大まかな原則は初期化時に決まった要素を追加するならマップリテラルを使い、それ以外なら `make()` (とあるならキャパシティのヒント)を使いましょう。

## Format Strings outside Printf
フォーマット用の文字列を`Printf`スタイルの外で定義する場合は`const`を使いましょう。
こうすることで`go vet`などの静的解析ツールでチェックしやすくなります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
msg := "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

</td><td>

```go
const msg = "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

</td></tr>
</tbody></table>

## Naming Printf-style Functions
`Printf`スタイルの関数を使う場合、`go vet`がフォーマットをチェックできるか確認しましょう。

これは可能であれば`Printf`スタイルの関数名を使う必要があることを示しています。
`go vet`はデフォルトでこれらの関数をチェックします。

事前に定義された関数名を使わない場合、関数名の最後を`f`にしましょう。
例えば`Wrap`ではなく`Wrapf`にします。
`go vet`は特定の`Printf`スタイルのチェックができるようになっていますが、末尾が`f`である必要があります。

```shell
$ go vet -printfuncs=wrapf,statusf
```

[go vet: Printf family check]( https://kuzminva.wordpress.com/2017/11/07/go-vet-printf-family-check/ )を更に参照してください。

# Patterns
## Test Tables
[サブテスト]( https://blog.golang.org/subtests )を利用したテーブルドリブンテストでコアのテストロジックを繰り返すときにコードの重複を避けるようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func TestSplitHostPort(t *testing.T)

host, port, err := net.SplitHostPort("192.0.2.0:8000")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("192.0.2.0:http")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "http", port)

host, port, err = net.SplitHostPort(":8000")
require.NoError(t, err)
assert.Equal(t, "", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("1:8")
require.NoError(t, err)
assert.Equal(t, "1", host)
assert.Equal(t, "8", port)
```

</td><td>

```go
// func TestSplitHostPort(t *testing.T)

tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  {
    give:     "192.0.2.0:8000",
    wantHost: "192.0.2.0",
    wantPort: "8000",
  },
  {
    give:     "192.0.2.0:http",
    wantHost: "192.0.2.0",
    wantPort: "http",
  },
  {
    give:     ":8000",
    wantHost: "",
    wantPort: "8000",
  },
  {
    give:     "1:8",
    wantHost: "1",
    wantPort: "8",
  },
}

for _, tt := range tests {
  t.Run(tt.give, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.give)
    require.NoError(t, err)
    assert.Equal(t, tt.wantHost, host)
    assert.Equal(t, tt.wantPort, port)
  })
}
```

</td></tr>
</tbody></table>

テストテーブルを使うと、エラーメッセージへの情報の追加やテストケースの追加も簡単ですし、コードも少なくなります。

テストケースのルールはテストケースの構造体のスライス名が `tests` 、ループ内のそれぞれのテストケースの変数名が `tt` とします。
更に入力値と出力値をわかりやすくするために`give`や`want`などのプレフィックスをつけることを推奨しています。

```go
tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  // ...
}

for _, tt := range tests {
  // ...
}
```

## Avoid Unnecessary Complexity in Table Tests
テーブルテストで実施するテストの中で条件付きのアサーションや分岐ロジックがあると、可読性が低くなり保守がとても難しくなります。
テーブルテストでは `for` の中で、複雑なコードや条件分岐を入れるべきではありません。

テストが失敗してデバッグする必要があるとき、大きくて複雑なテーブルテストは可読性と保守性を大きく損ないます。

そのようなテーブルテストはいくつかのテーブルテストに分割するか、そもそも別のテスト関数に分けるのも良いでしょう。

いくつかの方針を紹介します。

- 振る舞いが小さくなるように意識する
- 条件付きのアサーションを避けて、テストの深さを最小にする
- テーブルのフィールドが全てのテストで使われているか確認する
- 全てのロジックが全てのテストケースで実行されるか確認する

ここでいう「テストの深さ」とは、「そのテストで、前のアサーションを保持する必要がある連続したアサーションの数」といえます。
循環複雑度に近いです。より薄いテストはよりアサーション間の関係が薄く、より重要な点はそれらのアサーションは条件付きになる可能性が低くなることです。

具体的に言うと、次のような状況だとテストを読むのが難しくなります。

- テーブルのフィールドによって条件分岐が複数ある。( `shouldError` や `expectCall` などのフィールドがあると注意です )
- 特定のモックの期待値のためにたくさんの `if` がある。( `shouldCallFoo` などのフィールドがあると怪しいです )
- テーブルの中に関数がある。(フィールドの中に `setupMocks func(*FooMock)` がある )

しかし、変更された入力に基づいてのみ変化する動作をテストする場合、比較可能なユニットを別々のテストに分割して比較しにくくするのではなく、すべての入力に対してどのように動作が変化するかをよりよく説明するために、同様のケースをまとめてテーブルテストとすることが望ましい場合があります。

テスト本体が短くてわかりやすければ、成功ケースと失敗ケースの分岐経路を1つにして、`shouldErr` のようなフィールドでエラーを期待することもできます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func TestComplicatedTable(t *testing.T) {
  tests := []struct {
    give          string
    want          string
    wantErr       error
    shouldCallX   bool
    shouldCallY   bool
    giveXResponse string
    giveXErr      error
    giveYResponse string
    giveYErr      error
  }{
    // ...
  }

  for _, tt := range tests {
    t.Run(tt.give, func(t *testing.T) {
      // setup mocks
      ctrl := gomock.NewController(t)
      xMock := xmock.NewMockX(ctrl)
      if tt.shouldCallX {
        xMock.EXPECT().Call().Return(
          tt.giveXResponse, tt.giveXErr,
        )
      }
      yMock := ymock.NewMockY(ctrl)
      if tt.shouldCallY {
        yMock.EXPECT().Call().Return(
          tt.giveYResponse, tt.giveYErr,
        )
      }

      got, err := DoComplexThing(tt.give, xMock, yMock)

      // verify results
      if tt.wantErr != nil {
        require.EqualError(t, err, tt.wantErr)
        return
      }
      require.NoError(t, err)
      assert.Equal(t, want, got)
    })
  }
}
```

</td><td>

```go
func TestShouldCallX(t *testing.T) {
  // setup mocks
  ctrl := gomock.NewController(t)
  xMock := xmock.NewMockX(ctrl)
  xMock.EXPECT().Call().Return("XResponse", nil)

  yMock := ymock.NewMockY(ctrl)

  got, err := DoComplexThing("inputX", xMock, yMock)

  require.NoError(t, err)
  assert.Equal(t, "want", got)
}

func TestShouldCallYAndFail(t *testing.T) {
  // setup mocks
  ctrl := gomock.NewController(t)
  xMock := xmock.NewMockX(ctrl)

  yMock := ymock.NewMockY(ctrl)
  yMock.EXPECT().Call().Return("YResponse", nil)

  _, err := DoComplexThing("inputY", xMock, yMock)
  assert.EqualError(t, err, "Y failed")
}
```

</td></tr>
</tbody></table>

このコードの複雑さは変更したり、理解したり、このテストが正しいのかを証明するのが困難になります。

厳密なガイドラインはありませんが、システムへの複数の入力がある場合は、可読性と保守性を常に考慮して、個別のテストかテーブルテストか決定しましょう。

### Parallel Tests

並列テストなどの特殊なループ(例えば、ループの一部としてゴルーチンを起動し参照を保持するもの)では、ループのスコープ内の変数が正しい値を保持しているか注意しましょう。

```go
tests := []struct{
  give string
  // ...
}{
  // ...
}

for _, tt := range tests {
  tt := tt // for t.Parallel
  t.Run(tt.give, func(t *testing.T) {
    t.Parallel()
    // ...
  })
}
```

この例では、 `t.Parallel()` を下で読んでいるので、 `tt` という変数をループの中で再度宣言する必要があります。
もしやらないと、ほとんどのテストで `tt` 変数が期待しない値になったり、テスト中に値が変更されてしまいます。

## Functional Options
Functional Option パターンは不透明なOption型を使って内部の構造体に情報を渡すパターンです。
可変長引数を受け取り、それらを順に内部のオプションに渡します。

コンストラクタや、公開されたAPIで3つ以上の多くの引数が必要な場合、このパターンを使うと良いでしょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// package db

func Open(
  addr string,
  cache bool,
  logger *zap.Logger
) (*Connection, error) {
  // ...
}
```

</td><td>

```go
// package db

type Option interface {
  // ...
}

func WithCache(c bool) Option {
  // ...
}

func WithLogger(log *zap.Logger) Option {
  // ...
}

// Open creates a connection.
func Open(
  addr string,
  opts ...Option,
) (*Connection, error) {
  // ...
}
```

</td></tr>
<tr><td>

キャッシュやロガーはデフォルトを使う場合でも常に指定する必要があります

```go
db.Open(addr, db.DefaultCache, zap.NewNop())
db.Open(addr, db.DefaultCache, log)
db.Open(addr, false /* cache */, zap.NewNop())
db.Open(addr, false /* cache */, log)
```

</td><td>

オプションは必要なら提供されます

```go
db.Open(addr)
db.Open(addr, db.WithLogger(log))
db.Open(addr, db.WithCache(false))
db.Open(
  addr,
  db.WithCache(false),
  db.WithLogger(log),
)
```

</td></tr>
</tbody></table>

私達が紹介する方法は非公開のメソッドを持った `Option` インタフェースを使う方法です。
指定されたオプションは非公開の `options` 構造体に保持されます。

```go
type options struct {
  cache  bool
  logger *zap.Logger
}

type Option interface {
  apply(*options)
}

type cacheOption bool

func (c cacheOption) apply(opts *options) {
  opts.cache = bool(c)
}

func WithCache(c bool) Option {
  return cacheOption(c)
}

type loggerOption struct {
  Log *zap.Logger
}

func (l loggerOption) apply(opts *options) {
  opts.logger = l.Log
}

func WithLogger(log *zap.Logger) Option {
  return loggerOption{Log: log}
}

// Open creates a connection.
func Open(
  addr string,
  opts ...Option,
) (*Connection, error) {
  options := options{
    cache:  defaultCache,
    logger: zap.NewNop(),
  }

  for _, o := range opts {
    o.apply(&options)
  }

  // ...
}
```

このパターンを実装するためにクロージャを使っていますが、この方法は開発者により柔軟性をもたせ、デバッグやテストをしやすくなると考えています。
特に、テストやモックなどで比較する際にクロージャを使うことで比較しやすくなります。
更に、`options` に他のインタフェースを実装させる事もできます。
`fmt.Stringer` インタフェースを実装すると、設定を人間にわかりやすく表示させることも可能です。

更に以下の資料が参考になります。

* [Self-referential functions and the design of options]( https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html )
* [Functional options for friendly APIs]( https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis )


# Linting
どんなおすすめの linter のセットよりも重要なのは、コードベース全体で一貫した linter を使うことです。

私達は最小限の linter として以下のものをおすすめしています。これだけあれば、一般的な問題を発見でき、不必要に厳しすぎることもなく、良い品質を確立することができるからです。

- [errcheck]( https://github.com/kisielk/errcheck ) はエラーが正しく処理されているかを担保します
- [goimports]( https://godoc.org/golang.org/x/tools/cmd/goimports ) はライブラリのインポートの管理もするフォーマッタです
- [golint]( https://github.com/golang/lint ) は一般的なスタイルのミスを見つけます
- [govet]( https://golang.org/cmd/vet/ ) は一般的なミスを見つけます
- [staticcheck]( https://staticcheck.io/ ) は多くの静的チェックを行います

## Lint Runners

私達は、大きなコードベースでのパフォーマンスと、多くの linter を一度に設定できる点から、[golangci-lint]( https://github.com/golangci/golangci-lint ) をおすすめしています。
このガイドのリポジトリにはおすすめの [.golangci.yaml]( https://github.com/uber-go/guide/blob/master/.golangci.yml ) 設定ファイルがあります。

golangci-lint は[多くのlinter]( https://golangci-lint.run/usage/linters/ )が使えます。前述した linter は基本のセットですが、チームが必要に応じて linter を追加することを推奨しています。

