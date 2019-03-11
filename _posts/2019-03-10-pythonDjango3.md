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
```
```
#in polls/views.py, loads the template called polls/index.html and passes it a context. The context is a dictionary mapping template variable names to Python objects.
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

# 2, Raising a 404 error 
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

