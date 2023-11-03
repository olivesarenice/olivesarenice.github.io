---
title: Flask Application Factory Notes
tag: software
category: learning
---

## Intro

Right after deploying my basic Flask app and writing about it, I went to read up further on various example of Flask apps. I came across this article which talks about a proper way to structure Flask apps for scalability - Flask Application Factory. It is an enlightening read as it demonstrates how proper app structure can make development much clearer and extensible.

See the full post here: [Demystifying Flask’s "Application Factory"](https://hackersandslackers.com/flask-application-factory/)

These are my notes on the article.

I will attempt to apply this to the previous to-do list Flask app that I made.

## Notes

A proper Flask app should have this project structure:

    /app
    ├── /application        <-- holds the entire app contents
    │   ├── __init__.py     <-- initialises the application context by importing from all the other files below
    │   ├── auth.py
    │   ├── forms.py
    │   ├── models.py
    │   ├── routes.py       
    │   ├── /static
    │   └── /templates
    ├── config.py
    └── wsgi.py             <--- points the WSGI server to the entry point (which is application/__init__.py)

`__init__.py` is the one doing the heavy lifting as it brings all the separate parts of our app together into the `app_context`.

An example of an app that requires a database, authentication, and admin functionality would have an `__init__.py` file that resembles this structure:


    from flask import Flask
    from flask_sqlalchemy import SQLAlchemy
    from flask_redis import FlaskRedis

    #############################
    ### MAP TO GLOBAL PLUGINS ###
    #############################

    db = SQLAlchemy()
    r = FlaskRedis()


    ##################
    ### CREATE APP ###
    ##################

    def init_app():
        app = Flask(__name__, instance_relative_config=False)
        app.config.from_object('config.Config')

        ##########################
        ### INITIALISE PLUGINS ###
        ##########################

        db.init_app(app)
        r.init_app(app)

        #########################
        ### BUILD APP CONTEXT ###
        #########################
        
        with app.app_context():

            from . import routes # This is the base custom coded parts of the app

            app.register_blueprint(auth.auth_bp) # These are additional templated functions we want to add to the app 
            app.register_blueprint(admin.admin_bp)

            return app

Note: Blueprints are pieces of code that, when initiated by the app, create re-useable functionality. It allows larger functions to be broken down into smaller one and stacked into the main app. 

See this [discussion](https://stackoverflow.com/questions/24420857/what-are-flask-blueprints-exactly) for more info.

A blueprint typically looks like this:

    ```__init__.py

    import my_module.part_1
    import my_other_module.part_2

    app = Flask(__name__)

    app.register_blueprint(part_1)
    app.register_blueprint(part_2)

    ```

    ```my_module.py
    from flask import Blueprint
    part_1 = Blueprint(part_1)

    @part_1.route('/url')
    def function()
        return view
    ```

So now it is clear that the functionality associated with `part_1` is different from that of `part_2`.


And then in `wsgi.py`, we tell whatever server is running our app to look for the `__init__.py` file as the entry point:

    from application import init_app
    app = init_app()

    if __name__ == "__main__":
        app.run(host='0.0.0.0')


## Rewriting the basic Flask app

Previously, I [wrote](https://olivesarenice.github.io/posts/deploy-flask-app-from-scratch) a basic Flask app that uses `app.py` without any inherent structure. Now, I applied the Application Factory method to that same app and produced the repo [here](https://github.com/olivesarenice/flask-application-factory).

It was basic re-factoring and most of my issues came from dealing with circular imports due to the multiple files requiring the same variables.

Additional resources:

- [The Flask Mega-Tutorial Part XV: A Better Application Structure](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xv-a-better-application-structure)

Some things I noted:

1. It wasn't explicitly stated in the tutorial, but `models.py` should include `from . import db` because the model needs to reference the existing database instance when defining the database model

2. `db.create_all()` should be called within the `app_context()` section of `__init__.py`, after the imports have been done. This creates the database tables.

3. Same import issue for `routes.py`. Since the decorators reference `@app.route()`, we need to import the existing `app` instance that is in the process of being generated. importing `app` itself won't work because `app` doesn't get returned until the end of the `init_app()` function, while the `routes.py` module gets imported in the middle of `init_app()`. Hence, we need to use `from flask import current_app as app` which will tell Flask to take the `app` in the midst of config and pass it to `routes.py` as needed.

Also, since `routes.py` calls the database model class `myList`, we need to add `from .models import myList, db` too. 

That's all

