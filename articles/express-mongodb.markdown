Title: Blog rolling with mongoDB, express and Node.js
Author: Ciaran Jessup
Date: Thu Feb 18 2010 21:28:42 GMT+0000 (UTC)

In this article I hope to take you through the steps required to get a fully-functional (albeit feature-light) persistent blogging system running on top of [node][].

The technology stack that we'll be using will be [node][] + [express][] + [mongoDB][] all of which are exciting, fast and highly scalable. You'll also get to use [haml-js][] and [sass.js][] for driving the templated views and styling!

This article will be fairly in-depth so you may want to get yourself a rather large mug of whatever beverage you prefer before you settle down :)

## Getting Started / Pre-Requisites
Before we start with the code it's important to make sure you have [git][] installed, an up-to-date installation of [node][] and a running [mongodb][] server on your development environment.

###git###
Whilst not a mandatory requirement I will be using git to pull down code from various location. Sadly the installation, configuration and setup of git is way beyond the scope of this article and I recommend reading up on that before embarking on this article if you're not familiar with git!

###mongoDB###
Installation is as simple as downloading the [installer from here][]. For this tutorial I've been using v1.2.2 on MacOSX but any recent version should work. Once installed you can just execute 'mongod' to have a local instance up and running.

###node.js###
I'll assume that you already have an installed version of node.js (why else would you be looking at a how-to?! ;) ) However as [node][] is subject to a reasonably high rate of change for the purposes of this article everything has been written to run against the ['v0.1.28' tag][].
  
## Getting hold of express
To create our 'blog' we'll work with the default express repository locally, to do this please perform the following commands in your console:

    #!sh
    git clone git://github.com/visionmedia/express.git
    cd express
    git checkout 347c8847b0a8d9322c5555fa809b6213f49cd187 -b our_blog_branch
    git submodule init
    git submodule update
    
The checkout is just to make sure we get a version of express that works with the previously mentioned tag of node! Before we go any further lets just test that your node installation is fully working. To do this execute the following command inside the express folder:

    #!sh
    node examples/chat/app.js

If everything is going well then you should in your console see the following:

    Express started at http://localhost:3000/ in development mode

If you now browse to [localhost:3000][] with your web browser you will find a fully working long-polling / AJAX example chat server, exciting n'est pas?

If you now kill the running node process that we started a moment ago we can begin with the process of writing our blog, let the good times (blog)roll.

## Defining our application ##

We're going to build a very simple blogging application (perhaps we'll build on this in a future article?). It is going to support the reading of blog articles, posting of blog articles and commenting on them. There will be no security, authentication or authorisation. Hopefully this will demonstrate enough of the technology stack to let you move forward quickly.

### The data types ###

Because we're dealing with a [document orientated][] database rather than a [relational][] database we don't need to worry about what 'tables' we will need to persist this data to the database or the relationships between records within those tables. In fact we only have 1 datatype in the application at all, the article:

    {
      _id: 0,
      title: '',
      body: '',
      comments: [{
        person: '',
        comment: '',
        created_at: new Date()
      }],
      created_at: new Date()
    }

There are plenty of other document configurations we could've gone for but this will provide us with a good foundation (for example there's no notion of article authors.). Also the `created_at` field could most probably be omitted as the default 'Primary Key factory' that the mongo-db-native client that we're using generates time-based object-ids, but for simplicity we'll go with a 'proper' date.

> It should be noted that one oft-reported issue with mongoDB is the size of the data on the disk. As we're dealing with a [document orientated][] database each and every record stores all the field-names with the data so there is no re-use. This means that it can often be more space-efficient to have properties such as 't', or 'b' rather than 'title' or 'body', however for fear of confusion I would avoid this unless truly required!

### The operations ###

There is a discrete set of operations (or things we want to achieve) that fall within the scope of this article, they are (in the order that we will tackle them):

1. Create a new article.
2. Show the list of all the articles.
3. Show an individual article and its comments.
4. Comment on an article
 
Now that we know what we're trying to achieve lets try and achieve that goal in a step-by-step fashion.

### From small acorns do giant oak trees grow ###

