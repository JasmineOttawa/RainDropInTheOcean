---
layout: post
title: Build a web application using Django web framework - part 3 
category: Python
tags: [Python]
---
views, reference  https://docs.djangoproject.com/en/2.1/intro/tutorial03/  

# 1, Evolve a view

step1, Follow view has hard-coded design
```
def index(request)
    latest_question_list=Question.objects.order_by('-pub_date')[:5]
	output=', '.join([q.question_text for q in latest_question_list])
	return HttpResPonse(output)
```

step2, Use template system to seperate form design from view
```
#polls/templates/polls/index.html
{% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
        <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
{% else %}
    <p>No polls are available.</p>
{% endif %}

#in polls/views.py, loads the template called polls/index.html and passes it a context. The context is a dictionary mapping template variable names to Python objects.
```
from django.http import HttpResponse
from django.template import loader

from .models import Question

def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    template = loader.get_template('polls/index.html')
    context = {
        'latest_question_list': latest_question_list,
    }
    return HttpResponse(template.render(context, request))
```

step3, a shortcut to load a template, fill a context and return an HttpResponse object with the rest of the rendered template. 
The render() function takes the request object as its first argument, a template name as its second argument and a dictionary as its optional third argument. It returns an HttpResponse object of the given template rendered with the given context.
```
from django.shortcuts import render
from .models import Question

def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    context = {'latest_question_list': latest_question_list}
    return render(request, 'polls/index.html', context)
```

#2, Raising a 404 error 
Look at the page that displays the question text for a given poll
```
def detail(request, question_id):
    try:
        question = Question.objects.get(pk=question_id)
    except Question.DoesNotExist:
        raise Http404("Question does not exist")
    return render(request, 'polls/detail.html', {'question': question})
