========================
5. Blog models and views
========================

Models
------

Since our stubs are in place we can start developing blog related code.

First lets start with models and implement some more methods that we will use 
in our views and templates.

Lets open **models.py** and import some helper modules to generate our slugs, 
add pagination, and print nice dates - they will all come from excellent 
webhelpers package - so the top of models.py should look have following imports 
added::

    from webhelpers2.text import urlify #<- will generate slugs
    from webhelpers2.date import time_ago_in_words #<- human friendly dates
    from paginate_sqlalchemy import SqlalchemyOrmPage #<- provides pagination

Now update our Entry model with following methods:
::

    @classmethod
    def all(cls):
        return DBSession.query(Entry).order_by(sa.desc(Entry.created))

This method will return query object that can return whole dataset to us when needed.

The query object will be sorting the rows  by date in descending order. 

::

    @classmethod
    def by_id(cls, id):
        return DBSession.query(Entry).filter(Entry.id == id).first()
    
This method will return a single blog entry by id, or None object if nothig is 
found. 
::
    
    @property
    def slug(self):
        return urlify(self.title)

This property of entry instance will return nice slugs for us to use in urls, 
title of "Foo Bar Baz" will become "Foo-Bar-Baz". Also non-latin characters will 
be approximated to their closest counterparts.
::

    @property
    def created_in_words(self):
        return time_ago_in_words(self.created)

This property will return information when specific entry was created in a 
friendly form like "2 days ago".
::

    @classmethod
    def get_paginator(cls, request, page=1):
        query = DBSession.query(Entry)
        query_params = request.GET.mixed()

        def url_maker(link_page):
            query_params['page'] = link_page
            return request.current_route_url(_query=query_params)

**get_paginator** method will return an excellent paginator that is able to 
return us only the entries from specific "page" of database resulteset. It will 
add LIMIT/OFFSET to our query based on items_per_page and current page number.

Paginator uses SqlalchemyOrmPage wrapper that will attempt to generate a paginator with links,
link urls will be constructed using `url_maker` function that uses request object to generate new url from current one
replacing page query param with new value.



Index view
----------

First lets add our Entry model to imports in views.py::

    from .models import (
        DBSession,
        User,
        Entry
        )

Now it's time to implement our actual index view::

    @view_config(route_name='home', renderer='pyramid_blogr:templates/index.mako')
    def index_page(request):
        page = int(request.params.get('page', 1))
        paginator = Entry.get_paginator(request, page)
        return {'paginator':paginator}
    
We first retrieve from url the page number we want to present to the user, 
if not present it defaults to 1.

The paginator object returned by *Entry.get_paginator* will then be used in 
template to build nice list of entries.

.. hint::
    Everything we return from our views in dictionaries will be available in 
    templates as variables. So if we return {'foo':1, 'bar':2} we will be able 
    to access the variables inside the template directly as *foo* and *bar*.  
  
Index view template
-------------------

For the purpose of this tutorial please use ready-made mako templates + some 
minimal page styling.

First delete everything in /templates folder.

We will now create layout.mako template file that will store a "master" 
template that other view templates will inherit from. This template will contain 
page header and footer shared by all pages.

In /templates please create "layout.mako" with following contents::

    <!DOCTYPE html>
    <html>
    <head>
    <meta charset="UTF-8">
    <title>Pyramid Blogr</title>
      <meta name="keywords" content="python web application" />
      <meta name="description" content="pyramid web application" />
      <link rel="shortcut icon" href="${request.static_url('pyramid_blogr:static/favicon.ico')}" />
      <link rel="stylesheet" href="${request.static_url('pyramid_blogr:static/pylons.css')}" type="text/css" media="screen" charset="utf-8" />
      <link rel="stylesheet" href="http://static.pylonsproject.org/fonts/nobile/stylesheet.css" media="screen" />
      <link rel="stylesheet" href="http://static.pylonsproject.org/fonts/neuton/stylesheet.css" media="screen" />
      <!--[if lte IE 6]>
      <link rel="stylesheet" href="${request.static_url('pyramid_blogr:static/ie6.css')}" type="text/css" media="screen" charset="utf-8" />
      <![endif]-->
    </head>
    <body>
    
    <div id="wrap">
        <div id="top">
          <div class="top align-center">
            <div><img src="${request.static_url('pyramid_blogr:static/pyramid.png')}" width="750" height="169" alt="pyramid"/></div>
          </div>
        </div>
        <div id="middle">
          <div class="middle align-center">
            <p class="app-welcome">
              Welcome to <span class="app-name">Pyramid blogr</span>, an application generated by<br/>
              the Pyramid web application development framework.
            </p>
          </div>
        </div>
        <div id="bottom" style="padding-bottom:50px;">
          <div class="bottom">
            ${next.body()}
          </div>
        </div>
      </div>
      <div id="footer">
        <div class="footer">Footer of our application.</div>
      </div>
    </body>
    </html>