_Well alright, fairly small blogging apps can grow!_

In express a 'normal' application consists of a call to `configure`, followed by a series of method calls that declare `routes` and what happens to requests that match the route followed by a call to `run`.

Thus one of the simplest express applications could be written as follows:

    require.paths.unshift('lib');
    require('express');
    require('express/plugins');

    configure(function(){
      use(MethodOverride);
      use(ContentLength);
      use(CommonLogger);
      set('root', __dirname);
    })
    
    get('/', function(){
      this.halt(200, "Hello World!");
    })

    run();

The above code declares a single `route` that operates on `GET` requests to the address `/` from the browser and will just return a simple (non-HTML) text string and a response code of 200 to the client.

Convention seems to be to place this code into a file named `app.js` at the root of your express folder. If you do this and then execute it:

    #!sh
    node app.js
    
When you browse to [localhost:3000][] you should see that old favourite 'Hello World!'. This file is the starting point of the blogging application and we shall build on it now :)

## A chapter in which we build on our humble beginnings ##

Now that we have a fully working web server we should probably look at doing something with it. In this section we will learn how to use [Haml][] to render our data and create forms to post the data back to the server, initially we will store this in memory.

The layout of express applications is fairly familiar and is usually of the form:

    express                 /* The cloned express folder     */
    |-- app.js              /* The application code itself   */
    |-- lib                 /* Third-party dependencies      */
    |-- public              /* Publicly accessible resources */
    |   |-- images
    |   `-- javascript
    `-- views               /* The templates for the 'views' */

Please take a moment to create the folders that you require, these will need creating:

    #!sh
    mkdir public
    mkdir public/javascript
    mkdir public/images
    mkdir views
    
###Of providers and data###

Because the intention of this article is to show how one might use a persistent approach in node.js we shall start with an abstraction: provider. These 'providers' are going to responsible for returning and updating the data. Initially we'll create a dummy in-memory version just to bootstrap us up and running, but then we'll move over to using a real persistence layer without changing the calling code.

    // articleprovider-memory.js
    var articleCounter = 1;
    
    ArticleProvider = function(){};
    ArticleProvider.prototype.dummyData = [];

    ArticleProvider.prototype.findAll = function() {
      var promise = new process.Promise();
      promise.emitSuccess( this.dummyData );
      return promise;
    };

    ArticleProvider.prototype.findById = function(id) {
      var promise = new process.Promise(),
          result = null;
      for(var i =0;i<this.dummyData.length;i++) {
        if( this.dummyData[i]._id == id ) {
          result = this.dummyData[i];
          break;
        }
      }
      promise.emitSuccess( result );
      return promise;
    };

    ArticleProvider.prototype.save = function(articles) {
      var promise = new process.Promise(),
          article = null;

      if( typeof(articles.length)=="undefined")
        articles = [articles];
  
      for( var i =0;i< articles.length;i++ ) {
        article = articles[i];
        article._id = articleCounter++;
        article.created_at = new Date();
        
        if( article.comments === undefined )
          article.comments = [];
        
        for(var j =0;j< article.comments.length; j++) {
          article.comments[j].created_at = new Date();
        }
        this.dummyData[this.dummyData.length]= article;
      }
      promise.emitSuccess();
      return promise;
    };

    /* Lets bootstrap with dummy data */
    new ArticleProvider().save([
      {title: 'Post one', body: 'Body one', comments:[{author:'Bob', comment:'I love it'}, {author:'Dave', comment:'This is rubbish!'}]},
      {title: 'Post two', body: 'Body two'},
      {title: 'Post three', body: 'Body three'}
    ]);

    exports.ArticleProvider = ArticleProvider;

If the above code is saved to a file named `articleprovider-memory.js` in the same folder as the `app.js` we created earlier and `app.js` is modified to look as follows:

    require.paths.unshift('lib');
    require('express');
    require('express/plugins');
    var ArticleProvider = require('./articleprovider-memory').ArticleProvider;

    configure(function(){
      use(MethodOverride);
      use(ContentLength);
      use(CommonLogger);
      set('root', __dirname);
    });

    function getArticleProvider() {
      return new ArticleProvider();
    };

    get('/', function(){
      var self = this;
      getArticleProvider().findAll().addCallback(function (articles) {
        self.halt( 200, require('sys').inspect(articles) );
      });
    });

    run();

