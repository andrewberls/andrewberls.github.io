---
layout: post
title: "How It's Made: Sutro Post/Tag Associations"
date: "2012-02-11"
permalink: /blog/post/sutro-post-tag-associations
---

<p>
	As promised last week, today is dedicated to delicious Rails! This post is the first of several in which I'll talk about how the various feature implementations in Sutro, the blog engine that runs this site. In particular, today will focus on what it means to add tags to articles, and how to associate the models underneath.
</p>

<h5>Models and Associations</h5>
<p>
	Rails associations are a set of methods that enable us to tie conceptually related models together and describe their relationships. For example, a Car belongs to a Person, and a Customer has many Orders. The association keywords "belongs to" and "has many" are several of the built in associations that Rails provides. We'll see that by specifying the relationships between two models (posts and tags here), we gain access to high-level helper methods that make our lives infinitely easier.
</p>
<p>
Our first step is the most important - determing the right relationship to use between posts and tags. The complete list of Rails model associations, as well as an explanation of their purpose and usage can be found at the RailsGuide <a href="http://guides.rubyonrails.org/association_basics.html">here</a>. So which ones are right for our needs? Well, clearly a post can have many tags. But remember, the relationship must be defined both ways. In that case, a <code>post has_many :tags</code> and a <code>tag belongs_to :post</code> relationship seems like the obvious choice. After some thought though, you'll realize that's not right. A post can have many tags, but a tag can also belong to many posts. For example, if a post is tagged with "ruby", the "ruby" tag is not locked to that post. Many posts can be tagged with the same name. So the association we're looking for is <code>has_and_belongs_to_many</code>. A post has and belongs to many tags, and a tag has and belongs to many posts. This is a mouthful, and gets a bit complicated to put in place, but ultimately, it allows us to call the methods <code>@post.tags</code> and <code>@tag.posts</code>, which will be enormously useful. Let's move on to generating our models and migrations!
</p>

<break />

<p>
Fortunately, our tag model is extremely simple. All we care about is the tag name. Let's generate a model to represent our tags - run <code>rails generate model tags</code> to create the tag model, as well a migration to initialize the corresponding table.
</p>

<pre><code><span class="label"># db/migrate/(timestamp)_create_tags.rb</span>
def change
  create_table :tags do |t|
    t.string "name"
    t.timestamps
  end
end
</code></pre>

<p>That's all we have to do in our migration. Make sure to run a <code>rake db:migrate</code> afterwards. Now let's write the actual associations between our two models!</p>

<pre><code><span class="label"># app/models/tag.rb</span>
class Tag &lt; ActiveRecord::Base
  has_and_belongs_to_many :posts
end

<span class="label"># app/models/post.rb</span>
class Post &lt; ActiveRecord::Base
  has_and_belongs_to_many :tags
end
</code></pre>

<p>
We're not quite done yet. If you were paying attention, you'll notice we haven't placed any foreign keys anywhere - nothing in either table references the other. For example, let's look at a customer/order relationship. A customer has many orders, and an order belongs to a single customer. The traditional approach would be to place a <code>customer_id</code> field in the Order table, so that an order now references a Customer by foreign key lookup. But a many-to-many relationship is more complicated. Since we defined that a post has and belongs to many tags, and that a tag has and belongs to many posts, we need an intermediate table to keep track of the associations called a join table. This is a table with two columns only - <code>post_id</code> and <code>tag_id</code>. The join table holds no data about either model - purely association data for lookups.
</p>

<h5>Creating the join table</h5>
<p>
While join tables may be a tricky concept to master in your head, the implementation is fortunately fairly simple and Rails handles most of the heavy lifting for us. For example, it's Rails convention to define the table name in terms of the two model names in alphabetical order. So, our table joining Posts and Tags will be called <code>posts_tags</code>. To implement it, all we have to do is generate the appropriate migration using <code>rails generate migration CreatePostTagJoinTable</code>.
</p>

