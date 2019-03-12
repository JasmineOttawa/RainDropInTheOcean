---
layout: post
title: Build a web application using Django web framework - part 3 
category: Python
tags: [Python]
---
views, reference  https://docs.djangoproject.com/en/2.1/intro/tutorial03/  

Using polls example to show concepts, tags which conflicts with Liquid control flow tags will be skipped here. 

# 1, Evolve a view
step1, Follow is an index view, showing the latest 5 questions.  it has design hard-coded in view    
```
  latest_question_list=Question.objects.order_by('-pub_date')[:5]
	output=', '.join([q.question_text for q in latest_question_list])
	return HttpResPonse(output)
```

step2, Use template system to seperate form design from view      
Create a template html file: polls/templates/polls/index.html

In polls/views.py, loads the template called polls/index.html and passes it a context. The context is a dictionary mapping template variable names to Python objects.
```
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

Put another template file for details page : polls/templates/polls/detail.html   

a shortcut for get_object_or_404()
```
    question = get_object_or_404(Question, pk=question_id)
    return render(request, 'polls/detail.html', {'question': question})
```
The get_object_or_404() function takes a Django model as 1st argument and a pk, which it passes to the get() function of the model’s manager. It raises Http404 if the object doesn’t exist.     
There’s also a get_list_or_404() function, same as get_object_or_404() – except using filter() instead of get().   
Why do we use a helper function get_object_or_404() instead of automatically catching the ObjectDoesNotExist exceptions at a higher level, or having the model API raise Http404 instead of ObjectDoesNotExist?   
Because that would couple the model layer to the view layer. One of the foremost design goals of Django is to maintain loose coupling. Some controlled coupling is introduced in the django.shortcuts module.   

# 3, removing hardcoded URLs in templates 
in index.html, there is:
```
href="/polls/{{ question.id }}/"
```
The link is partially hardcoded. It could be removed by 
```
href="X url 'detail' question.id X"
```
As in urls.py, the detail path is defined as: path('<int:question_id>/', views.detail, name='detail'),      
Now if you change the URL of the polls detail view to something else, like following, it will still work      
path('specifics/<int:question_id>/', views.detail, name='detail')


# 4, Namespacing URL names
There might be other apps in same project that has a detail view, when using X url X tamplate tag, how does Django knows which app it is for? 
The answer is to add namespace to your URLconf, add app_name=... 
```
#polls/urls.py
app_name = 'polls'
urlpatterns = [
    path('', views.index, name='index'),
 ....
]
```
In this way, detail view can be referenced by polls:detail 

# 5, write a simple form  
+ Form method=POST,  post will submit data to server side      
+ template tag: csrf_token, for POST form, we need to worry about CSRF - Cross Site Request Forgeries, Django comes with a ease-to-use system for protecting against it.     
+ request.POST, a dictionary-like object to transfer data from form to view,   
+ request.GET access GET data in the same way 
+ request.POST['choice'] raise KeyError if choice wasn't provided in POST data 
+ Always return an HttpResponseRedirect after successfully dealing with POST data. This prevents data from being posted twice if a user hits the Back button.
+ reverse(), helps avoid having to hardcode a URL in the view function. It is given the name of the view that we want to pass control to and the variable portion of the URL pattern that points to that view.
+ request and response :  https://docs.djangoproject.com/en/2.1/ref/request-response/ 

# 6, use generic views, less is better 
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