If the app is re-run and you browse to [localhost:3000][] you will see the object structure of 3 blog posts that the memory provider starts off with, magic!

###A view to a kill###

Now we have a way of reading and storing data (patience, memory is only the beginning!) we'll want a way of displaying and creating the data properly. Initially we'll start by just providing an index view of all the blog articles. To do this create the following two files in your views sub-directory (be very careful about the indentation, that first lines should be up against the left-hand margin!):

    -# views/layout.haml.html
    #!haml
    %html
      %head
        %title = title
      %body
        #wrapper = body
    
    -# views/blogs_index.haml.html
    #!haml
    %h1 = title
    #articles
      :each article in articles
        %div.article
          %div.created_at = article.created_at
          %div.title = article.title
          %div.body = article.body
                                     
Next change your `get('/')` routing rule in your `app.js` to be as follows:

    get('/', function(){
      var self = this;
      getArticleProvider().findAll().addCallback(function(docs) {
        self.render('blogs_index.haml.html', {
          locals: {
            title: 'Blog',
            articles: docs
          }
        });
      });
    });

Now you should be able to restart the server and browser to [localhost:3000][]. Et voila! We'll not win any design awards, but you should now see a list of 3 very 'functional' blog postings (don't worry we'll come back to the style in a moment).

There are two important things to note that we've just done;

The first is the change to our application's routing rules. What we've done is say that for any browser requests that come in to the route ('/') we should ask the data provider for all the articles it knows about (a future improvement might be 'the most recent 10 posts etc.') and to 'render' those returned articles using the [haml-js][] template `blogs_index.haml.html`.

The second is the usage of a 'layout' [haml-js][] file `layout.haml.html`. This file will be used whenever a call to 'render' is made (unless over-ridden in that particular call) and provides a simple mechanism for common style across all page requests.

> If you're familiar with [Haml][] then you may want to skip this section, otherwise please read-on! [Haml][] is yet-another templating language, however this one is driven by the rule that 'Markup should be beautiful'. It provides a lightweight syntax for declaring markup with a bare minimum of typed characters.
> 
> [haml-js][] is a server-side JavaScript partial/mostly-complete implementation of [Haml][]. Reading a [haml-js][] template is simple. The hierarchy of elements is expressed as indentation on the left hand-side; that is, everything that starts in a given column shares the same parent. Each line of [Haml][] represents either a new element in the (eventual) HTML document or a function within [Haml][] (which offers conditions and loops etc). Effectively [haml-js][] takes a JSON object and binds it to any `literal` text in the [haml-js][] template, applies the rules that define [Haml][] and then processes the resulting bag of stuff to produce a well-formed and valid HTML document of the specified DOCTYPE. (Yay!)

As is probably obvious we need a little styling to be applied here, to do that we'll need to change our layout a little to request a stylesheet, add a new rule to service this request and add a Sass template to the `views` folder in order to generate the css.

    -# views/layout.haml.html

    #!haml
    %html
      %head
        %title = title
        %link{ rel: 'stylesheet', href: '/style.css' }
      %body
        #wrapper = body
    
    // views/style.sass.css
    
    #!haml
    body
      :font-family "Helvetica Neue", "Lucida Grande", "Arial"
      :font-size 13px
      :text-align center
      =text-stroke 1px rgba(255, 255, 255, 0.1)
      :color #555
    h1, h2
      :margin 0
      :font-size 22px
      :color #343434
    h1
      :text-shadow 1px 2px 2px #ddd
      :font-size 60px
    #articles
      :text-align left
      :margin-left auto
      :margin-right auto
      :width 320px
      .article
        :margin 20px
        .created_at
          :display none
        .title
          :font-weight bold
          :text-decoration underline
          :background-color #eee
        .body
          :background-color #ffa

And add a new route to app.js:

    get('/*.css', function(file){
      this.render(file + '.sass.css', { layout: false });
    });

