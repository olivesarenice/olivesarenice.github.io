---
title: Deploy Flask App from Scratch
tag: software
category: projects
---

{% include img-wrap name="app.png" caption="My Flask App" %}

## Summary

This is how I learned to write and deploy a basic Flask app (almost) for free on [Digital Ocean](https://www.digitalocean.com/)'s App Platform.

The live app can be found at: https://monkfish-app-cjxym.ondigitalocean.app/

In this writeup, I cover the components and concepts of a Flask app, and the options for deploying them outside of self-hosting.

I had zero knowledge of Flask apps before this, hence why I wanted to learn for myself.

The resources I used:

- App code: [Learn Flask for Python - Full Tutorial](https://www.youtube.com/watch?v=Z1RJmh_OqeA) 
- Deploy: [How to deploy a Flask app to Digital Ocean's app platform](https://dev.to/ajot/how-to-deploy-a-flask-app-to-digital-oceans-app-platform-goc)



## What is Flask?

Flask is a microframework for web applications written specifically in the Python language. 

The next question: What is a microframework/ web framework?

Web frameworks are templates that web applications can be built on. Web applications requires multiple components to interact with each other: server, database, front-end HTML, back-end programming logic, etc. For example, the LAMP stack - Linux (OS), Apache (server), MySQL (database), PHP (programming logic) is one such set of components used to build web apps. While all web applications can be built from scratch, it is much easier to use a template that will reduce time spent on configuring these components to work together.

Other Python web frameworks are: Django (more complex, more components) and Bottle (super basic framework, no dependencies needed, runs on a single file)
Similarly, there are web frameworks for other languages, such as Ruby on Rails for the Ruby language.

The reason I started learning about Flask is also because I wanted to learn to deploy Dash applications which use the Flask framework. Maybe this will be a future post.

Thus, compare a web app to a vehicle - a thing that transports people around, and Flask to a style of vehicle - car, bus, plane. All cars are built of similar components in standardized configurations, although their specifics differ from car to car.

## A basic Flask web app

The most basic Flask app requires just Python and the Flask library installed in your working environment. I used conda and created the `venv` [directly in my working folder](https://stackoverflow.com/questions/37926940/how-to-specify-new-environment-location-for-conda-create) rather than the central conda `env` folder.

    (Inside the working directory)

    conda create env -p=ENV_NAME python=3.11
    conda activate ENV_NAME

    pip install Flask

From there, create a new file called `app.py`. This is what Flask will execute to run the app.

    from flask import Flask
    app = Flask(__name__)
    
    @app.route('/hello')
    def say_hello():
        return 'Hello!'
    
    if __name__ == '__main__':
        app.run()

Upon running these 6 lines of code, you will be able to access the URL `127.0.0.1:5000/hello` and see `Hello!` in your browser.

Rather than copy-paste the code while following the tutorial, I wanted to know what each line does.

### Decorators

Learning about how Flask `@app.route("/")` actually works sent me down a rabbithole of learning about **decorators**.

First, I needed to understand what the decorators (@...) actually do. 

The purpose of decorators is to **modify the behaviour** of another function without having to touch the source code of said function. This is similar to how wrappers works. The difference is that `wrapper_name(function_name())` needs to be called everytime we want to get the modified behaviour. On the otherhand, decorators are applied from the beginning when the function is defined -  that's why the `@decorator_name` appears right above the function declaration statement, and not during execution.

Usage of decorators are not limited to Flask, a basic example of decorator usage:

    def decorator(func):
        def wrapper():
            print("Something is happening before the function is called.")
            func()
            print("Something is happening after the function is called.")
        return wrapper

    @decorator
    def say_hello():
        print("Hello!")

    say_hello()


The decorator has 2 parts:
- The `decorator()` function which when called will take in a function `func` and modify its behaviour.
- The `@decorator` syntax which says which functions `decorator()` should be applied to.

Thus, whenever `say_hello()` is called, it will now be calling `decorator(say_hello)`. The inner `wrapper()` specifies what to do with `say_hello` function subsequently. In this example, it will print extra text before and after `say_hello` is run.

### Back to Flask:

    from flask import Flask
    app = Flask(__name__)
    
    @app.route('/hello')
    def say_hello():
        return 'Hello!'
    
    if __name__ == '__main__':
        app.run()

In the case of Flask, the `@app.route()` decorator is referencing the `route` function within the `app` object that was created with the `Flask` command.

So why does requesting a HTTP GET to `http://flask_app_url/hello` trigger the `say_hello` command?

Unlike the basic example, `route` decorator doesnt actually modify the behaviour of `say_hello` function. 

It simply takes the string `/hello` and add its to the Flask `app` object. `app` contains a list of reachable HTTP endpoints which is how the application knows what to execute when it receives a request at an endpoint. Thus, `/hello` endpoint is mapped to the function `say_hello` which will be called when this endpoint is reached.

This is described in the Flask documentation for `route`:

    def route(self, rule, **options):
            """A decorator that is used to register a view function for a
            given URL rule.  This does the same thing as :meth:`add_url_rule`
            but is intended for decorator usage::
                @app.route('/')
                def index():
                    return 'Hello World'
            For more information refer to :ref:`url-route-registrations`.
            :param rule: the URL rule as string
            :param endpoint: the endpoint for the registered URL rule.  Flask
                            itself assumes the name of the view function as
                            endpoint
            :param options: the options to be forwarded to the underlying
                            :class:`~werkzeug.routing.Rule` object.  A change
                            to Werkzeug is handling of method options.  methods
                            is a list of methods this rule should be limited
                            to (``GET``, ``POST`` etc.).  By default a rule
                            just listens for ``GET`` (and implicitly ``HEAD``).
                            Starting with Flask 0.6, ``OPTIONS`` is implicitly
                            added and handled by the standard request handling.
            """
            def decorator(f):
                endpoint = options.pop('endpoint', None)
                self.add_url_rule(rule, endpoint, f, **options)
                return f
            return decorator

This is how Flask knows what functions to call when requests are made. 

Looking at the code for `route`, we see that it is using the `app`'s `add_url_rule` method under the hood.
In fact, we could achieve the same thing without `route` decorators with the following code:

    app = Flask(_name_)

    def say_hello():
        return "Hello!"

    app.add_url_rule("/", "hello", say_hello)

In this way, it is much clearer that we are telling `app` to execute `say_hello` when the user requests `127.0.0.1:5000/hello`.

## A proper Flask app

This portion follows the [tutorial](https://www.youtube.com/watch?v=Z1RJmh_OqeA) exactly.

### Plan

Create a to-do list web app with the following workflow:
- User visits webpage and views a HTML table containing some text content
- User can add new rows to the table
- User can delete rows from table
- User can edit text content and have it updated

### Setup

Since this is a web application, create the app's directories to follow that of a web server:

    |--- env (contains all the Python dependencies needed to run the app)
    |--- static
        |--- css
            |--- main.css (contains some boilerplate CSS)
        |--- imgs
    |    
    |--- templates (contains the `.html` files which are rendered for the user)
        |--- HTML FILES HERE
    |
    |---app.py

### Templating in HTML with Jinja2

As part of the framework, Flask uses jinja2 templating engine which allows us to generate templates for `.html` pages. We can tell child `.html` templates which parent `.html`s they should inherit content from.

Jinja2 syntax allows us to:
- create a base template and have other pages inherit from this template.
- dynamically replace content using template syntax like {% raw %}`{{ this_thing_will_be_turned_into_a_string }}`{% endraw %} and other control logic

### Steps

We need a main page that the user will land in.

In `templates`, create the `index.html` and `base.html` .

Our `index` page (and any other pages) will inherit its structure from the `base` file. This makes it so that we don't have to rewrite the HTML structure each time.

The parent `base.html` contains boilerplate HTML with 2 dynamically generated/ templatized items:

{% raw %}
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <title>My Website</title>
        <link rel="stylesheet" href="{{ url_for('static',filename='css/main.css') }}">
        <link rel="icon" href="./favicon.ico" type="image/x-icon">
        {% block head %}{% endblock %}
    </head>
    <body>
        {% block body %}{% endblock %}
    </html>
{% endraw %}

1. Dynamic Blocks
Wherever we want to add content, we can use {% raw %}`{% block BLOCK_NAME %}`{% endraw %} followed by {% raw %}`{% endblock %}`{% endraw %}. We will reference them in child templates later.

2. Dynamic strings
We can also dynamically generate strings based on functions and then pass them to HTML.

To generate the `href` to our `main.css` file rather than hardcoding it, we can use:

    href="{{ url_for('static',filename='css/main.css') }}"

`{{ variable }}` turns anything inside it into a string.

`url_for` is a built-in Flask function that generates a URL to a specified `app` endpoint along with parameters for that endpoint (if required). 

But wait, we didn't specify `/static` as an endpoint in `app.py`, how does it know this endpoint?

`/static` folder just happens to be a builtin endpoint.

For more details, we can look at `app.url_map` which is the `url_map` attribute of our `app`. By default, it contains the rule:

    Map([<Rule '/static/<filename>' (HEAD, GET, OPTIONS) -> static>])

*Note that endpoints containing `<some_param>` are endpoints that can take in parameters*

i.e. accessing static endpoint will return the string `/static/<filename>` where filename will be replaced with whatever parameter was sent to the endpoint. Hence we set the parameter as the path to the file inside /static directory `filename='css/main.css'`.

Now create the child template `index.html` to reference the template `base.html`

{% raw %}
    {% extends 'base.html'%} // use base.html as the template
    {% block head %} // define the content for the 'head' block
    <h1> Some Header... </h1>
    {% endblock %} 
    {% block body %} // define the content for the 'body' block
    Some text
    {% endblock %}
{% endraw %}

### Databases

Next, we need a database to store user inputs so that they persist from session to session.

Flask does this easily by using the SQL-Alchemy object-relational mapping (ORM). Unlike retrieving data from a database using SQL in the CLI or in a database management client, we need to interact with the database using Python code. This is why we use ORMs to "align programming code with database structures".

Before proceeding:

    pip install Flask-SQLAlchemy

We will be using SQLite database and our database table will be called `myList` and contain the columns:

- id: PK integer
- content: the text supplied by user
- date_created: the current time when the row is added

Inside `app.py`, the code should look like this:

    from flask import Flask
    from flask_sqlalchemy import SQLAlchemy
    from datetime import datetime 

    app = Flask(__name__)

    ######## SETUP THE DATABASE ########
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///test.db'  #
    db = SQLAlchemy(app)

    class myList(db.Model):
        id = db.Column(db.Integer, primary_key=True)
        content = db.Column(db.String(128), nullable = False)
        date_created = db.Column(db.DateTime, default=datetime.utcnow)


    with app.app_context():
        db.create_all()

    ######## REST OF THE APP ########
    ...

1. Tell the app that we want to add a sqlite db resource

        app.config['SQLALCHEMY_DATABSE_URI'] = 'sqlite:///test.db'

2. Command SQLAlchmey to create the database mapping to that resource

        db = SQLAlchemy(app)

3. Create the [db.Model](https://docs.sqlalchemy.org/en/20/orm/quickstart.html#declare-models) object for our needs

        class myList(db.Model):
            id = db.Column(db.Integer, primary_key=True)
            content = db.Column(db.String(128), nullable = False)
            date_created = db.Column(db.DateTime, default=datetime.utcnow)

4. Create a database instance for the app. A `.db` file should be created in `instance` folder when this is run

        with app.app_context():
            db.create_all()

    

Note that `db.Model` needs to be specified **before** the database creation, otherwise no tables will be created. See this [issue](https://stackoverflow.com/questions/74171824/db-create-all-doesnt-create-a-database-in-a-desired-directory).

## Rest of the app (owl)

After understanding the HTML template setup and database initialisation stage, building the rest of the app depends very much on basic Python coding.

This portion is better understood through the [video tutorial]() as it builds the code incrementally. As my purpose is to note down the conceptual parts of Flask components, I won't go into the code logic here.

## Deployment

After the app is built and tested locally, I needed to deploy it somewhere publicly accessible.

This section differs from the video as I didn't want to use the paid services of Heroku. I used DigitalOcean's App Platform instead and followed this [tutorial](https://dev.to/ajot/how-to-deploy-a-flask-app-to-digital-oceans-app-platform-goc) exactly.

If the app is able to run locally in the virtual environment, it should have no problem deploying.

Do note that when pushing the repo to Github, you can also add your `env` folder to `.gitignore`. The cloud platform builds environment from `requirements.txt` anyway, so you don't need to supply the dependencies.

Also, when using `gunicorn` server for deployment, you will need to tell it where the actual Flask code is located. This is why we need to add the line:

    gunicorn --worker-tmp-dir /dev/shm app:app

In DigitalOcean, this will be in App > Settings > App Spec:

{% include img-wrap name="do.png" caption="Configuring gunicorn" %}

The actual command to deploy a gunicorn server is 

    gunicorn --worker-tmp-dir /dev/shm <app-name>.wsgi

`app:app` means `app.py` is the file name to run. and `:app` assigns the server to the variable `app`.

If your `app.py` file is named something else like `main.py`, you can use `main:app` instead.