.. hint::
    request object is always available inside your templates namespace

Inside your template you will notice that we used request.static_url method, 
that will generate correct links to your static assets, this is handy when 
building apps using URL prefixes.

In the middle of template you will also notice **${next.body()}** tag - 
after we render a template that inherits from our layout file - 
this is the place where our index template (or another for other view) will appear.

Now lets create another template called index.mako with following contents::

    <%inherit file="pyramid_blogr:templates/layout.mako"/>
    
    % if paginator.items:
    
        ${paginator.pager()}
        
        <h2>Blog entries</h2>
        
        <ul>
        % for entry in paginator.items:
        <li>
        <a href="${request.route_url('blog', id=entry.id, slug=entry.slug)}">
        ${entry.title}</a>
        </li>
        % endfor
        </ul>
        
        ${paginator.pager()}
    
    % else:
    
    <p>No blog entries found.</p>
    
    %endif
    
    <p><a href="${request.route_url('blog_action',action='create')}">
    Create a new blog entry</a></p>

This template inherits from layout.mako which means that it's contents will be 
wrapped by layout provided by parent template.

**${paginator.pager()}** - will print nice paginator links (it will only show up, 
if you have more than 5 blog entries in database)

**${request.route_url** - is used to generate links based on routes defined in 
our project. For example::

    ${request.route_url('blog_action',action='create')} -> /blog/create

Blog view
---------

Time to update our blog view.

At the top of views.py lets add following import::

    from pyramid.httpexceptions import HTTPNotFound, HTTPFound
    
Those exceptions will be used to perform redirects inside our apps.

* **HTTPFound** will return a 302  HTTP code response, it can accept *location* 
  argument that will add a Location: header for the browser - we will perform 
  redirects to other pages this way.  
* **HTTPNotFound** on other hand will just make the server serve a standard 404 
  response. 

::

    @view_config(route_name='blog', renderer='pyramid_blogr:templates/view_blog.mako')
    def blog_view(request):
        id = int(request.matchdict.get('id', -1))
        entry = Entry.by_id(id)
        if not entry:
            return HTTPNotFound()
        return {'entry':entry}

This view is also very simple, first we get the id variable from our 
route. It will be present in **matchdict** property of request object - all of 
our defined route arguments will end up there.

After we get entry id, that will be passed to Entry classmethod **by_id()** to 
fetch specific blog entry, if it's found - we return the db row for the 
template to use, otherwise we present user with standard 404 response.

Blog view template
-------------------

The template used for blog article presentation is named view_blog.mako::

    <%inherit file="pyramid_blogr:templates/layout.mako"/>
    
    <h1>${entry.title}</h1>
    <hr/>
    <p>${entry.body}</p>
    <hr/>
    <p>Created <strong title="${entry.created}">
    ${entry.created_in_words}</strong> ago</p>
    
    <p><a href="${request.route_url('home')}">Go Back</a> :: 
    <a href="${request.route_url('blog_action', action='edit', 
    _query=(('id',entry.id),))}">Edit Entry</a>
    
    </p>

The **_query** argument introduced here to url generator is a list of k,v tuples ,
that will be used to append GET(query) parameters, in our case it will be ?id=X. 

.. toctree::

   blog_models_and_views_src
