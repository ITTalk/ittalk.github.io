---
layout: default
title:  "Yarn and angular 5"
date:   2017-11-08 1:12:08 +0200
categories: angular yarn
post_author: Sylwester Grabowski
---
This article is describing step by step how to use angular 5 with yarn. 

####  What is yarn? 
It's a new package manager that was built by Facebook, Exponent, Google, and Tilde. 

And the second question in my mind is why? 
Because JavaScript is now, an assembler of the web, everything is in JavaScript, from mobile app, through fronted to backend. Also at the database level we are using JavaScript (NoSQL).

With that popularity, current npm “sucks”. With tree structure of folders, it's very easy to hit maximum number of characters in windows path. This structure also depends of the order of installation. So, developers can have different structure of node modules folder in the same project. It also doesn't lock the version of the file, and create checksum, so we are never sure that the package that we are installing on prod, is the same that was used on dev. And so on.

People at Facebook tried [(source)](https://code.facebook.com/posts/1840075619545360) very hard to stay with npm, and find solutions for the problems above. But it was not enough. 
So they started working on a different approach, and they realized that it is the right direction. 
With help from folks from different companies, Yarn was “born”.
 Yarn is using existing npm catalog, it has locking feature, checksums, flat structure and so on. ;)
You can read about Yarn on their official page. https://yarnpkg.com .


####  How to use that?

First we need to install yarn using npm , it's the simplest way.
```cmd
npm install -g yarn
```
Wait, do I need to use one package manager to install another? Really?!

The answer is no, there is a standalone version of yarn.  It’s a single JavaScript file with all of Yarn’s dependencies bundled. In the last release (1.3.2) https://github.com/yarnpkg/yarn/releases/tag/v1.3.2 it's called yarn-1.3.2.js.


Ok, so we have yarn, so the next step is to use it. We will install angular cli with yarn.
Instead of calling 
```cmd
npm install -g @angular/cli //NPM example
```
We will write 
```cmd
yarn global add @angular/cli
```

Then, we can check if we have access to ng command. 
```cmd
ng help
```

If the ng command isn’t available, check the ensure that path to the Yarn bin is in your Environment  Variables. Ex. C:\Users\[Username]\AppData\Local\Yarn\bin

Then, we will set yarn as a default package manager for angular-cli. 
```cmd
ng set --global packageManager=yarn
```
And verify it has been set. 
```cmd
ng get --global packageManager
```
So we are ready to go. 

For security reasons, we will add angular/cli to our local package folder (-dev means devDependecy), and add mean that is should be saved in package.json. Just simple yarn @angular/cli will install angular/cli.
```cmd
yarn add @angular/cli --dev
```
The last step is to generate new project.
```cmd
ng new SolomonApp
ng serve 
```
And we are good to go.

Continuation will be in the next article.