---
layout: post
title:  "How to build a website hosted in Github in Windows with Jetyll"
date:   2016-03-08 12:15:56 +0800
categories: jekyll update

---

1. Install Github desktop

2. create a repo in github like username.github.io and open it in the github desktop

2. Install ruby 
	- add bin/ruby to path 
	- try command ruby -v to verify ruby installation

3. Install ruby Dev-kit
	- add a folder called devkit ruby installation folder 
	- extract dev-kit to devkit folder

	```cmd
		ruby dk.rb init
		ruby dk.rb install

	```

4. Install jekyll

	```cmd
		gem install bundler
		gem install jekyll

	```

5. create a blog

Navigate your blog folder and type in command line and done. 

	```cmd
		jekyll new . --force

	```

6. Start blog in the _posts folder



[ref1](https://www.youtube.com/watch?v=E512qOn8tZE)

[ref2](https://jekyllrb.com/docs/quickstart/)

