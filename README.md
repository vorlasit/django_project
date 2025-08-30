# django_project
install python and virtualenv

    sudo apt update && sudo apt install python3 python3-virtualenv
Installing PostgreSQL

    sudo apt install postgresql postgresql-contrib
    sudo systemctl status postgresql
    sudo systemctl start postgresql
Setting up a PostgreSQL database 

    sudo su postgres
    createuser --createdb --username postgres --no-createrole --superuser --pwprompt postgresuser
    password postgrespassword

    createdb KCDB

    SHOW hba_file;
will show 
 /etc/postgresql/14/main/pg_hba.conf

    sudo nano /etc/postgresql/14/main/pg_hba.conf
Search for the following line in the file:
    local   all             all                                     peer
    
  And change it to:

    local   all             all                                     md5
Save and close the file, then restart the PostgreSQL server:

    sudo systemctl restart postgresql 
Setting up your Django project

    virtualenv env && source env/bin/activate
    pip install Django psycopg2-binary
create a new Django project called recipe_project

    django-admin startproject recipe_project
    cd recipe_project
Connecting to the database
 create a .env file
   .env
   
    DB_ENGINE=django.db.backends.postgresql
    DB_NAME=recipes
    DB_USER=<project_user>
    DB_PASSWORD=<secure_password>
    DB_HOST=localhost
    DB_PORT=5432
Return to your terminal and install this package with pip:

    pip install python-decouple
Afterwards, open the recipe_project/settings.py file as follows:

    code recipe_project/settings.py
Import the python-decouple package near the top of the file by adding the following highlighted line:
recipe_project/settings.py
    
    from pathlib import Path
    from decouple import config
Locate the DATABASES section in the settings.py file, and update it as shown below, then save and close the file:
  recipe_project/settings.py

    . . .
    DATABASES = {
        'default': {
            'ENGINE': config('DB_ENGINE'),
            'NAME': config('DB_NAME'),
            'USER': config('DB_USER'),
            'PASSWORD': config('DB_PASSWORD'),
            'HOST': config('DB_HOST'),
            'PORT': config('DB_PORT'),
        }
    }
    . . .
You can do this by applying the default Django migrations with run manager.py
 
    import os
    import sys 
    def main():
        """Run administrative tasks."""
        os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'kcc_project.settings')
        try:
            from django.core.management import execute_from_command_line
            # add start from this
            import django
            from django.core.management import call_command
            django.setup() 
            # ทำ migrations อัตโนมัติ
            call_command('makemigrations', interactive=False)
            call_command('migrate', interactive=False)
            # end from this
        except ImportError as exc:
            raise ImportError(
                "Couldn't import Django. Are you sure it's installed and "
                "available on your PYTHONPATH environment variable? Did you "
                "forget to activate a virtual environment?"
            ) from exc
        execute_from_command_line(sys.argv)
    
    
    if __name__ == '__main__':
        main()
Creating the Django application models

    python manage.py startapp recipe
Next, open the project/settings.py file, and add the newly created app to the INSTALLED_APPS list as shown below:

    . . .
    INSTALLED_APPS = [
        . . .
        'recipe', 
    ]
    . . .

Once registered, you can now create the model for a recipe by editing the recipe/models.py file using the code below. This model defines fields for the recipe name, ingredients, and instructions:
  recipe/models.py
  
    from django.db import models
    
    class Recipe(models.Model):
        name = models.CharField(max_length=200)
        ingredients = models.TextField()
        instructions = models.TextField()
    
        def __str__(self):
            return self.name
run and magation

    python3 manage.py runserver 0.0.0.0:8000
Adding recipes
Let's start by creating the form to handle recipe creation in a new recipe/forms.py file:
  recipe/forms.py
  
    from django import forms
    from .models import Recipe
    
    class RecipeForm(forms.ModelForm):
        class Meta:
            model = Recipe
            fields = ['name', 'ingredients', 'instructions']
Next, you will create a new view for processing the request to add a new recipe to the database. Open the recipe/views.py file and paste in the following code:
  recipe/views.py

    from django.shortcuts import render, redirect
    from .forms import RecipeForm
    from .models import Recipe
    
    def add_recipe(request):
        if request.method == 'POST':
            form = RecipeForm(request.POST)
            if form.is_valid():
                form.save()
                return redirect('list_recipes')
        else:
            form = RecipeForm()
        return render(request, 'recipe/add_recipe.html', {'form': form})
Next, set up an endpoint for the view you just created. Open the project/urls.py file, and add the highlighted lines below:
  project/urls.py
  
    from django.contrib import admin
    from django.urls import path
    from recipe import views # add this
    
    
    urlpatterns = [
        path('admin/', admin.site.urls),
        path('recipes/add/', views.add_recipe, name='add_recipe'), # add this
    ]
Go ahead and create the following two files: recipe/templates/recipe/base.html and recipe/templates/recipe/add_recipe.html:

    code recipe/templates/recipe/base.html
  recipe/templates/recipe/base.html
  
    <!DOCTYPE html>
    <html>
      <head>
        <title>Recipe Management System</title>
      </head>
      <body>
        <header>
          <h1>Recipe Management System</h1>
        </header>
    
        <main>
          {% block content %}
          <!-- This block will be overridden by content from other templates -->
          {% endblock %}
        </main>
    
        <footer></footer>
      </body>
    </html>

  Then do the same for add_recipe.html:
    
    code recipe/templates/recipe/add_recipe.html
  recipe/templates/recipe/add_recipe.html

    {% extends 'recipe/base.html' %}
    {% block content %}
    <h2>Add Recipe</h2>
    <form method="POST">
      {% csrf_token %} {{ form.as_p }}
      <button type="submit">Add</button>
    </form>
    {% endblock %}