Again after restarting your app and browsing to [localhost:3000][] you should see the posts, with a little more style (grantedly not much more!).

A couple of things to notice here:

1. We setup a route to handle the request for the css as a regular expression match. We then used the matched url segment to load a Sass file from the available views and `express` dynamically converts the Sass into CSS for us on the fly. In a production environment there are configuration options that can be passed to the `configure` method to make sure these views are cached but during development its rather useful to be able to change your Sass on the fly :)
2. As express is treating the sass to CSS rendering in exactly the same manner as the Haml to HTML rendering we need to suppress the default `layout` behaviour as there is no meaningful layout here for our Sass.

> Sass is to CSS as Haml is to HTML. However reading Sass can be a little more complex as the hierarchy that is being described is really individual selectors. Lines that start at the same column and begin with a `:` are rules, these rules are applied to the hierarchy that they're found under, for example in the above Sass example the bottom most line of Sass `:background-color #ffa` is equivalent to the CSS `#articles .article .body {background-color: #ffa;}` this equivalence is due to the position of the start of this line relative to its parent lines :) (Easy really!)


###Great, so how do I make my first post?###

Now we can view a list of blog posts it would be nice to have a simple form for making new posts and being re-directed back to the new list. To achieve this we'll need a new view (to let us create a post) and two new routes (one to accept the post data, the other to return the form).

    -# views/blog_new.haml.html

    #!haml
    %h1 = title
    %form{ method: 'post' }
      %div
        %div
          %span Title :
          %input{ type: 'text', name: 'title', id: 'editArticleTitle' }
        %div
          %span Body :
          %textarea{name: 'body', rows: 20, id: 'editArticleBody' }
        %div#editArticleSubmit
          %input{ type: 'submit', value: 'Send' }

Add two new routes to app.js

    get('/blog/new', function(){
      this.render('blog_new.haml.html', {
        locals: {
          title: 'New Post'
        }
      });
    });

    post('/blog/new', function(){
      var self = this;
      getArticleProvider().save({
        title: this.param('title'),
        body: this.param('body')
      }).addCallback(function(docs) {
        self.redirect('/')
      });
    });
    
Upon restarting your app if you browse to [new post][] you will be able to create new blog articles, awesome! Looking at the post route we can see that upon successfully saving we redirect back to the index page where all the articles are displayed.

If I've lost you along the way you can get a patch of this fully working (but non-persisting) blog here: [Checkpoint 1][]. This patch should apply cleanly to the previously described SHA of express :)

### Adding permanent persistence to the mix ###

I promised that by the end of this article we'd be persisting our data across restarts of node, I've not yet delivered on this promise but now I will ..hopefully ;)

To do this we need to install a dependency on [node-mongodb-native][], which will allow our burgeoning application to access [mongoDB][]. If we open the console up and enter the `express` directory we created earlier we will be able to type the following commands to install the driver.

    #!sh
    git submodule add git://github.com/christkv/node-mongodb-native.git lib/support/mongodb
    cd lib/support/mongodb
    git checkout a0392fb6095aefdb8889dc8a269a36ee957e27a4

