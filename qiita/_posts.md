



OS X への Redis インストール
==================================================

homebrew 経由でインストールしました。

#### Install

```bash
$ brew search redis
hiredis	   redis      redis1310	 redis24
homebrew/nginx/redis2-nginx-module	       homebrew/php/php54-redis			      homebrew/php/php56-redis
homebrew/php/php53-redis		       homebrew/php/php55-redis

$ brew install redis
$ brew install hiredis
```


#### 起動

```bash
$ redis-server
[80855] 31 Oct 00:18:38.270 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
[80855] 31 Oct 00:18:38.272 * Increased maximum number of open files to 10032 (it was originally set to 2560).
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 2.8.17 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in stand alone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 80855
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[80855] 31 Oct 00:18:38.278 # Server started, Redis version 2.8.17
[80855] 31 Oct 00:18:38.279 * The server is now ready to accept connections on port 6379

```


`{ ... }` と `do ... end` の違いと使い分けるべきシーン
==================================================

ブロック付きメソッド呼び出しを記述する際には `{ ... }` あるいは `do ... end` を使うと思います。
しかしこれらは厳密にいうと異なります。
その違いは __結合強度__ です。

### `{ ... }` の場合

```ruby
some_method other_method { ... }
```

この式は次のように解釈されます。

```ruby
some_method(other_method{ ... })
```

### `do ... end` の場合

```ruby
some_method other_method do ... end
```

この式は次のように解釈されます。

```ruby
some_method(other_method) do ... end
```

## 使い分けるべきシーン

### `{ ... }` を使うべき
* メソッドの戻り値を利用する場合
* メソッドチェーン(a.foo{}.bar{}.baz{})をする場合
* 1行で記述できる場合

### `do ... end` を使うべき
* 上記以外の場合、基本こちらを使うべき。


## Ref.
* 初めてのRuby



expect change を複数データ同時にTESTしたいときに
==================================================

expect change を複数データ同時にTESTしたいときなどがあります。
そんな時には以下のようにするとできます。(例はあまり適切ではありません...)

```ruby
expect{ execute }.to change{ [User.find_by(id: 1).coin, User.find_by(id: 1).skill] }.from([100, 0]).to([0, 1])
```

### Ref.
* [Rspec expect to change record value](http://blog.houen.net/rspec-expect-change-record-value/)