run and magation 

    python3 manage.py runserver 0.0.0.0:8000
Listing recipes
Start by updating the code in your views.py file with the following function:
  recipe/views.py

    . . . 
    def list_recipes(request):
        recipes = Recipe.objects.all() # retrieve all the recipes
        return render(request, 'recipe/list_recipes.html', {'recipes': recipes})
Next, create the route that will render the view you just created. You can do this by including the following in urlpatterns list in the urls.py file:
project/urls.py
    
    . . . 
    urlpatterns = [
        path("admin/", admin.site.urls),
        path("recipes/add/", views.add_recipe, name="add_recipe"),
        path('recipes/', views.list_recipes, name='list_recipes'), # add this line 
    ]
Finally, create the recipe/templates/recipe/list_recipes.html file and populate it with the following code:
    
    code recipe/templates/recipe/list_recipes.html
  recipe/templates/recipe/list_recipes.html
  
    {% extends 'recipe/base.html' %}
    {% block content %}
    <h2>Recipe List</h2>
    <ul>
      {% for recipe in recipes %}
      <li>{{ recipe.name }}</li>
      {% empty %}
      <li>No recipes found.</li>
      {% endfor %}
    </ul>
    {% endblock %}
    
Updating recipes
handler in your views.py file with the following code:
  
  recipe/views.py
  
    . . .
    def update_recipe(request, recipe_id):
        recipe = Recipe.objects.get(pk=recipe_id)
        if request.method == 'POST':
            form = RecipeForm(request.POST, instance=recipe)
            if form.is_valid():
                form.save()
                return redirect('list_recipes')
        else:
            form = RecipeForm(instance=recipe)
        return render(request, 'recipe/update_recipe.html', {'form': form})

Go ahead and add the endpoint to render the update_recipe view in your urls.py file:

    urlpatterns = [
    path("admin/", admin.site.urls),
    path("recipes/add/", views.add_recipe, name="add_recipe"),
    path('recipes/', views.list_recipes, name='list_recipes'),
    path('recipes/update/<int:recipe_id>/', views.update_recipe, name='update_recipe'), # add this
    ]

Finally, add the Django template that displays the form for updating a recipe:
  recipe/templates/recipe/update_recipe.html
  
    {% extends 'recipe/base.html' %}
    {% block content %}
    <h2>Update Recipe</h2>
    <form method="POST">
      {% csrf_token %} {{ form.as_p }}
      <button type="submit">Update</button>
    </form>
    {% endblock %}

Deleting recipes
You can do this by updating the contents in your views.py file with the following code:
  recipe/views.py

    from django.contrib import messages
    
    . . .
    
    def delete_recipe(request, recipe_id):
        recipe = Recipe.objects.get(pk=recipe_id)
        recipe.delete()
        messages.success(request, 'Recipe deleted successfully.')
        return redirect('list_recipes')

Next, add the endpoint for deleting a recipe as follows: 
  project/urls.py 
  
    urlpatterns = [
        path("admin/", admin.site.urls),
        path("recipes/add/", views.add_recipe, name="add_recipe"),
        path("recipes/", views.list_recipes, name="list_recipes"),
        path("recipes/update/<int:recipe_id>/", views.update_recipe, name="update_recipe"),
        path("recipes/delete/<int:recipe_id>/", views.delete_recipe, name="delete_recipe"),
    ]
Searching or filtering recipes
To implement this in Django, we will update the Recipe model with the following code: 
  recipe/models.py

    . . .
    class Recipe(models.Model):
       ...
    
        class Meta:
            indexes = [models.Index(fields=['name', 'ingredients','instructions'])]


run and magation 

    python3 manage.py runserver 0.0.0.0:8000
it in a case-insensitive manner with the search query. Finally, it renders the recipe/search_recipes.html template, passing the search results (recipes) and the query itself (query) as context variables
  
  recipe/views.py

    . . . 
    def search_recipes(request):
        query = request.GET.get('q')
        recipes = Recipe.objects.filter(name__icontains=query)
        return render(request, 'recipe/search_recipes.html', {'recipes': recipes, 'que
For the form, create a new file recipe/templates/recipe/search_form.html and paste the following code.
recipe/templates/recipe/search_form.html
 
    <form method="GET" action="{% url 'search_recipes' %}">
      <input type="text" name="q" placeholder="Search" />
      <button type="submit">Search</button>
    </form>
Now add this to base.html so that it can be viewed on all pages in the application:

  [recipe/templates/recipe/base.html]
  
    . . .
    <main>
       {% include 'recipe/search_form.html' %}
    
       {% block content %}
      <!-- This block will be overridden by content from other templates -->
       {% endblock %}
    </main>
    . . .
Next, you will create the template to display the search results. Create a new file at recipe/templates/recipe/search_recipes.html and paste the following code in it:
  recipe/templates/recipe/search_recipes.html
  
    {% extends 'recipe/base.html' %}
    {% block content %}
    <h2>Search Results for "{{ query }}"</h2>
    <ul>
      {% for recipe in recipes %}
      <li>{{ recipe.name }}</li>
      {% empty %}
      <li>No recipes found.</li>
      {% endfor %}
    </ul>
    {% endblock %}

To round off this section, add the endpoint for handling search queries. You can do this by including the following in urlpatterns list in the urls.py file:
    project/urls.py
    
    . . . 
    urlpatterns = [
        path("admin/", admin.site.urls),
        path("recipes/add/", views.add_recipe, name="add_recipe"),
        path("recipes/", views.list_recipes, name="list_recipes"),
        path("recipes/update/<int:recipe_id>/", views.update_recipe, name="update_recipe"),
        path("recipes/delete/<int:recipe_id>/", views.delete_recipe, name="delete_recipe"),
        path("recipes/search/", views.search_recipes, name="search_recipes"),
    ]