Now we need to replace our old memory based data provider with one thats capable of using mongodb, this will also require a minor change to `app.js` to use the replacement provider.

    // articleprovider-mongodb.js

    require.paths.unshift("./lib/support/mongodb/lib");
    var mongo = require("mongodb/db");
    process.mixin(mongo, require('mongodb/connection'));
    var ObjectID = require('mongodb/bson/bson').ObjectID;

    ArticleProvider = function(host, port) {
      this.connected = false;
      this.db = new mongo.Db('node-mongo-blog', new mongo.Server(host, port, {auto_reconnect: true}, {}));
    };


    ArticleProvider.prototype.findAll = function() {
      var promise = new process.Promise();
      this.db.open(function(db) {
        db.collection(function(article_collection) {
          article_collection.find(function(cursor) {
            cursor.toArray(function(results) {
              promise.emitSuccess( results );
            });
          });
        }, 'articles');
      });
      return promise;
    };

    ArticleProvider.prototype.findById = function(id) {
      var promise = new process.Promise();
      this.db.open(function(db) {
        db.collection(function(article_collection) {
          article_collection.findOne(function(result) {
            promise.emitSuccess(result);
          }, {_id: ObjectID.createFromHexString(id)});
        }, 'articles');
      });
      return promise;
    };

    ArticleProvider.prototype.save = function(articles) {
      var promise = new process.Promise()
      this.db.open(function(db) {
        db.collection(function(article_collection) {
          if( typeof(articles.length)=="undefined")
            articles = [articles];

          for( var i =0;i< articles.length;i++ ) {
            article = articles[i];
            article.created_at = new Date();
            if( article.comments === undefined ) article.comments = [];
            for(var j =0;j< article.comments.length; j++) {
              article.comments[j].created_at = new Date();
            }
          }

          article_collection.insert(articles, function() {
            promise.emitSuccess();
          });
        }, 'articles');
      });

      return promise;
    };

    exports.ArticleProvider = ArticleProvider;


    // app.js

    require.paths.unshift('lib');
    require('express');
    require('express/plugins');
    var ArticleProvider = require('./articleprovider-mongodb').ArticleProvider;

    configure(function(){
      use(MethodOverride);
      use(ContentLength);
      use(CommonLogger);
      set('root', __dirname);
    });

    function getArticleProvider() {
      return new ArticleProvider('localhost', 27017);
    }
            
    get('/', function(){
      var self = this;
      getArticleProvider().findAll().addCallback(function(docs) {
        self.render('blogs_index.haml.html', {
          locals: {
            title: 'Blog',
            articles: docs
          }
        });
      });
    });

    get('/blog/new', function(){
      this.render('blog_new.haml.html', {
        locals: {
          title: 'New Post'
        }
      });
    });

    post('/blog/new', function(){
      var self = this;
      getArticleProvider().save({
        title: this.param('title'),
        body: this.param('body')
      }).addCallback(function(docs) {
        self.redirect('/')
      });
    })
    
    get('/*.css', function(file){
      this.render(file + '.sass.css', { layout: false })
    });
    
    run();

As you can see we had to make only the smallest of changes to move away from a temporary in-memory JSON store to the fully persistent and highly scalable mongoDB store.

Let us pause for a second to take a look at one of the methods we've just written to access [mongoDB][]:

    ArticleProvider.prototype.findById = function(id) {
      var promise = new process.Promise();
      this.db.open(function(db) {
        db.collection(function(article_collection) {
          article_collection.findOne(function(result) {
            promise.emitSuccess(result);
          }, {_id: ObjectID.createFromHexString(id)});
        }, 'articles');
      });
      return promise;
    };

It is perhaps not immediately obvious what is happening here, as there are a *lot* of different things going on <g>. I'll try to describe each line:
    
1. Declares the `findById` method on the provider's `prototype`. This method is going to take in one argument the `id` of the article we wish to retrieve and it is going to return a promise. (A promise is a data-structure that exists in versions of [node][] prior to 0.1.30, it allows for handling synchronous and asynchronous events equivalently in the calling code.)
2. Creates the promise that will be returned to the calling code
3. Opens the [mongoDB][] connection asynchronously and passes in a callback function that will be executed (and passed a reference to this connection) once it has been opened.
4. In mongoDB there are no tables as such (hence schema-less) but there are `collections`. A `collection` seems to be a logical grouping of similar documents, but there appears to be no real constraint on what types of document is put in these `collections`. For our purpose we will have a single `collection` called `articles`. By calling `collection` on the `db` object and passing in our collection name `articles` and a callback to deal with the response mongoDb will quietly create the collection from scratch and return it if there wasn't a collection of that name already or it will just return a reference to an existing connection. (This behaviour can actually be controlled by configuring mongoDB to be `strict`.)
5. The `collection` type that is passed in the callback from the previous call to `db.collection(...)` exposes various methods for manipulating the data stored inside of the database. Here we've chosen to use `findOne` which when given some criteria to search on will return the sole record that matches those criteria, but there are others such as `find` which returns a `cursor` that can be iterated over etc.
6. Now we have the record we searched for in the database we tell the promise we created back at the start that we're done so any associated callbacks that the promise has are executed (in our case this callback would do the page rendering.)
7. This is the `specification` argument (think `criteria` or `WHERE clause`) used by the `findOne` method that started on line 5, here we're using the passed in `id` (which is a hexadecimal string ultimately coming from the browser so needs to be converted to the real type that our `_id` fields are being stored as.) It basically states 'Find me the document in the collection who has a property named `_id` and a value equivalent to an `ObjectId` constructed with the passed in hexadecimal string.
8. This is the `collection` argument (think `FROM clause`) used by the `collection` method that started on line 4. This tells mongoDB which set of documents we care about.
9. Meh! some brackets and stuff :)
10. Returns the promise. (The code above may or may not have executed yet, but when the calling code attaches its callback it will be notified with the results that were emitted on line 6)

