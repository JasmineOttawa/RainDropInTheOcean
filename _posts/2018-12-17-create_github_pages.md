---
layout: post
title: Create Github Pages
category: Others 
tags: [others]
---


```
sh.addShard( "rs1/IP1:27017")  
sh.addShard( "rs2/IP2:27017")  
db.adminCommand( { listShards : 1 } )  
```  
## Q & A   
### Q1 - On ubuntu 16.04, when editing mongod.conf in Mobaxterm, it changes first character to 'g'  
A - change env variable TERM from xterm to linux   
  
### Q2 - 2 types of Github Pages Sites
There are two basic types of GitHub Pages sites: Project & User and Organization.   
- Project Pages sites:   
  they are published from one of the following locations:   
  . master branch   
  . gh-pages branch   
  . a folder names "docs" located on the master branch   
  A Project Pages site for a personal account is available at http(s)://<username>.github.io/<projectname> .   
- User and Organization Pages Sites: 
User or Organization Page that has a repository named <username>.github.io or <orgname>.github.io , you cannot publish your site's source files from different locations. User and Organization Pages that have this type of repository name are only published from the master branch.

  
### Q3 - how to delete a forked repo in git hub   
On GitHub, navigate to the main page of the repository.  
Under your repository name, click  Settings.  
Under Danger Zone, click Delete this repository.  
Read the warnings.  
To verify that you're deleting the correct repository, type the name of the repository you want to delete.  
Click I understand the consequences, delete this repository   

### Q4 - origin, remote 
