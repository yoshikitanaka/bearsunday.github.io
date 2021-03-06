---
layout: default_ja
title: BEAR.Sunday | ブログチュートリアル 記事表示ページの作成
category: Blog Tutorial
---

# 記事表示ページの作成

## ページリソース

### ページの作成

BEAR.Sundayで新しくページを作成する場合、通常次の２つを作成します。

 * ページリソースクラス
 * ページリソーステンプレート

ページリソースもアプリケーションリソースと同じ構成、インターフェイスを持ちます。アプリケーションリソースが標準の状態では何も持たず必要なDBオブジェクトをDIでインジェクトしたように、ページリソースも依存をインジェクトして使用します。

### ページコントローラー

MVCのコントローラにあたる部分はBEARではページリソースです。ページはWebのリクエストを受け取り、アプリケーションリソースをリクエストして、自らを構成します。ページはそのまま出力用のオブジェクトとしても扱われます。

Note: BEAR.Sundayではルーターも利用できますが、このブログアプリでは利用しません。

Note: BEAR.Sundayではサイトの１ページが１ページリソースクラスに相当し [Page Controller](http://www.martinfowler.com/eaaCatalog/pageController.html) の働きをします。

### ページクラス

この記事表示ページの役割は「アプリケーションAPIの記事リソースをGETリクエストで取得してページのpostsスロットにアサインする」ということです。

[アプリケーションリソース](blog_get.html) のセクションではコンソールからアプリケーションリソースを実行しましたが、PHPでリソースをリクエストするにはリソースクライアントを使います。リソースクライントはtraitでインジェクトすることが出来ます。

traitを使用する `use` 文で次のように記述するとリソースクライアントが `$resource` プロパティにインジェクトされます。

{% highlight php startinline %}
use ResourceInject;
{% endhighlight %}

インジェクトされたリソースクライアントを使ってリソースリクエストを行うにはこのようにします。

{% highlight php startinline %}
$this->resource->get->uri('app://self/blog/posts')->request()
{% endhighlight %}

まとめるとこうなります。

{% highlight php startinline %}
<?php

namespace Demo\Sandbox\Resource\Page\Blog;

use BEAR\Resource\ResourceObject;
use BEAR\Sunday\Inject\ResourceInject;
use BEAR\Sunday\Annotation\Cache;

class Posts extends ResourceObject
{
    use ResourceInject;

    /**
     * @Cache
     */
    public function onGet()
    {
        $this['posts'] = $this->resource->get->uri('app://self/blog/posts')->request();
        return $this;
    }
}
{% endhighlight %}

`app://self/blog/posts` リソースへのリクエストを自らのpostsというスロットに格納しています。

Note: `$this['posts']` は `$this->body['posts']` の省略した書き方のシンタックスシュガー（＝読み書きのしやすさのために導入される構文）です。

Note: MVCのコントローラーと違って、出力に関心が払われてないのに注目してみてください。テンプレートファイルの指定や、テンプレートに対しての変数のアサイン等がありません。

## リソースとしてのページ

それではページリソースをアプリケーションリソースと同じようにコンソールからアクセスしてみましょう。

```
$ php apps/Demo.Sandbox/bootstrap/contexts/api.php get page://self/blog/posts

200 OK
...
[BODY]
posts => Demo_Sandbox_Resource_App_Blog_Posts_76b99d827d509b164e02c644e3749f54RayAop(
...
```

`posts` というスロットに *get app://self/blog/posts* というリクエスト結果が格納されてます。（@TODO 説明が現状と合っていない。ここは、どう説明すべきか？）

ページリソースはページコントローラーの役割をするとともに出力用のオブジェクトの役割も果たしています。しかしどのように表現されるかにはまだ関心が払われていません。

## リソースキャッシュ

ページリソースには `@Cache` とアノテートされていて、Sandboxアプリケーションではこのアノテーションを持つメソッドにはキャシュインターセプターがバインドされています。例えば30秒間リソースをキャッシュしたいならこのように表記します。

{% highlight php startinline %}
use BEAR\Sunday\Annotation\Cache;

/**
 * @Cache(30)
 */
{% endhighlight %}

Note: CacheのFQNのために `use` 文が必要です。

## 無期限キャッシュ

このページリソースには時間が指定されていないので、リソースのGETリクエストは無期限にキャッシュされ `onGet` メソッドが実行されるのは最初の一回のみです。それでは記事が追加されたり削除されてもこの記事表示ページは変更されないのでしょうか？

この記事表示ページリソースの役割は、記事リソースをリクエストして `posts` にセットすることです。その役割はリクエストによらず不変でこの役割がキャッシュされます。

ページリソースにセットしているのはリクエストの結果ではなくて、リクエストそのものです。`@Cache` で無期限のキャッシュを指定してもキャッシュされた記事リソースリクエストは毎回実行され、記事リソースのリソース状態は反映されます（この場合、`@Cache` でセーブされるのは `onGet` メソッド内でリクエストを作るわずかなコストだけです）。

つまりこのキャッシュでカットしているのはリソースリクエストを組み立てるコストです。
