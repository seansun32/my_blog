---
layout: post
title:  "My first blog: markdown test"
date:   2017-06-28 10:28:29 +0800
categories: document
tag: document 
---

* content
{:toc}


First POST build by Jekyll.

markdown header test
==================

##This is H2##

###This is H3

####This is H4

#####This is H5

---

markdwon quote test
====================

>THis is the first level of quoting
>
>>THis is the second level of quoting
>>
>>>This is the thid level
>>
>
>Back to first level

$$
\frac{7x+5}{1+y^2}

\begin{pmatrix}
1 &  2&3 \\ 
 4&  5&6 
\end{pmatrix}

$$

$\frac{7x+5}{1+y^2}$

---

markdown list test
===================

*   Red
*   Green
*   Blue

1.  Bird
2.  Cat
3.  Dog

*   this is a test by paragraph1
    this is a test by paragraph1
*   this is a test by paragraph2
    this is a test by paragraph2
*   list content with quote

    >quoted
*   list content with code

        int main(){
            printf("hello world\n"); 
        }

---

markdown mathjax render
===
https://webdemo.myscript.com/views/math.html#
https://www.codecogs.com/latex/eqneditor.php?lang=zh-cn


markdown code test
===================


##code with indent

    void
    ngx_close_channel(ngx_fd_t *fd, ngx_log_t *log)
    {
        if (close(fd[0]) == -1) {
            ngx_log_error(NGX_LOG_ALERT, log, ngx_errno, "close() channel failed");
        }
    
        if (close(fd[1]) == -1) {
            ngx_log_error(NGX_LOG_ALERT, log, ngx_errno, "close() channel failed");
        }
    }

##code with back-ticks

```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```
 
```python
s = "Python syntax highlighting"
print s
```

```c
void
ngx_close_channel(ngx_fd_t *fd, ngx_log_t *log)
{
    if (close(fd[0]) == -1) {
        ngx_log_error(NGX_LOG_ALERT, log, ngx_errno, "close() channel failed");
    }
    
    if (close(fd[1]) == -1) {
        ngx_log_error(NGX_LOG_ALERT, log, ngx_errno, "close() channel failed");
    }
}
```
 
```
No language indicated, so no syntax highlighting. 
But let's throw in a <b>tag</b>.
```

markdown linkage test
======================

I get 10 times more traffic from [Google] [1] than from
[Yahoo] [2] or [MSN] [3].

  [1]: http://google.com/        "Google"
  [2]: http://search.yahoo.com/  "Yahoo Search"
  [3]: http://search.msn.com/    "MSN Search"

This is [a link](http://www.baidu.com "Title") of baidu

---------------------

markdown inline test
======================

*this is emphasize*

**this is strong**

_underscores_

~~scratch this~~

use the `printf()` function

markdown image test
======================

![test img](../../../../styles/images/favicon.jpg)

------------------

markdown tables test
=====================

|   Tables  |   Are |   Cool    |
|   -----   |:----: |   ------: |
| col 3 is  | right |   $20000  |
| col 2 is  | cen   |   $30000  |

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |

-----------------

markdown linebreak test
=======================

hello world1

hello world2
hello world3


hello world4

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
