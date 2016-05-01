


http://crunchtimer.jp/blog/technology/aws/2234/
http://bayashi.net/wiki/linux/mysql/connections_threads
http://dev.mysql.com/doc/refman/5.5/en/processlist-table.html

processlist とはなにか




Amazon RDS な MySQL で 不要 process を kill する
==================================================

MySQL で `Too many connections` が発生しました。
`processlist` を確認したところ、Command が Sleep なプロセスが多く発生しており、
結果最大接続数の上限に達してしまうことで発生していたのでした。
Sleep なプロセスが接続を持ったまま残っていることは問題なので、これを削除することとします。


まずは processlist を表示します。

`show full processlist` を実行するか、`select * from information_schema.PROCESSLIST` を実行します。

COMMAND が 'Sleep' で、TIME が 1000 以上のものを抽出するには下記のようにします。

```
mysql> select * from information_schema.PROCESSLIST where USER='user' and COMMAND='Sleep' and TIME > 1000;
```

これらの process を一気に kill するためにコマンドを作ります。

```
mysql -uroot -hhostname testdb -p -e "select concat('KILL ', id, ';') from information_schema.PROCESSLIST where USER='mery_db_user' and Command='Sleep' and TIME > 1000;" > /tmp/a.txt
```

mysql 上で source するか、shell上で 流し込みます。

```
mysql> source /tmp/a.txt
```


ちなみに、Amazon RDS だと kill で process を kill できません...

```
mysql> kill 64890;
ERROR 1095 (HY000): You are not owner of thread 64890
```

そのような場合には `mysql.rds_kill` を使って kill します。

```
mysql> CALL mysql.rds_kill(66825542);
Query OK, 0 rows affected (0.00 sec)
```


`show full processlist` と `select * from information_schema.PROCESSLIST`

```
root@localhost[footest]:4> show full processlist ;
+-----+------+-----------+----------+---------+-------+-------+-----------------------+
| Id  | User | Host      | db       | Command | Time  | State | Info                  |
+-----+------+-----------+----------+---------+-------+-------+-----------------------+
| 774 | root | localhost | foo     | Sleep   |    55 |       | NULL                  |
| 775 | root | localhost | NULL     | Sleep   |    24 |       | NULL                  |
| 918 | root | localhost | foo     | Sleep   | 13242 |       | NULL                  |
| 933 | root | localhost | footest | Sleep   | 21712 |       | NULL                  |
| 965 | root | localhost | foo     | Sleep   | 21472 |       | NULL                  |
| 969 | root | localhost | footest | Query   |     0 | init  | show full processlist |
+-----+------+-----------+----------+---------+-------+-------+-----------------------+
6 rows in set (0.00 sec)

root@localhost[footest]:5> select * from information_schema.PROCESSLIST;
+-----+------+-----------+----------+---------+-------+-----------+----------------------------------------------+
| ID  | USER | HOST      | DB       | COMMAND | TIME  | STATE     | INFO                                         |
+-----+------+-----------+----------+---------+-------+-----------+----------------------------------------------+
| 965 | root | localhost | foo     | Sleep   | 21640 |           | NULL                                         |
| 918 | root | localhost | foo     | Sleep   | 13410 |           | NULL                                         |
| 969 | root | localhost | footest | Query   |     0 | executing | select * from information_schema.PROCESSLIST |
| 774 | root | localhost | foo     | Sleep   |    43 |           | NULL                                         |
| 775 | root | localhost | NULL     | Sleep   |     2 |           | NULL                                         |
| 933 | root | localhost | footest | Sleep   | 21880 |           | NULL                                         |
+-----+------+-----------+----------+---------+-------+-----------+----------------------------------------------+
6 rows in set (0.02 sec)
```


## Ref
* https://forums.aws.amazon.com/message.jspa?messageID=337243
* http://hsuzuki.hatenablog.com/entry/2014/05/13/105649
* http://stackoverflow.com/questions/1903838/how-do-i-kill-all-the-processes-in-mysql-show-processlist



配列における `<<` と `+` の違い
==================================================


`a = [1, 2, 3]` に `b = [4, 5, 6]` を `a << b`  とかで突っ込むと `b` は `a` の1要素として入ります。

つまり、`[1, 2, 3, [4, 5, 6]]` ですね。

これを平坦化したいときには `a.flatten!` とかやると `a` が破壊的に平坦化されます。


```ruby
[1, 2, 3, 4, 5, 6]
```

&nbsp;

しかし、最初からこれを得たいときには `+` のほうが処理が自然です。

`a + b` とかすることで得られます。

でもこれは破壊的メソッドとかがないのでこの値が欲しい時には代入して得る必要があります。


```ruby
c = a + b
```



&nbsp;

例えば配列 `a` に `b` を足してランダム化したものは


```ruby
(a + b).shuffle
```


とか


```ruby
(a << b).flatten!.sort_by{rand}
```


とかとすれば得ることができます。


MySQL の分離レベルとか自動コミットとか
==================================================


今回は分離レベルの話と自動コミットについて雑多なメモを。

&nbsp;
<h3>分離レベル</h3>
MySQL InnoDB のデフォルト分離レベルは REPEATABLE-READ になっています。

このモードはダーティーリードは禁止するものの、ファントムリードなどは発生するというやつで、
4つの分離レベルの中で2番目に厳しいトランザクション独立性を持ちます。
<ul>
  <li>READ UNCOMMITTED</li>
  <li>READ COMMITTED</li>
  <li>REPEATABLE READ</li>
  <li>SERIALIZABLE</li>