<pre><code><span class="label"># db/migrate/(timestamp)_create_post_tag_join_table.rb</span>
def change
  create_table :posts_tags, :id => false do |t|
    t.integer :post_id
    t.integer :tag_id
  end
end
</code></pre>

<p>
Note that we're specifying <code>:id => false</code> here. Rails automatically adds an id primary key to tables, but since this a pure join table we don't need it. Be sure to run a <code>rake db:migrate</code> again!
</p>

<p>
That's all we need to worry about in terms of migrations. Now we have to create an interface for our tags, and the ability to actually add a tag to a post.
</p>

<h5>has_and_belongs_to_many or has_many :through?</h5>
<p>
Before I continue, I want to explain a design decision here. If you have experience with Rails, you'll know that we had the option of doing has_many :through instead of has_and_belongs_to_many. For those unfamiliar with the association, has_many :through creates an intermediary model for the relationship. For example, consider a system that tracks appointments for physicians and patients. A physician has many patients, and a patient (potentially) has many physicians, but we need information about the relationship itself as well, such as the appointment time. In this case, we'd want to create the intermediary model Appointments and change our associations so that a <code>physician has_many :patients, :through => :appointments</code> and a <code>patient has_many :physicians, :through => :appointments</code>. Again, Appointments acts as a join table tracking the relationship, but now we have access to other information about the relationship itself. Other tutorials I've seen have implemented the post-tag association using has_many :through and created an intermediary model called Taggings. This doesn't appeal to me - a direct relationship seems to make more sense, so I've chosen to implement it using has_and_belongs_to_many. For more information, I highly recommend the <a href="http://guides.rubyonrails.org/association_basics.html">RailsGuide on Associations</a> or the <a href="http://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html">ActiveRecord Associations ClassMethods Documentation</a>.

<h5>Adding tags to posts</h5>
<p>
Before writing out any code, it's critical to sit back and imagine how you want the end result to work. I think of it in terms of a story - here, "when writing a post, I want to input the tags as a comma-separated list and have Rails do the rest of the work". This is possible, but takes a little bit of extra work on our part since we're no longer doing direct input as in a post title, for example. Let's write the corresponding code in our view first.
</p>

<pre><code><span class="label"># app/views/admin/posts/new.html.erb</span>
&lt;%= form_for(:post, :url=&gt;{:action =&gt; 'create'}) do |f| %&gt;	
  &lt;%= f.label :title %&gt;
  &lt;%= f.text_field :title %&gt;

  &lt;%= f.label :body, "Post" %&gt;
  &lt;%= f.text_area :body %&gt;

  &lt;%= f.label :tag_list, "Tags (optional)" %&gt;
  &lt;%= f.text_field :tag_list %&gt;	

  &lt;%= f.submit "Publish" %&gt;			
  &lt;%= link_to "Cancel", dashboard_path %&gt;
&lt;% end %&gt;
</code></pre>

<p>
At this point, you're like to run into an error if you go to the page - after all, our Post model has no attribute <code>tag_list</code>. We have to define methods to accept this tag list and transform it into a list of Tags associated with this post. Let's go into <code>post.rb</code> and define the appropriate methods.
</p>

<pre><code><span class="label"># app/models/post.rb</span>
def tag_list
  self.tags.map {|t| t.name }.join(", ")
end
</code></pre>

<p>
With that defined, our <code>new</code> view displays fine, and our <code>edit</code> view displays the tags in a nicely comma separated list. But this doesn't actually associate any tags with a post yet. To do that, we have to define the special <code>tag_list=</code> method back in <code>post.rb</code>.
</p>

