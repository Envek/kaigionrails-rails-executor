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


<div class="absolute bottom-0 left-0 w-full px-10 py-8 grid grid-cols-2 justify-items-stretch items-end gap-4">
  <div class="text-left text-xl">
    Andrey Novikov, Evil Martians<br />
    <small><a href="https://kaigionrails.org/2023/">Kaigi on Rails 2023</a></small><br />
    <small><time datetime="2023-10-27">27 October 2023</time></small>
  </div>

  <div class="w-28 h-28 scaled-image justify-self-end">
    <a href="https://evilmartians.com/"><img alt="Evil Martians" src="/images/01_Evil-Martians_Logo_v2.1_RGB.svg" class="block dark:hidden" /><img alt="Evil Martians" src="/images/02_Evil-Martians_Logo_v2.1_RGB_for-Dark-BG.svg" class="hidden dark:block" /></a>
  </div>
</div>

<style>
  a {
    border-bottom: none !important;
  }
</style>

<!-- 皆さん、こんにちは！ -->

---
layout: image-right
image: ./images/20230305_193526.jpg
class: annotated-list
---

## About me

Hi, I'm Andrey (アンドレイ){class="text-xl"}

- Back-end engineer at Evil Martians

- Writing Ruby, Go, and whatever

  SQL, Dockerfiles, TypeScript, bash…

- Love open-source software

  Created and maintaining a few Ruby gems

- Living in Japan for 1 year already

- Driving a moped

  And also a bicycle to get kids to kindergarten

<div class="text-center absolute bottom-0">
<img src="/images/01_Evil-Martians_Logo_Lurkers_v2.0_on-Transparent.png" class="max-w-25% scaled-image mx-auto" />
</div>

<!--
はじめまして、アンドレイと申します。もう一年以上大阪の近くに住んでいます。
-->

---

<a href="https://evilmartians.com/?utm_source=kaigionrails&utm_medium=slides&utm_campaign=postgresql-as-seen-by-rubyists">
<img alt="Evil Martians" src="/images/01_Evil-Martians_Logo_v2.1_RGB.svg" class="block dark:hidden object-contain text-center m-auto max-h-112" />
<img alt="Evil Martians" src="/images/02_Evil-Martians_Logo_v2.1_RGB_for-Dark-BG.svg" class="hidden dark:block object-contain text-center m-auto max-h-112" />
<p class="text-2xl text-center">evilmartians.com</p>
</a>
<div class="absolute bottom-32px left-32px rotate-10 text-2xl">邪悪な火星人？</div>
<div class="absolute bottom-32px right-32px rotate-350 text-2xl">イービルマーシャンズ！</div>

<!--
さらに自分は火星人です。我々は、平和目的で地球に来ました。

真面目に言うと、「イービル・マーシャンズ」という会社に勤めています。

我々はスタートアップや大企業のためにプロジェクトを開発したり、コンサルティングしたりしています。バックエンドをもちろん、フロントエンドやデザインも含めてプロダクトをターンキー開発しています。
-->

---

# Martian Open Source

<div class="grid grid-cols-4 grid-rows-2 gap-4">
  <a href="https://github.com/yabeda-rb/yabeda">
    <figure>
      <img alt="Yabeda" src="/images/martian-oss/yabeda.svg" class="scaled-image h-40 mx-auto" />
      <figcaption>Yabeda: Ruby application instrumentation framework</figcaption>
    </figure>
  </a>
  <a href="https://github.com/evilmartians/lefthook">
    <figure>
      <img alt="LeftHook" src="/images/martian-oss/lefthook.svg" class="scaled-image h-40 mx-auto" />
      <figcaption>Lefthook: git hooks manager</figcaption>
    </figure>
  </a>
  <a href="https://anycable.io/">
    <figure>
      <img alt="AnyCable" src="/images/martian-oss/anycable.svg" class="scaled-image h-40 mx-auto" />
      <figcaption>AnyCable: Polyglot replacement for ActionCable server</figcaption>
    </figure>
  </a>
  <a href="https://postcss.org/">
    <figure>
      <img alt="PostCSS" src="/images/martian-oss/postcss.svg" class="scaled-image h-40 mx-auto" />
      <figcaption>PostCSS: A tool for transforming CSS with JavaScript</figcaption>
    </figure>
  </a>
  <a href="https://imgproxy.net/">
    <figure>
      <img alt="Imgproxy" src="/images/martian-oss/imgproxy-light.svg" class="scaled-image h-40 mx-auto block dark:hidden" />
      <img alt="Imgproxy" src="/images/martian-oss/imgproxy-dark.svg" class="scaled-image h-40 mx-auto hidden dark:block" />
      <figcaption>Imgproxy: Fast and secure standalone server for resizing and converting remote images</figcaption>
    </figure>
  </a>
  <a href="https://github.com/evilmartians/figma-polychrom">
    <figure>
      <img alt="Polychrom" src="/images/martian-oss/polychrom.svg" class="scaled-image h-40 mx-auto block dark:hidden" />
      <figcaption>A Figma plugin that ensures UI text is readable by leveraging the new APCA algorithm</figcaption>
    </figure>
  </a>
  <a href="https://github.com/DarthSim/overmind">
    <figure>
      <img alt="Overmind" src="/images/martian-oss/overmind.svg" class="scaled-image h-40 mx-auto" />
      <figcaption>Overmind: Process manager for Procfile-based applications and tmux </figcaption>
    </figure>
  </a>
  <a href="https://evilmartians.com/oss">
    <figure>
      <div class="h-40 text-2xl flex items-center justify-center">
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
-->