</ul>
でも僕は READ COMMITED を使うことが多いです。
理由はパフォーマンスです。

また、READ COMMITED の場合、コミットされたデータは別のトランザクションから参照可能です。
これにより、別トランザクション内でも無駄にクエリを発行することを防いだりすることができたりするメリットもあります。

&nbsp;
<h3>自動コミット</h3>
MySQL には自動コミットというのがあります。
これが有効の場合、更新系クエリは即座にコミットされてしまいます。
一方トランザクションを使うと、明示的に commit, rollback をしない限り更新が確定されません。

これは複数処理を一つのトランザクションとして all or nothing を実現したいときに便利です。
(Webアプリケーションを作成するときによく見かける方針だと思います。)

それで、これを命令するのが

start transaction, begin, set autocommit = 0

だったりします。

一つの起動プロセスの中で何度かトランザクション処理をするときには
AUTO COMMIT モードをOFFにしてしまったほうがらくだと思います。

そういう時には

```mysql
select @@autocommit;
set autocommit = 0
select @@autocommit;
```

とかで確認、設定してしまって、トランザクション処理を行うのがオススメです。

一方で一つのトランザクション処理しかしないよー、という場合には、
start transaction とか begin を使うのもいいと思います。

このへんはおこのみですね。

&nbsp;
<h3>Ref.</h3>
<ul>
  <li><a title="http://orangain.hatenablog.com/entry/django-mysql-transaction" href="http://orangain.hatenablog.com/entry/django-mysql-transaction" target="_blank">http://orangain.hatenablog.com/entry/django-mysql-transaction</a></li>
  <li><a title="http://d.hatena.ne.jp/fat47/20140212/1392171784" href="http://d.hatena.ne.jp/fat47/20140212/1392171784" target="_blank">http://d.hatena.ne.jp/fat47/20140212/1392171784</a></li>
  <li><a title="http://open-groove.net/mysql/autocommit/" href="http://open-groove.net/mysql/autocommit/" target="_blank">http://open-groove.net/mysql/autocommit/</a></li>
</ul>


**args は引数として期待していないハッシュが送られてきた時にとりあえず受け取ってくれるものらしい
==================================================


```ruby
def log(msg, level: "ERROR", time: Time.now)
  puts "#{ time.ctime } [#{ level }] #{ msg }"
end
```

みたいなメソッドがあった時に、正しくハッシュを渡してやると

```ruby
log('Hi!', level: 'ERROR', time: Time.now) #=> Thu Nov 13 01:42:07 2014 [ERROR] Hi!
```

みたいに返ってくるわけなのですが、これに対して余分なハッシュを送った場合、下記のように例外を吐きます。

```ruby
log('Hi!', level: 'ERROR', time: Time.now, date: Time.now) #=> ArgumentError: unknown keyword: date
```

もしこれが嫌なとき、つまり期待するハッシュ以外のハッシュがきても別にスルーしてほしい時などには `**` で受け取ってやるといいみたいです。

こんなかんじで。

```ruby
def log(msg, level: "ERROR", time: Time.now, **kwrest)
  puts "#{ time.ctime } [#{ level }] #{ msg }"
end
```

&nbsp;

そうすれば期待しないハッシュを送っても例外を吐きません。

```ruby
log('Hi!', level: 'ERROR', time: Time.now, date: Time.now) #=> Thu Nov 13 01:45:11 2014 [ERROR] Hi!
log('Hi!', date: Time.now) #=> Thu Nov 13 01:45:19 2014 [ERROR] Hi!
```

<h3>Ref.</h3>
<ul>
  <li><a href="http://magazine.rubyist.net/?0041-200Special-kwarg" target="_blank">るびま Ruby 2.0.0 のキーワード引数</a></li>
</ul>


binding.pry から脱出するコマンド
==================================================


binding.pry は debug の際に step 実行を行うことができ非常に便利なものです。
ただ脱出方法がいまいちわからず RSpec などで複数のテストケースを実行してしまう時など Ctrl + D の連打はつらいものがありました。
それでちょっと調べてみたらすぐにでてきました・・・。

```ruby
exit!
```

とかで脱出できるみたいです。
他にも `exit-program` や `disable-pry` などを使う人もいるみたいです。

### Ref.

* [How do I step out of a loop with Ruby Pry?](http://stackoverflow.com/questions/8015531/how-do-i-step-out-of-a-loop-with-ruby-pry)
* [RubyistならデバッグにはPryのbinding.pryがおすすめ](http://blog.livedoor.jp/sasata299/archives/51841232.html)
* [#280 Pry with Rails](http://railscasts.com/episodes/280-pry-with-rails)



Setup Octopress
==================================================

まだ実導入していないのですがとりあえずメモしておきます。

1- Octopress を取ってくる

```bash
git clone git://github.com/imathis/octopress.git octopress
cd octopress
```

2- rbenv 経由で使用したいバージョンのrubyをインストールし、localにセットした上で、bundler経由で依存gemのインストール

```bash
rbenv install 2.1.3
rbenv local 2.1.3
bundle install
```

3- デフォルトの Octopress テーマをインストール

```bash
$ bundle exec rake install
## Copying classic theme into ./source and ./sass
mkdir -p source
cp -r .themes/classic/source/. source
mkdir -p sass
cp -r .themes/classic/sass/. sass
mkdir -p source/_posts
mkdir -p public
```

### Ref.

* http://octopress.org/docs/setup/





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

