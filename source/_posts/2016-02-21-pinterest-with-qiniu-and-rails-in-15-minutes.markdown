---
layout: post
title: '"Pinterest" with Qiniu and Rails in 15 minutes'
date: 2016-02-27 19:42:42 +0800
comments: true
categories: [Rails, Qiniu in 15 Minutes]
---

{% video http://7xp2m5.com2.z0.glb.qiniucdn.com/%22Pinterest%22%20with%20Qiniu%20and%20Rails%20in%2015%20minutes.mp4 800 600 http://7xp2m5.com2.z0.glb.qiniucdn.com/%22Pinterest%22%20with%20Qiniu%20and%20Rails%20in%2015%20minutes.png %}

十年前，Web 应用框架 Rails 创始人 David Heinemeier Hansson 曾录制视频，向我们演示如何使用 Ruby on Rails 在 15 分钟内创作一个 blog 引擎。这个视频通过 Rails 优秀的 MVC 、习惯优于配置（Convention over Configuration）等设计，以及强大的代码生成、scaffold 等功能，成功展示了 Ruby on Rails 编写 Web 应用核心功能的高效简洁。Ruby on Rails 这门技术也在 Web 2.0 时代大放异彩，成为了 Web 应用开发最佳的技术方案选择之一。

经过十年的发展，软件行业早已迈入云计算时代。为了应对大规模的访问量，同时控制研发和运营成本，作为云计算基石的云存储，已经成为了 Web 开发必不可少的基础设施。今天，就让我们一起来看看如何使用
  Rails 和七牛云存储，在 15 分钟内打造一个图片分享社交应用原型。

[七牛云存储](http://www.qiniu.com) 是一个公有云服务，提供海量对象存储功能，以及云端文件处理和分发服务。开始之前，我们需要创建一个七牛云存储 [试用帐户](https://portal.qiniu.com/signup)，并且了解一些基础知识：

* 七牛云存储是一个 Key-Value 形式的对象存储系统，一个 key 对应一个资源（文件）。
* 资源必须存储在某个空间（Bucket）中，不可单独存在。一个帐户可以创建多个空间。

## 创建基本 Rails 项目

你应该可以使用 Ruby 1.9 以上，Rails 3.0 以上的任意版本，本例中我们使用的是 Ruby 2.2.3 和 Rails 4.2.5。

安装好 Ruby 和 Rails 之后，使用 rails 命令创建应用程序 konata 的目录结构和基本文件：

```bash
rails new konata
```

稍候这条命令执行完成，我们立即得到了一个可以运行的空白 Rails 应用程序，执行以下命令，并在浏览器中访问 `http://localhost:3000` 查看运行效果：

```bash
cd konata
rails server
```

## 使用 Rails Scaffold 实现 CRUD

我们将使用 Rails 的 scaffold 功能，生成用于处理图片发表的 model、controller、view，以及 database migration 等源代码文件。

```bash
rails generate scaffold post title filename qiniu_hash
rake db:migrate
```

访问 `http://localhost:3000/posts` 可以看到，我们已经获得了 post 的完整 CRUD 功能，只是暂时还不能上传图片。

## 使用七牛 API 实现图片上传

修改 `Gemfile`，在其中加入对七牛 Ruby SDK 的引用：

```ruby
gem 'qiniu'
```

执行以下命令安装七牛 Ruby SDK。这里原本应该执行 `bundle` 进行安装，但由于七牛 Ruby SDK 依赖的 `mime-types` 版本设定比较保守，需要使用 `bundle update` 命令降级 `mime-types`，解决依赖冲突。

```bash
bundle update mime-types
```

编辑 `config/secrets.yml`，在其中加入七牛云存储帐户的密钥：

```yaml
development:
  secret_key_base: <YOUR_SECRET_KEY_BASE>
  qiniu_access_key: <YOUR_QINIU_ACCESS_KEY>
  qiniu_secret_key: <YOUR_QINIU_SECRET_KEY>
```

创建 `config/initializers/qiniu.rb`，使用刚才加入的密钥与七牛云存储服务器建立连接。内容如下：

```ruby
require 'qiniu'

Qiniu.establish_connection!(
  access_key: Rails.application.secrets.qiniu_access_key,
  secret_key: Rails.application.secrets.qiniu_secret_key
)
```

##### 注意：

AccessKey 和 SecretKey 必须绝对保密，不可出现在用户可以查看的 Web 前端源代码里，或是编译进客户端二进制代码中。

七牛 API 提供了 [多种上传方式](http://developer.qiniu.com/docs/v6/api/overview/up/upload-models/upload-types.html) 以满足不同的业务场景需求。这里我们选择使用最有代表性，也最简单的 HTML 表单上传＋HTTP 303 重定向返回的方式实现客户端文件直接上传七牛云存储。这种方法的好处是客户端文件无需通过业务服务器（app server）中转，既可以利用七牛强大的 CDN 优化上传速度及提高可靠性，也可以节省业务服务器带宽。

编辑 `app/views/posts/_form.html.erb`，根据七牛云存储 SDK 构造上传表单。注意其中的上传凭证字段，我们将在 `PostsController` 里创建它：

```erb
<%= form_tag 'http://upload.qiniu.com', multipart: true do %>
  <%= hidden_field_tag :token, @qiniu_upload_token %>

  <div class="field">
    <%= label_tag :title %><br>
    <%= text_field_tag 'x:title' %>
  </div>
  <div class="field">
    <%= label_tag :image %><br>
    <%= file_field_tag :file %>
  </div>
  <div class="actions">
    <%= submit_tag 'Create' %>
  </div>
<% end %>
```

编辑 `app/controllers/posts_controller.rb`，添加代码生成上传凭证，以及根据七牛云存储自定义响应内容创建 `post` 实例：

```ruby
  def new
    @qiniu_upload_token = generate_qiniu_upload_token
    @post = Post.new
  end

  def create
    upload_ret = JSON.parse(Base64.urlsafe_decode64(params[:upload_ret]))
    @post = Post.new(
      title: upload_ret['title'],
      filename: upload_ret['fname'],
      qiniu_hash: upload_ret['hash']
    )
    # ...
  end

  private

    def generate_qiniu_upload_token
      put_policy = Qiniu::Auth::PutPolicy.new('konata')
      put_policy.return_body = {
        fname: '$(fname)',
        hash: '$(etag)',
        title: '$(x:title)'
      }.to_json
      put_policy.return_url = create_posts_url
      Qiniu::Auth.generate_uptoken(put_policy)
    end
```

编辑 `config/routes.rb`，将 `create` action 定义为使用 `get` 方法亦可访问:

```ruby
  resources :posts do
    collection do
      get 'create', as: :create
    end
  end
```

重新启动 rails server，访问 `http://localhost:3000/posts/new`，现在我们已经可以在发表新 post 的时候上传图片至七牛云存储。

##### 提示：

* 文件将会上传到名为 `konata` 的公开空间（bucket）。
* 我们没有在代码中指定 `key`，七牛云存储默认会使用根据文件内容计算的 hash (etag) 值做为 key。这种做法可以非常简单地避免内容相同的文件存储多份浪费空间。
* 由于上传表单将会直接提交到七牛云存储服务器，我们的应用程序后端无法获得 `title` 等业务对象字段，我们使用七牛云存储 API 的 [自定义变量](http://developer.qiniu.com/docs/v6/api/overview/up/response/vars.html#xvar) 和 [自定义响应内容](http://developer.qiniu.com/docs/v6/api/overview/up/response/response-body.html) 功能，通过七牛云存储上传 API 中转获得这些字段。

## 展示用户上传的图片

修改 `app/helpers/application_helper.rb`，添加 `qiniu_image_url` 方便生成图片 URL。为了保持简单我们直接硬编码了空间的域名：

```ruby
  def qiniu_image_url(post, format = :raw)
    url = "http://7xokus.com2.z0.glb.qiniucdn.com/#{post.qiniu_hash}"

    case format
    when :square
      url << '?imageView2/1/w/300/h/300/q/90'
    when :preview
      url << '?imageView2/2/w/1000/h/1000/q/90'
    when :raw
      url << "?attname=#{post.filename}"
    else
      url
    end
  end
```

修改 `app/views/posts/index.html.erb` 和 `app/views/posts/show.html.erb`，调用刚才创建的 URL helper 展示图片：

```erb
      <tr>
        <td><%= post.title %></td>
        <td><%= link_to image_tag(qiniu_image_url(post, :square), size: '300'), post %></td>
        <td><%= link_to 'Destroy', post, method: :delete, data: { confirm: 'Are you sure?' } %></td>
      </tr>
```

```erb
<p>
  <%= link_to image_tag(qiniu_image_url(@post, :preview)), qiniu_image_url(@post), class: 'image' %>
</p>

<%= link_to 'Back', posts_path %>
```

修改 `config/routes.rb`，将网站根目录设置为 posts 列表页面：

```ruby
  root 'posts#index'
```

访问 `http://localhost:3000`，现在我们可以看到刚才上传的图片。

##### 提示：

* 每个空间（bucket）内的资源都可以通过该空间的默认或自定义域名，加上文件 key 构造的 HTTP URL 进行访问。
* 可以在 URL 后增加特定查询参数，调用七牛云存储强大的 [数据处理（Fop）](http://developer.qiniu.com/docs/v6/api/overview/fop/) API，实时生成自定义格式的缩略图。
* 可以通过查询参数 `attname` 指定 URL 下载时使用的文件名。

## 简单的 UI 美化

修改 `app/views/posts/index.html.erb`，将原有的 table 布局改为 flexbox 布局：

```erb
<div class="posts">
  <% @posts.each do |post| %>
    <p>
      <%= link_to image_tag(qiniu_image_url(post, :square), size: '300'), post, class: 'image' %>
      <br>
      <%= post.title %>
      <br>
      <%= link_to 'Destroy', post, method: :delete, data: { confirm: 'Are you sure?' } %>
    </p>
  <% end %>
</div>
```

修改 `app/assets/stylesheets/application.css`，加入对应的 CSS：

```css
body {
  padding: 20px;
}

a.image:hover {
  background-color: transparent;
}

div.posts {
  display: flex;
  flex-flow: row wrap;
  justify-content: space-between;
}
```

访问 `http://localhost:3000`，图片列表看起来像一个正常的相册了。

## 添加用户账户功能

我们的应用程序还有两个明显的问题：没有记录图片是由谁分享的；任何人都可以删除 post。这对于一个社交应用来说显然是不能接受的问题。这对于一个社交应用来说显然是不可接受的，所以接下来我们将使用 Rails 社区流行的 Devise 组件直接获得用户注册、登录、验证子系统，实现图片发表者信息记录等功能。

修改 `Gemfile`，加入对 Devise 的依赖：

```ruby
gem 'devise'
```

执行以下命令安装 Devise，生成 `User` model，以及我们所需的 data migration：

```bash
bundle
rails generate devise:install
rails generate devise user
rails generate migration add_author_to_posts creator:belongs_to
rake db:migrate
```

修改 `app/controllers/posts_controller.rb`，对查看以外的操作要求登录，并在发表 post 时记录发表者身份：

```ruby
  before_action :authenticate_user!, except: [:index, :show]

  def create
    # ...
    @post = Post.new(
      title: upload_ret['title'],
      filename: upload_ret['fname'],
      qiniu_hash: upload_ret['hash'],
      creator: current_user
    )
    # ...
  end
```

修改 `app/models/post.rb`，添加与 `User` model 的从属关联：

```ruby
  belongs_to :creator, class_name: 'User'
```

修改 `app/views/layouts/application.html.erb`，添加登录状态信息和注销链接：

```erb
  <header>
    <% if user_signed_in? %>
      <p>
        Hello <%= current_user.email %>
      <%= link_to 'Logout', destroy_user_session_path, method: :delete %>
      </p>
    <% end %>
  </header>
```

修改 `app/views/posts/index.html.erb`，限制仅 post 发表者可以删除该 post：

```erb
      <% if user_signed_in? and post.creator == current_user %>
        <br>
        <%= link_to 'Destroy', post, method: :delete, data: { confirm: 'Are you sure?' } %>
      <% end %>
```

重启 `rails server` 后，访问 `http://localhost:3000`，现在用户需要注册登录才能发表图片了。

## 实现点赞功能

最后，做为一个社交应用，没有点赞功能怎么能让点赞狂魔们满足呢？！我们将添加一个 `like` scaffold 用来处理点赞和存储点赞信息。

执行以下命令生成 scaffold，`Like` model 将做为一个 join model 同时从属于 `Post` 和 `User`：

```bash
rails generate scaffold like post:belongs_to user:belongs_to
rake db:migrate
```

修改 `app/models/post.rb`，建立 `Post` 与 `Like` model 的“拥有／嵌套”关系：

```ruby
  has_many :likes
```

修改 `app/models/like.rb`，限制一个点赞狂魔对一个 post 只能点一次赞：

```ruby
  validates_uniqueness_of :user, scope: :post
```

修改 `config/routes.rb`，将 `likes` 设置为 `posts` 的嵌套资源：

```ruby
  resources :posts do
    # ...
    resources :likes
  end
```

修改 `app/views/posts/index.html.erb`，添加点赞信息显示以及 AJAX 点赞链接：

```erb
      <%= post.title %>
      <br>
      <%= content_tag(:span, "#{post.likes.count} likes", id: "post_#{post.id}_likes") %>

      <% if user_signed_in? %>
        <br>
        <%= link_to 'Like', post_likes_path(post), method: :post, remote: true  %>

        <% if post.creator == current_user %>
          <%= link_to 'Destroy', post, method: :delete, data: { confirm: 'Are you sure?' } %>
        <% end %>
      <% end %>
```

修改 `app/controllers/likes_controller.rb`，真正将 `likes` 资源实现为 `posts` 的嵌套资源，并在点赞时记录点赞狂魔的身份：

```ruby
  before_action :set_post

  def create
    @like = @post.likes.new(user: current_user)

    respond_to do |format|
      if @like.save
        format.js { }
      else
        format.js { }
      end
    end
  end

  private

    def set_post
      @post = Post.find(params[:post_id])
    end
```

创建 `app/views/likes/create.js.erb`，使用 Server-generated JavaScript 更新点赞信息：

```erb
$("#<%= "post_#{@post.id}_likes" %>").text("<%= "#{@post.likes.count} likes" %>")
```

重启 `rails server` 后，访问 `http://localhost:3000`，让我们愉快地点赞吧～

## 小结

虽然这只是一个简单的原型，但是因为使用了七牛云存储作为图片存储后端，我们的社交应用产品在一开始的原型阶段就拥有了能为大规模用户提供高速、可靠服务的潜能。

简单回顾一下我们刚才学习到的内容吧。使用 Rails 的代码生成功能， 基于 CRUD 结构的 scaffold 实现业务对象维护，以及业务操作非常高效；使用七牛云存储则可以轻松处理业务系统中的文件存储，获得图片视频等多媒体文件的云端处理能力，减少，甚至避免了部分研发和运维工作。希望这个视频可以帮助大家认识这两个优秀的开发工具，直观感受到它们的高效强大和简单易用。

## 参考资料

* [Ruby on Rails Guides](http://guides.rubyonrails.org)
* [七牛开发者中心](http://developer.qiniu.com)