---
layout: section
---

# The border

Why we need to distinguish between application and framework code?

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

---

## What path it takes

```sh
$ rails middleware

use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
use ActionDispatch::Executor # Spoiler alert!
use ActionDispatch::ServerTiming
use ActiveSupport::Cache::Strategy::LocalCache::Middleware
use Rack::Runtime
use ActionDispatch::RequestId
use ActionDispatch::RemoteIp
use Rails::Rack::Logger
# 16 entries skipped
run MyApp::Application.routes
```

A long way to serve a request!

---

## Two worlds

 - Framework code (and gems’ code also)
 - Your application code

And for basic actions a lot more framework code is executed than your code.

---

## What we take for granted

 1. Automatic code reloading in development
 2. Implicit database connection management
 3. … 

---

## Code reloading

It should be fast or developer experience will be bad.

And framework and gems usually doesn't change at all.

So only application code need to be reloaded.

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

---

## Resource management

If we had to do it manually:

```ruby{4,7}
class KaigiOnRailsController < ApplicationController
  # framework code executes before action
  def index
    ActiveRecord::Base.connection_pool.with_connection do |conn|
      record = MyModel.find_by(…)
      record.update(…)
    end
  end
end
```

(Thanks to all possible gods we don't have to!)

---

## Why I should care?

You should not!{class="text-2xl"}

That's why we love Ruby on Rails.

And that's why there is Rails Executor.

---

## But when I should care?

When you are writing a gem that _calls_ application code.

---

## Examples of gems that call application code

- Background jobs: Sidekiq, DelayedJob, GoodJob…
- Scheduled jobs: Whenever, …
- Non-HTTP handlers: ActionCable, AnyCable…
- Custom instrumentation: Yabeda…
- Messaging: NATS subscriptions, Karafka consumers, …

---
layout: footnote
---

# Rails Executor

<Transform scale="1.25">

- Wraps _unit of work_ (action, job, etc.)

- Is re-entrant

  <small>safe to call wrap inside of another wrap and  from different threads</small>

- Defines callbacks

  <small>called before or after enclosed block</small>

Use it when you need to call application code once.

</Transform>

::footnote::

Read more: [guides.rubyonrails.org/threading_and_code_execution.html](https://guides.rubyonrails.org/threading_and_code_execution.html#executor)

<qr-code url="https://railsguides.jp/threading_and_code_execution.html#executor" caption="Rails guides on threading (日本語)" class="w-42 absolute bottom-10px right-10px" />

---
layout: footnote
---

## Rails Reloader

<Transform scale="1.25">

- Wraps _unit of work_ too

- Calls Executor if needed

- Reloads code if changed

- Adds more callbacks called only on code reload

Use it in long running processes instead of Executor

</Transform>

::footnote::

Read more: [guides.rubyonrails.org/threading_and_code_execution.html](https://guides.rubyonrails.org/threading_and_code_execution.html#reloader)

<qr-code url="https://railsguides.jp/threading_and_code_execution.html#reloader" caption="Rails guides on threading (日本語)" class="w-42 absolute bottom-10px right-10px" />

---

## How to integrate with Rails Executor

Just wrap the palce that calls application code in `Rails.application.executor.wrap` or `Rails.application.reloader.wrap`:

```ruby
Rails.application.reloader.wrap do
  # call application code here
end
```

And that's it!{class="text-2xl"}

---

## So all aforementioned gems are doing it?

YES!{class="text-2xl mb-4"}

E.g. see how Sidekiq does it:

- Wraps every job execution to reloader

  See [`lib/sidekiq/processor.rb:135`](https://github.com/sidekiq/sidekiq/blob/7f83b2afca8fdfff87ebbb1742826e4fd9887be0/lib/sidekiq/processor.rb#L135-L142)

- In Rails app it calls Rails Executor/Reloader

  See [`lib/sidekiq/rails.rb`](https://github.com/sidekiq/sidekiq/blob/7f83b2afca8fdfff87ebbb1742826e4fd9887be0/lib/sidekiq/rails.rb#L15-L17)

- In non-Rails it is just a pass-through block

  See [`lib/sidekiq/config.rb:33`](https://github.com/sidekiq/sidekiq/blob/7f83b2afca8fdfff87ebbb1742826e4fd9887be0/lib/sidekiq/config.rb#L33)

- And you can write and provide your own!

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
