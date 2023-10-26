---
theme: default
highlighter: shiki
lineNumbers: false
info: |
  # Rails Executor: the border between application and framework code

  Do you know how Ruby on Rails cleans up caches and resources between requests and background jobs and how it knows when and which code to reload?

  [Rails Executor](https://guides.rubyonrails.org/threading_and_code_execution.html#executor) is a little-known Rails internal detail a typical application developer will never notice or need to know about. However, if you create gems that call an application’s code via callbacks, that knowledge becomes essential.

  As an open-source Ruby gem developer, I’ve encountered a few bugs that were resolved using the Rails Executor. For this reason, knowledge of the Rails Executor is essential for developers creating gems that are calling Rails app code. And you’ll get some interesting knowledge about how Rails works along the way!
drawings:
  persist: false
fonts:
  provider: none
  fallback: false
  local: Martian Grotesk, Martian Mono
  sans: Martian Grotesk
  serif: Martian Grotesk
  mono: Martian Mono
aspectRatio: '16/9'

transition: slide-left
title: Threads, callbacks, and execution context in Ruby
mdc: true
---

# Rails Executor

the border between application and framework code


<div class="absolute bottom-0 left-0 w-full px-14 py-8 flex justify-between items-end gap-4">
  <div class="text-left text-xl">
    Andrey Novikov, Evil Martians<br />
    <small><a href="https://kaigionrails.org/2023/">Kaigi on Rails 2023</a></small><br />
    <small><time datetime="2023-10-27">27 October 2023</time></small>
  </div>

  <div class="flex gap-4">
  <div class="w-28 h-28 scaled-image justify-self-end">
    <a href="https://evilmartians.com/"><img alt="Evil Martians" src="/images/01_Evil-Martians_Logo_v2.1_RGB.svg" class="block dark:hidden" /><img alt="Evil Martians" src="/images/02_Evil-Martians_Logo_v2.1_RGB_for-Dark-BG.svg" class="hidden dark:block" /></a>
  </div>
  <div class="w-28 h-28 scaled-image justify-self-end">
    <a href="https://kaigionrails.org/2023/"><img alt="Kaigi on Rails 2023" src="/images/kaigionrails2023_logo_rgb.svg" class="block" /></a>
  </div>
  </div>
</div>

<style>
  a {
    border-bottom: none !important;
  }
</style>

<!--
皆さん、こんにちは！

今日はあまり知られていないRailsの内部詳細について話したいです。通常のRailsアプリケーション開発者には見えない、Rails Executorというものです。

Hey everyone, let's talk about one for not very well known Rails internals, that lays a bit beyound the world, observable by an application developer, Rails Executor.
-->

---
layout: image-right
image: ./images/20230305_193526.jpg
class: annotated-list
---

## About me

Hi, I'm Andrey (アンドレイ){class="text-xl"}

- Back-end engineer at Evil Martians

- Writing Ruby, Go, and whatever

  <small>SQL, Dockerfiles, TypeScript, bash…</small>

- Love open-source software

  <small>Created and maintaining a few Ruby gems</small>

- Living in Japan for 1 year already

- Driving a moped

  <small>And also a bicycle to get kids to kindergarten</small>

<div class="text-center absolute bottom-0">
<img src="/images/01_Evil-Martians_Logo_Lurkers_v2.0_on-Transparent.png" class="max-w-25% scaled-image mx-auto" />
</div>

<!--
アンドレイと申します。バックエンドエンジニアで、RubyやGoを使っています。オープンソースの大ファンで、いくつかのRubyのジェムを作りました。

もう一年以上大阪の近くに住んでいて、家族と一緒に日本のくらしを楽しんでいます。

よろしくお願いします！

I'm Andrey, back-end engineer at Evil Martians. I and my family are living in Japan for 1 year already, just a bit north of Osaka.
-->

---

<a href="https://evilmartians.com/?utm_source=kaigionrails&utm_medium=slides&utm_campaign=postgresql-as-seen-by-rubyists">
<img alt="Evil Martians" src="/images/01_Evil-Martians_Logo_v2.1_RGB.svg" class="block dark:hidden object-contain text-center m-auto max-h-100" />
<img alt="Evil Martians" src="/images/02_Evil-Martians_Logo_v2.1_RGB_for-Dark-BG.svg" class="hidden dark:block object-contain text-center m-auto max-h-100" />
<p class="text-2xl text-center">evilmartians.com</p>
</a>
<a href="https://evilmartians.jp/?utm_source=kaigionrails&utm_medium=slides&utm_campaign=postgresql-as-seen-by-rubyists">
<p class="text-xl text-center">🇯🇵 evilmartians.jp 🇯🇵</p>
</a>
<div class="absolute bottom-32px left-32px rotate-10 text-2xl">邪悪な火星人？</div>
<div class="absolute bottom-32px right-32px rotate-350 text-2xl">イービルマーシャンズ！</div>

<!--
さらに自分は火星人です。我々は、平和目的で地球に来ました。

真面目に言うと、「イービル・マーシャンズ」という会社に勤めています。

我々は開発者ツールのスタートアップと協力して、それらのスタートアップをユニコーンに変え、素晴らしいオープンソースも開発します。いろいろなスタートアップや大企業にコンサルティングもしています。バックエンドをもちろん、フロントエンドやデザインも含めてプロダクトをターンキー開発しています。

イービル・マーシャンズは元々に Ruby on Rails 開発ショップとして知られてきましたが、それをはるかに超えています。

Also I'm not only an alien, but also a martian. We came to Earth in peace.

We work with growth stage startups focusing on developer tools. To transform those startups into unicorns, build amazing products, and create awesome open source products.

Evil Martians is a team of senior developers and designers who know the industry really well. Historically we've been known as a Ruby on Rails development shop, but we go much beyond that.
-->

---

# Martian Open Source

<div class="grid grid-cols-4 grid-rows-2 gap-4">
  <a href="https://github.com/yabeda-rb/yabeda">
    <figure>
      <img alt="Yabeda" src="/images/martian-oss/yabeda.svg" class="scaled-image h-36 mx-auto" />
      <figcaption>Yabeda: Ruby application instrumentation framework</figcaption>
    </figure>
  </a>
  <a href="https://github.com/evilmartians/lefthook">
    <figure>
      <img alt="LeftHook" src="/images/martian-oss/lefthook.svg" class="scaled-image h-36 mx-auto" />
      <figcaption>Lefthook: git hooks manager</figcaption>
    </figure>
  </a>
  <a href="https://anycable.io/">
    <figure>
      <img alt="AnyCable" src="/images/martian-oss/anycable.svg" class="scaled-image h-36 mx-auto" />
      <figcaption>AnyCable: a real-time server for Rails and beyound</figcaption>
    </figure>
  </a>
  <a href="https://postcss.org/">
    <figure>
      <img alt="PostCSS" src="/images/martian-oss/postcss.svg" class="scaled-image h-36 mx-auto" />
      <figcaption>PostCSS: A tool for transforming CSS with JavaScript</figcaption>
    </figure>
  </a>
  <a href="https://imgproxy.net/">
    <figure>
      <img alt="Imgproxy" src="/images/martian-oss/imgproxy-light.svg" class="scaled-image h-36 mx-auto block dark:hidden" />
      <img alt="Imgproxy" src="/images/martian-oss/imgproxy-dark.svg" class="scaled-image h-36 mx-auto hidden dark:block" />
      <figcaption>Imgproxy: Fast and secure standalone server for resizing and converting remote images</figcaption>
    </figure>
  </a>
  <a href="https://github.com/evilmartians/figma-polychrom">
    <figure>
      <img alt="Polychrom" src="/images/martian-oss/polychrom.svg" class="scaled-image h-36 mx-auto block dark:hidden" />
      <figcaption>A Figma plugin that ensures UI text is readable by leveraging the new APCA algorithm</figcaption>
    </figure>
  </a>
  <a href="https://github.com/DarthSim/overmind">
    <figure>
      <img alt="Overmind" src="/images/martian-oss/overmind.svg" class="scaled-image h-36 mx-auto" />
      <figcaption>Overmind: Process manager for Procfile-based applications and tmux </figcaption>
    </figure>
  </a>
  <a href="https://evilmartians.com/oss">
    <figure>
      <div class="h-36 text-2xl flex items-center justify-center">
        <qr-code-vue value="https://evilmartians.com/oss" class="scaled-image w-full h-full mx-auto p-4" render-as="svg" margin="1" />
      </div>
      <figcaption style="font-size: 1rem; margin-top: 0; line-height: 1.25rem;">Even more at evilmartians.com/oss</figcaption>
    </figure>
  </a>
</div>

<style>
  a { border-bottom: none !important; }
  figcaption {
    margin-top: 0.5rem;
    font-size: 0.6rem;
    line-height: 1rem;
    text-align: center;
  }
</style>

<!--
それに、我々はオープンソースの大ファンなので、できる限りオープンソースソフトウェアを使ったり、貢献したり、そしてよく自分のライブラリやプロダクトを作って維持しています。このスライドでは一番有名なものですが、今は火星で作ったオープンソースプロジェクトが数十のものが存在しています。どうぞ自由に使ってください。

One of our pillars is open source. We use open source products, and we create our own. And most probably there is a gem or two in your application Gemfile as well! Here is just a small part of our open source projects, but you can find much more at our website.
-->

---
layout: section
---

# The border

Why we need to distinguish between application and framework code?

<!--

では、今日の話に入りましょう。なぜアプリケーションコードとフレームワークコードを区別する必要があるのでしょうか？

But let's get to the topic! Have you ever asked yourself why we need to distinguish between application and framework code?

-->

---

## What developer see

```ruby
class KaigiOnRailsController < ApplicationController
  # framework code executes before action
  def index
    # your code here
  end
  # framework code executes after action
  # implicit rendering (your code again)
  # framework code executes after action
end
```

<!--

Rails アプリケーションの最小限の部分を見てみましょう。これはコントローラのアクションです。そして、これは空でも大丈夫で、自動的にビューをレンダリングします。設定より規約が優先の影で簡潔性のことは、私たちがRailsが大好きの理由の一つです。

Let's take a look at minimal piece of a Ruby on Rails application. It is a controller action, and sometimes it can be absolutely empty, rendering some view template automatically. That's one of the reason why we love Ruby on Rails: its conciseness.
-->

---

## What path it takes

```sh
$ rails middleware

use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
use ActionDispatch::Executor # Spoiler alert!
use Rack::Runtime
use ActionDispatch::RequestId
use ActionDispatch::RemoteIp
use Rails::Rack::Logger
# 18 entries skipped
run MyApp::Application.routes
```

A long way to serve a request!

<!--

もちろん、魔法だと見えても、魔法ではないんです。自分でコードを書く必要がないことは、フレームワークのコードがそれをやってくれるということです。

Railsが遅いと文句を言う人がいますが、そうではありません。それはあなたのために多くの作業を行っているだけです。したがって、生産性を高めることができます。

However, there are no magic. If you don't write code to do the job, that means that framework has to have that code.

And people sometimes can complain that Rails is slow. But it is not. It is just doing a lot of work behind the scenes for you. So you can be productive.
-->

---

## Two worlds

<div class="text-xl my-8">

 - **Framework code** (and gems’ code also)
 - Your **application code**

</div>

And for basic actions a lot more framework code is executed than your code.

<hr class="my-8">

It can be compared to a kernel and user space in an operating system.

 - Kernel code manages resources and provides APIs
 - User space code is executed by, and uses APIs provided by kernel

<!--

ですので、それにRubyもRailsも開発者の生産性を重視していることで、ほとんどの場合、特に単純なアクションの場合、アプリのコードよりもはるかに多くのフレームワークコードが実行されるという状況が発生します。

これは、オペレーティング システムのカーネルとユーザー空間によく似ています。 カーネルコードはリソースを管理し、実装の詳細を隠すため、ユーザー空間プログラムはsyscallを呼び出すだけでよく、実装方法は気にしません。

And this, and also the focus of both Ruby and Rails on developer productivity, leads to a situation when in most cases, especially for simpler actions, there is a lot more framework code executed than your code.

It is very much like a kernel and user space in an operating system. Kernel code manages resources and hides implementation details, so user space program can just call syscalls and don't care about how it is implemented.
-->

---

## What we take for granted

<Transform scale="1.2">

 1. Automatic code reloading in development
 2. Implicit database connection management
 3. Caches cleanup
 4. and more…

</Transform>

<!--

Railsはあなたのために多くの仕事をしてくれます。 私たちが当たり前だと思っていることがたくさんあります。 たとえば、開発時のコードの自動再読み込みや、暗黙的なデータベース接続管理などのさまざま退屈なことです。

And Rails does a lot of work for you. There is a lot of things that we take for granted. For example, automatic code reloading in development. Or implicit database connection management. Or caches cleanup. And so on.

-->

---

## Code reloading

<Transform scale="1.2">

It should be fast or developer experience will be bad.

And framework and gems usually doesn't change at all.

So only application code need to be reloaded.

</Transform>

<!--

開発者の生産性にとって主なことは、もちろん、開発中に変更されたコードの自動的な再読み込みのこと、再リロードです。高速である必要があります。そうしないと、開発者のエクスペリエンスが低下します。 また、フレームワークとジェムのコードが多数あるため、変更のたびにアプリケーション全体をリロードすることは受け入れられません。したがって、アプリケーションのコードのみをリロードする必要があります。

Main thing for developer productivity is, of course, automatic reloading of changed code in development. It should be fast or developer experience will be bad. And as there is a lot of framework and gem code, it is not a good idea to reload the whole application on every change. So only application code, your code, need to be reloaded.

-->

---

## Resource management

We've got used not to care about connection pools, etc.

```ruby
class KaigiOnRailsController < ApplicationController
  # framework code executes before action
  def index
    record = MyModel.find_by(…)
    record.update(…)
    # Database connection? What database connection?
  end
end
```

<!--
次に重要なことはリソース管理です。 接続プールやキャッシュのクリーンアップなどを気にする必要はありません。コードを記述するだけで機能します。

Second most important thing is resource management. We don't have to care about connection pools, cleaning up caches, etc. We just write our code and it works.
-->

---

## Resource management

If we had to do it manually:

```ruby{4,7}
# This is not real code!
class KaigiOnRailsController < ApplicationController
  def index
    ActiveRecord::Base.connection_pool.with_connection do |conn|
      record = MyModel.find_by(…, connection: conn)
      record.update(…, connection: conn)
    end
  end
end
```

(Thanks to all possible gods we don't have to!)

<!--
もしかしたら、これを手動で行う必要があったら、次のようになります。非常に冗長で面倒で、生産性もなくなると思います。
-->

---

## Why I should care?

Usually, you should not!{class="text-2xl mt-16"}

<hr class="my-8">

That's why we love Ruby on Rails.

And that's why there is Rails Executor.

<!--

アプリケーションを開発するなら、これについて知る必要さえありません。ただRailsを使用して、楽しんでください！

What is best part of this? If all you do is writing application code, you don't even need to know about this! Just use Rails.
-->

---

## But when I should care?

When you are writing a gem that _calls_ application code.

```ruby{3-5}
class KaigiOnRailsController < ApplicationController
  def index
    MyGem.do_something_later do
      # Your callback code here
    end
    # render something
  end
end
```

<!--
ただし、アプリケーションコードを呼び出すジェムを作成したいとき、知る必要になります。

もし、あなたの作成したジェムが、アプリからのコールバックを受け、保存して、いつか呼び出すようなものであれば、呼び出す前後にリソース管理を行わないと、厄介なバグの発生するおそれがあります。

また、使っているツールの内部詳細を知ることは良いことですよね。これは、プロフェッショナルとして成長しに役くに立ちます。サードパーティのgemの問題をデバッグするにも役に立ちます。

However, when you want to step out of comfort zone of application code and write a library for your application, a gem that will provide an API that accepts a callback with application code and eventually calls it. Now you'd better know about how to do it properly to avoid tricky bugs from happening.

Also, it is a good thing to know internal details of tools you are using. It will help you to grow as a professional and debug issues in gems that was written by others.
-->

---

## Examples of gems that call application code

<Transform scale="1.2" class="mt-16">

- Background jobs: Sidekiq, DelayedJob, GoodJob…
- Scheduled jobs: Whenever, …
- Non-HTTP handlers: ActionCable, AnyCable…
- Custom instrumentation: Yabeda…
- Messaging: NATS subscriptions, Karafka consumers, …

</Transform>


<!--
また、アプリケーションコードを呼び出すジェムの種類も数も多くあります。 バックグラウンド・ジョブ、cron ジョブ、メッセージング、などなど。

And there is a lot of kinds of gems that may need to call application code. Background jobs, cron jobs, instrumentation, messaging.
-->

---
layout: section
---

# Rails Executor

**Border point** between application and framework code

Or API to call application code from framework code

<!--

では、今日はRails Executorを紹介したいと思います。これは、アプリケーションコードとフレームワークコードの境界点です。 または、フレームワークコードからアプリケーションコードを呼び出すためのAPIです。

Welcome to Rails executor, the border point between application and framework code. API that allows you to travel back and forth between these two worlds.

-->

---
layout: footnote
---

## Rails Executor

<Transform scale="1.2">

- Wraps _unit of work_ (action, job, etc.)

- Is re-entrant

  <small>safe to call wrap inside of another wrap and  from different threads</small>

- Defines callbacks `to_run` and `to_complete`

  <small>called before and after enclosed block</small>

<p class="mt-4 px-4 py-2 border-2 border-blue-500 float-left">
Use it when you need to call application code once.
</p>

</Transform>

::footnote::

Read more: [guides.rubyonrails.org/threading_and_code_execution.html](https://guides.rubyonrails.org/threading_and_code_execution.html#executor)

<qr-code url="https://railsguides.jp/threading_and_code_execution.html#executor" caption="Rails guides on threading (日本語)" class="w-42 absolute bottom-10px right-10px" />

<!--

Rails Executorは「仕事の単位」を受けます。コントローラーのアクションもバックグラウンドジョブも全ては「仕事の単位」になります。アプリケーションコードのコールバックという意味です。

このコールバックを実行する前に、Executorはto_runという自分のコールバックを実行してから、アプリケーションコードを呼び出して、その後にto_completeというコールバックを実行します。以上です、こんな簡単なものです。

Executorに包まれたコードはもう一回Executorに包んでもかまいません、安全です。

ただし、Executorはリソース管理を行いますが、コードの再読み込みは行いません。

Rails Executor wraps _unit of work_ of an application: it can be controller action, background job, whatever. It defines callbacks `to_run` and `to_complete` that are called before and after enclosed block. That's it, that simple.

But Executor handles only resource management, it doesn't reload code.

-->

---
layout: footnote
---

## Rails Reloader

<Transform scale="1.2">

- Wraps _unit of work_ too

- Calls Executor if needed (its callbacks are executed)

- Reloads code if changed

- Adds more callbacks (called only on code reload!)

  <small>`to_prepare`, `to_run`, `to_complete`, `before_class_unload`, `after_class_unload`</small>

<p class="mt-4 px-4 py-2 border-2 border-blue-500 float-left">
Use it in long running processes instead of Executor
</p>

</Transform>

::footnote::

Read more: [guides.rubyonrails.org/threading_and_code_execution.html](https://guides.rubyonrails.org/threading_and_code_execution.html#reloader)

<qr-code url="https://railsguides.jp/threading_and_code_execution.html#reloader" caption="Rails guides on threading (日本語)" class="w-42 absolute bottom-10px right-10px" />

<!--

Executorとともに、Rails Reloaderもあります。これは、Executorをラップして、アプリケーションコードを実行する前に、最新のコードが読み込まれているかを確実します。

長時間実行されるプロセスでは、リソース管理だけでなくコードの再読み込みも行うために、ExecutorではなくReloaderを使用されるべきです。

For that purpose there is a separate Rails Reloader which actually wraps Executor. It also reloads code before every unit of work.

Long running processes should use Reloader instead of Executor, to get not only resource management, but also code reloading.
-->

---

## How to integrate with Rails Executor/Reloader

**Case 1**: to call application code from a gem code.

Wrap the call to application code in `Rails.application.executor.wrap` or, most often, `Rails.application.reloader.wrap`:

```ruby
Rails.application.reloader.wrap do
  # call application code here
end
```

And that's it!{class="text-2xl"}

<!--
ジェムコードからアプリケーションコードを呼び出すとき、リソース管理とコードの再読み込みを行うには必要なのは、「仕事の単位」の呼び出しを「Rails.application.reloader.wrap」で包むだけです。簡単じゃないですか？

To just call application code from your gem code to get resource management and code reloading, all you need is to wrap the call to a unit of work in `Rails.application.reloader.wrap` and that's it! Isn't it easy?
-->

---

## How to integrate with Rails Executor/Reloader

**Case 2**: to do something before/after every request/job/etc.

Register `to_run` and `to_complete` callbacks on the application *Executor* instance:

```ruby
Rails.application.executor.to_run do
  # do something before
end

Rails.application.executor.to_complete do
  # do something after
end
```

<!--
「仕事の単位」の前後にリソースを管理したい場合は、アプリケーションの Executorのインスタンスに `to_run` および `to_complete` コールバックを登録してください。

If you want to do manage some resources before or after a unit of work, you can register `to_run` and `to_complete` callbacks on the application Executor instance.
-->

---

## How to integrate with Rails Executor/Reloader

**Case 3**: to do something before/after every code reload.

Register `to_prepare` or other callbacks on the application *Reloader* instance:

```ruby
Rails.application.reloader.to_prepare do
  # do something whe code has been reloaded
end
```

<!--
コードの再読み込みの前後に何かをしたい場合は、アプリケーションの Reloader のインスタンスに `to_prepare` またはその他のコールバックを登録してください。

If you want to do something before or after code reload, you can register `to_prepare` or other callbacks on the application Reloader instance.
-->

---
layout: statement
---

## So all aforementioned gems are doing it?

<p class="text-5xl rotate-10 animate-pulse text-green-500 p-4 border-3 border-green-500 font-black mx-auto max-w-50 text-center">YES!</p>

<!--

前に述べた多くの、実際にはほとんどのジェムがアプリケーションコードを呼び出すためにRails Executorを使用していることを意味します。 これは、長時間実行されるプロセスが確実に動作するための前提条件です。

And it means that many, actually most of the gems I listed before are using Rails Executor/Reloader to call application code. It is a precondition for reliable work of long running processes.

-->

---
layout: footnote
footnote-class: text-sm
---

## Rails itself uses it too <small>(of course!)</small>

ActionCable wraps every incoming WebSocket connection into Rails Executor:

<iframe class="speakerdeck-iframe max-h-80 max-w-136 my-8" style="border: 0px; background: rgba(0, 0, 0, 0.1) padding-box; padding: 0px; border-radius: 6px; box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 40px; width: 100%; height: auto; aspect-ratio: 560 / 315;" frameborder="0" src="https://speakerdeck.com/player/1b44a2be77c242d2b55b7e43a981dda9?slide=58" title="[RailsWorld 2023] Untangling cables &amp; demystifying twisted transistors" allowfullscreen="true" data-ratio="1.7777777777777777"></iframe>

<Arrow x1="900" y1="250" x2="555" y2="250" />

::footnote::

See [RailsWorld 2023: Untangling cables & demystifying twisted transistors](https://speakerdeck.com/palkan/railsworld-2023-untangling-cables-and-demystifying-twisted-transistors?slide=58)

<qr-code url="https://speakerdeck.com/palkan/railsworld-2023-untangling-cables-and-demystifying-twisted-transistors?slide=58" caption="Action Cable Executor slide" class="w-42 absolute bottom-10px right-10px" />

<!--
もちろん、Rails 自体も Rails Executor を使用しています。 コントローラーのアクションもActiveJobもフレームワークのすべてのコンポーネントはアプリコードをExecutor軽油で呼び出します。 たとえば、ActionCableは、すべてのWebSocketのメッセージを処理する時は、チャンネルのアクションをExecutorにラップして呼び出します。

Action Cable アーキテクチャに興味があったら、同僚の Vladimir Dementyev の今年のRailsWorldのトークをご覧ください。

And of course Rails itself uses Rails Executor/Reloader too. For controller actions, for ActiveJob, for all components of the framework. For example, ActionCable wraps every incoming WebSocket connection into Rails Executor.

If you are interested in Action Cable architecture, please watch a talk from my colleague Vladimir Dementyev at RailsWorld 2023 conference.
-->

---
layout: footnote
---

## My contribution: NATS subscriptions

NATS is a modern, simple, secure and performant message communications system for microservice world.

```ruby
nats = NATS.connect("demo.nats.io")

nats.subscribe("service") do |msg|
  # Your logic to handle incoming messages
end
```

- runs in a long-running process
- executes application code callback multiple times

Nice to have automatic code reloading!

::footnote::

See [nats-pure.rb pull request № 120](https://github.com/nats-io/nats-pure.rb/pull/120)

<qr-code url="https://github.com/nats-io/nats-pure.rb/pull/120" caption="nats-pure.rb pull request № 120" class="w-42 absolute bottom-10px right-10px" />

---
layout: footnote
---

## Typical example: resource management

[ActionPolicy](https://actionpolicy.evilmartians.io/) gem caches authorization rules in a per-thread cache.

```ruby{3-6}
module ActionPolicy
  class Railtie < ::Rails::Railtie
    initializer "action_policy.clear_per_thread_cache" do |app|
      app.executor.to_run { ActionPolicy::PerThreadCache.clear_all }
      app.executor.to_complete { ActionPolicy::PerThreadCache.clear_all }
    end
  end
end
```

Rails Executor cleans up this cache between requests.

::footnote::

See [`lib/action_policy/railtie.rb:59`](https://github.com/palkan/action_policy/blob/8204e9b82f2767728dce69fccb5c4b2088c532ec/lib/action_policy/railtie.rb#L59-L62)

<qr-code url="https://github.com/palkan/action_policy/blob/8204e9b82f2767728dce69fccb5c4b2088c532ec/lib/action_policy/railtie.rb#L59-L62" caption="action_policy/railtie.rb:59" class="w-42 absolute bottom-10px right-10px" />


---
layout: footnote
---

## Advanced usage: AnyCable messaging batching

```ruby
executor.to_run do
  # Start collecting messages instead of sending them immediately
  AnyCable.broadcast_adapter.start_batching
end

executor.to_complete do
  # Send all collected messages at once
  AnyCable.broadcast_adapter.finish_batching
end
```

Less network round-trips, guaranteed order of messages.

::footnote::

See [anycable-rails pull request № 189](https://github.com/anycable/anycable-rails/pull/189)

<qr-code url="https://github.com/anycable/anycable-rails/pull/189" caption="anycable-rails pull request № 189" class="w-42 absolute bottom-10px right-10px" />

---
layout: footnote
---

## Advanced usage: ViewComponent previews

Use Reloader `to_prepare` callback to show actual [Source Previews](https://viewcomponent.org/guide/previews.html#source-previews) of components in development.

```ruby
app.config.to_prepare do
  # Clear source code cache on code reload
  MethodSource.instance_variable_set(:@lines_for_file, {})
end
```

::footnote::

See [view_component pull request № 1147](https://github.com/ViewComponent/view_component/pull/1147)

<qr-code url="https://github.com/ViewComponent/view_component/pull/1147" caption="view_component pull request № 1147" class="w-42 absolute bottom-10px right-10px" />

---
layout: section
---

# That’s it!

---

## Up to the next time!

Come to Izumo, Shimane, Japan!

<a href="https://evilmartians.com/events/kujira-ni-notta-ruby-izumorb">
<img src="/images/izumo-rb-announce.png" class="w-75%" />
</a>

See you at Izumo Ruby meet-up on 2023-11-11!

<qr-code url="https://evilmartians.com/events/kujira-ni-notta-ruby-izumorb" caption="Izumo Ruby meet-up talk announce" class="w-42 absolute bottom-10px right-10px" />

---

# Thank you!

<div class="grid grid-cols-[8rem_3fr_4fr] mt-12 gap-2">

<div class="justify-self-start">
<img alt="Andrey Novikov" src="https://secure.gravatar.com/avatar/d0e95abdd0aed671ebd0920c16d393d4?s=512" class="w-32 h-32 scaled-image" />
</div>

<ul class="list-none">
<li><a href="https://github.com/Envek"><logos-github-icon class="dark:invert" /> @Envek</a></li>
<li><a href="https://twitter.com/Envek"><logos-twitter /> @Envek</a></li>
<li><a href="https://facebook.com/Envek"><logos-facebook /> @Envek</a></li>
<li><a href="https://t.me/envek"><logos-telegram /> @Envek</a></li>
</ul>

<div>
<qr-code url="https://github.com/Envek" caption="github.com/Envek" class="w-32 mt-2" />
</div>

<div class="justify-self-start">
<a href="https://evilmartians.com/"><img alt="Evil Martians" src="/images/01_Evil-Martians_Logo_v2.1_RGB.svg" class="w-32 h-32 scaled-image block dark:hidden" /><img alt="Evil Martians" src="/images/02_Evil-Martians_Logo_v2.1_RGB_for-Dark-BG.svg" class="w-32 h-32 scaled-image hidden dark:block" /></a>
</div>

<div>

- <logos-github-icon class="dark:invert" /> [@evilmartians](https://github.com/evilmartians?utm_source=kaigionrails&utm_medium=slides&utm_campaign=rails-executor)
- <logos-twitter /> [@evilmartians](https://twitter.com/evilmartians/?utm_source=kaigionrails&utm_medium=slides&utm_campaign=rails-executor)
- <logos-twitter /> [@evilmartians_jp](https://twitter.com/evilmartians_jp/?utm_source=kaigionrails&utm_medium=slides&utm_campaign=rails-executor) <small>(日本語)</small>
- <logos-instagram-icon class="dark:invert" /> [@evil.martians](https://www.instagram.com/evil.martians/?utm_source=kaigionrails&utm_medium=slides&utm_campaign=rails-executor)
</div>

<div>
<qr-code url="https://evilmartians.jp/" caption="evilmartians.jp" class="w-32 mt-2" />
</div>

<div class="col-span-3">

Our awesome blog: [evilmartians.com/chronicles](https://evilmartians.com/chronicles/?utm_source=kaigionrails&utm_medium=slides&utm_campaign=rails-executor)!

<p class="text-sm">See these slides at <a href="https://envek.github.io/kaigionrails-rails-executor/">envek.github.io/kaigionrails-rails-executor</a></p>

</div>
</div>

<style>
  ul a { border-bottom: none !important; }
  ul { list-style-type: none !important; }
  ul li { margin-left: 0; padding-left: 0; }
</style>

<!--

最後までご視聴してくださって、ありがとうございました！

我が社のブログでは、Rubyについての記事がたくさんあります。ぜひお読みください！日本語の翻訳もあります。

-->