I hope this explains a little better what is now going on inside our new provider code.
    

## Adding comments

We're about halfway through the set of (4) operations we defined earlier but you'll be pleased to know that we've completed the majority of the work, everything from here on in is just minor improvements :)

Just to re-cap we've done:

* Create a new article.
* Show the list of all the articles.

and we still need to do:

* Show an individual article and its comments.
* Comment on an article.

So, lets crack on!

### Showing an individual article and its comments

Displaying an individual article isn't much different to displaying one of the articles within the list of articles that we've already done, so we'll pinch some of that template. In addition to displaying the title and body though we will also want to render all the existing comments and provide a form for readers to add their own comment.

We'll also need a new route to allow the article to be referenced by a URL and we'll need to tweak the rendered list so our titles on the list can now be hyperlinks to the real article's own page.

> One thing that we should touch on here is [surrogate][] vs [natural][] keys. It seems that with [document orientated][] databases it is encouraged where possible to use [natural][] keys however in this case we've not got any sensible one to use (_unless you fancy title to be unique enough_.)
> 
> Normally this wouldn't be that much of an issue as [surrogate][] keys are usually fairly sane things like auto-incremented integers, unfortunately the *default* primary key provider that we're using generates universally unique (and universally opaque) binary objects / large numbers in byte arrays. These 'numbers' don't really translate well into HTML so we need to use some utility methods on the `ObjectId` class to translate to and from a hex-string into the id that can located on the database.

    -# views/blogs_index.haml.html

    %h1 = title
    #articles
      :each article in articles
        %div.article
          %div.created_at = article.created_at
          %div.title
            %a{href:"/blog/"+article._id.toHexString()}= article.title
          %div.body = article.body

      
    -# views/blog_show.haml.html

      %h1 = title
      %div.article
        %div.created_at = article.created_at
        %div.title = article.title
        %div.body = article.body
        :each comment in article.comments
          %div.comment
            %div.person = comment.person
            %div.comment = comment.comment
        %div
          %form{ method: 'post', action:"/blog/addComment" }
            %input{type: "hidden", name:"_id", value: article._id.toHexString()}
            %div
              %span Author :
              %input{ type: 'text', name: 'person', id: 'addCommentPerson' }
            %div
              %span Comment :
              %textarea{name: 'comment', rows: 5, id: 'addCommentComment' }
            %div#editArticleSubmit
              %input{ type: 'submit', value: 'Send' }


    // views/style.sass.css

    body
      :font-family "Helvetica Neue", "Lucida Grande", "Arial"
      :font-size 13px
      :text-align center
      =text-stroke 1px rgba(255, 255, 255, 0.1)
      :color #555
    h1, h2
      :margin 0
      :font-size 22px
      :color #343434
    h1
      :text-shadow 1px 2px 2px #ddd
      :font-size 60px
    #articles
      :text-align left
      :margin-left auto
      :margin-right auto
      :width 320px
      .article
        :margin 20px
        .created_at
          :display none
        .title
          :font-weight bold
          :text-decoration underline
          :background-color #eee
        .body
          :background-color #ffa
    #article
      .created_at
        :display none
      input[type =text]
        :width 490px
        :margin-left 16px
      input[type =button]
        :text-align left
        :margin-left 440px
      textarea
        :width 490px
        :height 90px

