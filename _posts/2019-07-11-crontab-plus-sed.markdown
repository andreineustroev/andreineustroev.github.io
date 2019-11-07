---
layout: post
title:  "crontab + sed"
date:   2019-10-09 23:29:20 +0500
categories: bash, linux
---
Возникла у меня необходимость поправить много строк в кроне, сделаю я это конечно же `sed` ом =) Загвоздка в том что правила были написаны через `crontab -e`, и просто как файл их не прочитать, да и писать в файлы не рекомендуется, crontab даёт валидацию и сообщает демону об измененниях. Поэтому используем пайп)
{% highlight bash %}
crontab -l | sed \'s/one@example.com/two@example.com/g\' | crontab -
{% endhighlight %}