<pre><code><span class="label"># app/models/post.rb</span>
def tag_list=(tag_list)
  self.tags.clear <span class="comment"># For the update method, just in case we're changing tags</span>
  
  <span class="comment"># Split the tags into an array, strip whitespace , and convert to lowercase</span>
  tags = tag_list.split(",").collect{|s| s.strip.downcase}
 
  <span class="comment"># For each tag, find or create by name, and associate with the post</span>
  tags.each do |tag_name|
    tag = Tag.find_or_create_by_name(tag_name)
    tag.name = tag_name      
    self.tags &lt;&lt; tag <span class="comment"># Append the tag to the post</span>
  end
end
</code></pre>

<p>
We're making progress! We can now create posts with associated tags in a successful many to many relationship. Of course, we can't stop here. Tags aren't useful unless you have the ability to filter posts by their tags.
</p>

<h5>Filtering posts by tag</h5>
<p>
While it might seem like a good idea to generate a separate controller to handle our tags, I'm going to forgo that step. For now, it's somewhat of a lazy design decision, but we don't need that level of  functionality and instead can just build our filtering into our post controller's list action. Let's do routes first.
</p>

<pre><code><span class="label"># config/routes.rb</span>
match "blog/tag/:tag"  => "posts#list", :as => "tag"
</code></pre>

<p>
That gives us a simple named path that matches any tag string, such as <code>/blog/tag/ruby</code> and dispatches it to the list action on the posts controller. In the post controller itself, all we have to do is add a little bit of logic to determine whether or not we're filtering by a tag or just looking at the main blog index.
</p>

<pre><code><span class="label"># app/controllers/posts_controller.rb</span>
def list
    
  if !params[:tag].nil? <span class="comment"># Are we looking at a tag?</span>
    @tag = Tag.find_by_name(params[:tag])
    if @tag.nil?
      <span class="comment"># Redirect if user is trying to look at a non-existent tag</span>
      redirect_to blog_path
    else           
      @posts = @tag.posts.order("id DESC")
    end
  else
    <span class="comment"># No tag - display posts as normal</span>      
    @posts = Post.all.order("id DESC")
  end
                     
end
</code></pre>

<p>The actual Sutro implementation is slightly different (I check for status and do some pagination), but I've left those off for brevity. The full controller source is available on GitHub <a href="https://github.com/andrewberls/andrewberls/blob/master/app/controllers/posts_controller.rb">here</a>.</p>

<p>Since we're lumping this in with the existing action, let's add a notice in the view so there's some indication that you're viewing by tag. My solution, although simple, was this:</p>

<pre><code><span class="label"># app/views/posts/list.html.erb</span>
&lt;% unless @tag.nil? %&gt;
  &lt;h1&gt;Tagged: &lt;%= @tag.name %&gt;&lt;/h1&gt;
&lt;% end %&gt;
</code></pre>

<p>This creates an h1 at the top of the page displaying the current tag (if it exists)</p>

<h5>Finishing touches</h5>
<p>At this point, we've done all the real work for our tag implementation and now we're just working on presentational stuff. To close, this is how the tag 'pills' are displayed at the bottom of each post. A method called <code>render_tags</code> is defined in a helper that takes an array of Tag objects (as outputted by @post.tags or similar), and outputs a raw HTML string containing formatted links to each tag's respective path.</p>

<pre><code><span class="label"># app/helpers/post_helper.rb</span>
def render_tags(tag_list)
  raw tags.map { |t| link_to(t.name, tag_path(t.name), :class=> "tag") }
    .join("")
end
</code></pre>

<p>Note that the Sutro implementation differs in that I use the same method for the public 'pill' display, and the backend dashboard which is a comma-separated listing. The full code is available <a href="https://github.com/andrewberls/andrewberls/blob/master/app/helpers/posts_helper.rb">here</a>.</p>

<p>And we're done! We've succesfully created a many to many association between two models, the ability to enter tags with a post, and format them in a nice display. Rails provides powerful helpers to work with complex associations, and this is only a peek at what's possible. If you enjoyed what you read, or want to chide me on my lazy coding habits, make sure to inform me in the comments. Until next time!</p>
