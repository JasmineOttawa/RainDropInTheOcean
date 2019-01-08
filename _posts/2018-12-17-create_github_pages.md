---
layout: post
title: Create Github Pages
category: Others 
tags: [others]
---

Fork a repository
Open it in GitHub Desktop, make changes
Commit, 
Publish 
Waiting 15 minutes,  https://username.github.io/repositoryName is accessible 

 
## Q & A   
 
### Q1 - 2 types of Github Pages Sites
There are two basic types of GitHub Pages sites: Project & User and Organization.   
- Project Pages sites:   
  they are published from one of the following locations:   
  . master branch   
  . gh-pages branch   
  . a folder names "docs" located on the master branch   
  A Project Pages site for a personal account is available at http(s)://<username>.github.io/<projectname> .   
- User and Organization Pages Sites: 
User or Organization Page that has a repository named <username>.github.io or <orgname>.github.io , you cannot publish your site's source files from different locations. User and Organization Pages that have this type of repository name are only published from the master branch.

  
### Q2 - how to delete a forked repo in git hub   
On GitHub, navigate to the main page of the repository.  
Under your repository name, click  Settings.  
Under Danger Zone, click Delete this repository.  
Read the warnings.  
To verify that you're deleting the correct repository, type the name of the repository you want to delete.  
Click I understand the consequences, delete this repository   

### Q3 - origin & remote 
In Git, "origin" is a shorthand name for the remote repository that a project was originally cloned from. 
More precisely, it is used instead of that original repository's URL - and thereby makes referencing much easier. 
Note that origin is by no means a "magical" name, but just a standard convention.

In the following example, the URL parameter to the "clone" command becomes the "origin" for the cloned local repository:
```
git clone https://github.com/user/repo.git
```
In github Desktop, you can see "Fetch Origin" 

A remote URL is Git's fancy way of saying "the place where your code is stored." 
That URL could be your repository on GitHub, or another user's fork, or even on a completely different server. 
You can only push to two types of URL addresses: 
```
An HTTPS URL like https://github.com/user/repo.git
An SSH URL, like git@github.com:user/repo.git
```
Git associates a remote URL with a name, and your default remote is usually called origin.

You can use the git remote add command to match a remote URL with a name. following statement associates the name origin with the REMOTE_URL:
```
git remote add origin  <REMOTE_URL> 
```