```

Put another template file for details page 
```
polls/templates/polls/detail.html
{{ question }}
```

a shortcut for get_object_or_404()
```
from django.shortcuts import get_object_or_404, render
...
def detail(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, 'polls/detail.html', {'question': question})
```
The get_object_or_404() function takes a Django model as its first argument and an arbitrary number of keyword arguments, which it passes to the get() function of the model’s manager. It raises Http404 if the object doesn’t exist.   
There’s also a get_list_or_404() function, which works just as get_object_or_404() – except using filter() instead of get(). It raises Http404 if the list is empty.   
Why do we use a helper function get_object_or_404() instead of automatically catching the ObjectDoesNotExist exceptions at a higher level, or having the model API raise Http404 instead of ObjectDoesNotExist?   
Because that would couple the model layer to the view layer. One of the foremost design goals of Django is to maintain loose coupling. Some controlled coupling is introduced in the django.shortcuts module.   

# 3, use the template system
```
#polls/tempaltes/polls/detail.html 
<h1>{{ question.question_text }}</h1>
<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }}</li>
{% endfor %}
</ul>
```
In the example of {{ question.question_text }}, first Django does a dictionary lookup on the object question. Failing that, it tries an attribute lookup – which works, in this case. If attribute lookup had failed, it would’ve tried a list-index lookup.
https://docs.djangoproject.com/en/2.1/topics/templates/

# 4, removing hardcoded URLs in templates 
in index.html, there is:
```
<li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
```
The link is partially hardcoded. We can remove the reliance on specific URL paths defined in your URL by using the {% url %} template tag: 
```
<li><a href="{% url 'detail' question.id %}">{{ question.question_text }}</a></li>
```
As in urls.py, the detail path is defined as: path('<int:question_id>/', views.detail, name='detail'),   
Now if you change the URL of the polls detail view to something else, like following, it will still work   
path('specifics/<int:question_id>/', views.detail, name='detail'),  

# 5, Namespacing URL names
There might be other apps in same project that has a detail view, when using {% url %} tamplate tag, how does Django knows which app it is for? 
The answer is to add namespace to your URLconf, add app_name=... 
```
#polls/urls.py
app_name = 'polls'
urlpatterns = [
    path('', views.index, name='index'),
    path('<int:question_id>/', views.detail, name='detail'),
    path('<int:question_id>/results/', views.results, name='results'),
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```
In index.html 
```
<li><a href="{% url 'polls:detail' question.id %}">{{ question.question_text }}</a></li>
```

# 6, write a simple form 
```
#polls/templates/polls/detail.html
<h1>{{ question.question_text }}</h1>

{% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}

<form action="{% url 'polls:vote' question.id %}" method="post">
{% csrf_token %}
{% for choice in question.choice_set.all %}
    <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}">
    <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br>
{% endfor %}
<input type="submit" value="Vote">
</form>
```
+ <form action="{% url 'polls:vote' question.id %}" method="post">  
  post will submit data to server side    
+ template tag : {% csrf_token %}    
  for POST form, we need to worry about CSRF - Cross Site Request Forgeries, Django comes with a ease-to-use system for protecting against it.   

Create view function to handle data submitted in this form 
```
def vote(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    try:
        selected_choice = question.choice_set.get(pk=request.POST['choice'])
    except (KeyError, Choice.DoesNotExist):
        # Redisplay the question voting form.
        return render(request, 'polls/detail.html', {
            'question': question,
            'error_message': "You didn't select a choice.",
        })
    else:
        selected_choice.votes += 1
        selected_choice.save()
        # Always return an HttpResponseRedirect after successfully dealing with POST data
		# This prevents data from being posted twice if a user hits the Back button.
        return HttpResponseRedirect(reverse('polls:results', args=(question.id,)))
```
+ request.POST, a dictionary-like object to transfer data from form to view, 
  request.GET access GET data in the same way 
+ request.POST['choice'] raise KeyError if choice wasn't provided in POST data 
+ HttpResponseRedirect takes a single argument: the URL to which the user will be redirected
+ Always return an HttpResponseRedirect after successfully dealing with POST data. This prevents data from being posted twice if a user hits the Back button.
+ reverse(), helps avoid having to hardcode a URL in the view function. It is given the name of the view that we want to pass control to and the variable portion of the URL pattern that points to that view.
+ request and response :  https://docs.djangoproject.com/en/2.1/ref/request-response/ 

the results view it will be directed to:
```
def results(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, 'polls/results.html', {'question': question})
```
This is almost same as the detail view, the only difference is the template name. this redundancy could be addressed by generic view. 
Template for results.html:
```
<h1>{{ question.question_text }}</h1>

<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
{% endfor %}
</ul>

<a href="{% url 'polls:detail' question.id %}">Vote again?</a>
```
Race condition may happen in this vote function.  https://docs.djangoproject.com/en/2.1/ref/models/expressions/#avoiding-race-conditions-using-f

# 7, use generic views, less is better 
detail and results views are redundant, they represent a common case of basic web development: getting data from the database according to a parameter passed in the URL, 
loading a template and returning the rendered template. It's so common, Django provides a shortcut, called the "generic views" system. Taking following steps: 
+ Convert the URLconf 
+ Delete some of the old, unneeded views 
+ Introduce new views based on Django's generic views 

Convert the URLconf 
```
#polls/urls.py 
from django.urls import path

from . import views

app_name = 'polls'
urlpatterns = [
    path('', views.IndexView.as_view(), name='index'),
    path('<int:pk>/', views.DetailView.as_view(), name='detail'),
    path('<int:pk>/results/', views.ResultsView.as_view(), name='results'),
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```

Delete some of the old, unneeded views 
```
#polls/views.py
from django.http import HttpResponseRedirect
from django.shortcuts import get_object_or_404, render
from django.urls import reverse
from django.views import generic

from .models import Choice, Question

class IndexView(generic.ListView):
    template_name = 'polls/index.html'
    context_object_name = 'latest_question_list'

    def get_queryset(self):
        """Return the last five published questions."""
        return Question.objects.order_by('-pub_date')[:5]

class DetailView(generic.DetailView):
    model = Question
    template_name = 'polls/detail.html'

class ResultsView(generic.DetailView):
    model = Question
    template_name = 'polls/results.html'

def vote(request, question_id):
    ... # same as above, no changes needed.
```

Two generic views are introduced here:   
+ ListView - display a list of objects
+ DetailView - display a detail page for a particular type of object 
Each generic view needs to know what model it will be acting upon.   
The DetailView generic view expects the primary key value captured from the URL to be called "pk"  
By default, the DetailView generic view uses a template called <app name>/<model name>_detail.html, it is overrided by template_name.     
the ListView generic view uses a default template called <app name>/<model name>_list.html  


further info regarding generic view:  https://docs.djangoproject.com/en/2.1/topics/class-based-views/ 