We also need to add a new rule to `app.js` for serving these view requests:

    get('/blog/*', function(id){
      var self = this;
      getArticleProvider().findById(id).addCallback(function(article) {
        self.render('blog_show.haml.html', {
          locals: {
            title: article.title,
            article:article
          }
        });
      });
    });

Now if you browse to [localhost:3000][] the previous articles you added (click [new post] to create a new one if you have none) should now be visible (apologies for the lack of style once again!) The titles of these articles are now hyperlinks to individual pages that display the article in all itself original glory, comments and all (but alas no comments have been added so far.)

### Comment on an article ###

Commenting on an article is a simple extension upon everything we've already gone through, the only minor complexity is in the style of 'update' we use on the back-end.

Transactions are largely non-existent in mongoDB but there are several approaches to achieving atomicity in certain scenarios. For comment addition we're going to use a '$push' update that allows us to add an element to the end of an array property of an existing document atomically    (which is absolutely perfect for our needs!)

All the views/stylesheet changes we need were made in the last set of changes but we need to add in a new route to handle the POST and a method to our provider to make the change on the persistent store:

    // app.js

    post('/blog/addComment', function() {
      var self = this;
      getArticleProvider().addCommentToArticle(this.param('_id'), {
        person: self.param('person'),
        comment: self.param('comment'),
        created_at: new Date()
      }).addCallback(function(docs) {
        self.redirect('/blog/' + self.param('_id'))
      });
    });


    // articleprovider-mongodb.js
    
    ArticleProvider.prototype.addCommentToArticle = function(articleId, comment) {
      var promise = new process.Promise();
      this.db.open(function(db) {
        db.collection(function(article_collection) {
          article_collection.update(function(article){
              promise.emitSuccess();
            },
            {_id: ObjectID.createFromHexString(articleId)},
            {"$push": {comments: comment}});
        }, 'articles');
      });
      return promise;
    };

After restarting and browsing to a blog article (any one will do) you should now be able to add comments to your articles ad-infinitum. How easy was that?

If you've made it to this point without any issues then congratulations! Otherwise this [Checkpoint 2][] patch should contain all the code as I have it now!

## Where next ##

Clearly this blogging application is very rough and ready (there is no style to speak of for starters) but there are several clear directions that it could take, depending on feedback I'll either leave these as exercises for the reader or provide additional tutorials over time:

 * Markup language support (HTML, Markdown etc. in the posts and comments)
 * Security, authentication etc.
 * An administrative interface
 * Multiple blog-support.
 * Decent styling <g> (inc. themes)

I hope this helps at least someone out there get to grips with how you might start actually writing web apps with [node][], [express][] and [mongoDB][].

Good luck! :)

__Fin__.

[git]: http://git-scm.com/
[node]: http://nodejs.org
[express]: http://github.com/visionmedia/express
[mongoDB]: http://www.mongodb.org
[installer from here]: http://www.mongodb.org/display/DOCS/Downloads
['v0.1.28' tag]: http://github.com/ry/node/tree/v0.1.28
[localhost:3000]: http://localhost:3000
[new post]: http://localhost:3000/blog/new
[document orientated]: http://en.wikipedia.org/wiki/Document-oriented_database
[relational]: http://en.wikipedia.org/wiki/Relational_database_management_system
[haml-js]: http://github.com/creationix/haml-js
[Haml]: http://haml-lang.com/
[sass.js]: http://github.com/visionmedia/sass.js
[node-mongodb-native]: http://github.com/christkv/node-mongodb-native
[surrogate]: http://en.wikipedia.org/wiki/Surrogate_key
[natural]: http://en.wikipedia.org/wiki/Natural_key
[Checkpoint 1]: http://github.com/creationix/howtonode.org/tree/master/articles/express-mongodb/0001-Checkpoint-1.patch
[Checkpoint 2]: http://github.com/creationix/howtonode.org/tree/master/articles/express-mongodb/0001-Checkpoint-2.patch