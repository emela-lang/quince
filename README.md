# quince

Emela の HTTP API を，route の list として書く．route は method，パスの pattern，handler の 3 つで，handler は入力の型を 1 つ受け取り，出力の型を 1 つ返す．quince は request から入力を読み，handler を呼び，出力を JSON で返す．

```emela
import Std.Http
import Std.Http.Request.{Request, Post}
import Std.Http.Response.{Response}
import Quince.{Api, InvalidInput}

type TodoPath(id: String)
type CreateTodo(body: TodoBody)
type TodoBody(title: String)
type TodoView(id: String, title: String, done: Bool)

fn api() -> Api[TodoNotFound, { TodoStore }] {
  Quince.new()
    |> Quince.get("/todos/:id", show)
    |> Quince.route(Post, "/todos", 201, create)
}

fn show(input: TodoPath) -> TodoView fails TodoNotFound use TodoStore { ... }

fn on_error[R](run: fn() -> Response fails TodoNotFound use R) -> Response use R {
  run() escape {
    TodoNotFound(id: _) -> Quince.problem(404, "todo not found")
  }
}

@entry(Http.serve)
pub fn app(request: Request) -> Response {
  with KvTodos { Quince.handle(api(), request, on_error) }
}
```

層に分けた Todo API の全体は `examples/todo` にある．

## 入力

入力の型は record にする．フィールドの値の出どころは名前で決まる．

- pattern の引数（`:id`）と同じ名前のフィールドは，パスから読む
- `body` という名前のフィールドは，本文を JSON として読む
- それ以外のフィールドは query から読む．List のフィールドは同じ名前の値を全部集め，1 つもなければ空にする

パスと query の値は文字列で来る．quince は入力の型の Shape を見て，Int，Int64，Float のフィールドは数に，Bool のフィールドは `true` と `false` を Bool に直してから `Decode` に渡す．直せない値は文字列のまま渡すので，`Decode` がそのフィールドで失敗する．

入力の型には `Decode` と `Shaped` を，出力の型には `Encode` を要求する．

## 400

入力を読めなければ，handler を呼ばずに 400 を返す．`from_dynamic_all` で読むので，壊れたフィールドは全部並ぶ．

```json
{"errors":[{"field":"body.title","message":"expected text, found number"}],"status":400,"title":"Bad Request"}
```

- `content-type` は `application/problem+json`（RFC 9457）
- `field` はフィールドへの道で，`title`，`body.title`，`tags[1]` の形
- 本文が JSON でなければ，`field` は `body`

handler の中で domain の値に直すときは `Quince.field` を使う．`Broken`，`AllBroken`（x/shape の `Rule`）と `ParseError` を，そのフィールドの 400 にする．それ以外の error はそのまま `on_error` に届く．

```emela
fn create(input: CreateTodo) -> TodoView fails InvalidInput use TodoStore {
  title = Quince.field("title", fn() { Todo.title(input.body.title) })
  view(Todos.create(title))
}
```

## handler の error

handler が失敗しうる error（`InvalidInput` を除く）は `Api[E, R]` の E になり，`handle` に渡す `on_error` が受ける．`on_error` は `escape` で error を response に直すので，対応を書き忘れた error は compiler が見つける．error の少ない handler も，effect を使わない handler も，同じ `Api` に登録できる．

## quince が答えるもの

| request | response |
|---|---|
| どの route の pattern にも合わないパス | 404 |
| パスに合う route はあるが，method が違う | 405 と `allow` |
| HEAD | GET の route を走らせ，本文を空にする |
| OPTIONS | 204 と `allow` |
| 何も返さない handler（`route_empty`，`delete`） | 204 |

2 つの route が同じ request に合うときは，先に登録したほうを使う．

## 登録の確かめ

`api()` は request ごとに組み立て直されるので，pattern と入力の型が合っているかは request が来るまで判らない．`Quince.check(api())` をテストから呼ぶ．pattern の引数に対応するフィールドがない route を，1 行ずつ返す．

```emela
@test
fn the_routes_are_well_formed() {
  assert Quince.check(Todos.api()) == []
}
```

## まだないもの

- `derive Decode, Shaped`．compiler が導出できるのは Eq，Ord，Show だけなので，入力と出力の型の impl は今は手で書く（`examples/todo/src/web/todos.emel`）．std に移す proposal（emela-lang/emela#455）が入れば，型の宣言に `derive` を書くだけになる
- OpenAPI の生成．route は入力の型の Shape を持っているので，その上に作れる．出力の型にも `Shaped` を要求することになる
- 認証の結果のような，request ごとの値の受け渡し．今は effect と handler で書く
