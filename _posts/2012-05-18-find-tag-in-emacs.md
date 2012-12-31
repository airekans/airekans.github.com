---
layout: post
title: "Emacs中的Tag查找功能"
description: "Explains the implementation of find-tag function in Emacs"
category: Emacs
tags: [emacs, elisp]
---
{% include JB/setup %}

在Emacs里面，查找symbol并跳转到其定义上是通过etags来完成的，和Vim的相类似。但是在用了一段时间之后，发觉etags的跳转在对python的支持有时候很不智能，经常会跳转到`import`语句而不是`def`语句，这个让我颇为恼火，当时就下决心要抽个时间看看Emacs里面的实现是怎么回事，有没有什么改进的余地。

首先简单的介绍一下etags的用法。一般要用etags，就要经过下面几步：

1.  在源文件的根目录下，执行后面的语句：`find . -name '*.c' -exec etags -a {} \;`
		
    这个会生成一个TAGS文件，
	是Emacs用来查找tags的默认名字。
	
2.  打开emacs，在想要看定义的symbol(变脸或函数)上面按`M-.`(英文里面的句号)，
    或者直接`M-x find-tag`来查找。然后会提示TAGS的目录，输入就是了。
	
3.  一般来说，到了这一步，Emacs就会跳转到对应的symbol定义处了。

在讲解之前，先说清楚一个概念，就是tag。tag就是在etags里面识别出来的一个作为标识symbol。

那么上面的1主要处理的是tag的生成，而2是从生成的tag里面查找。所以我的问题主要是在2里面，也就是Emacs是怎么查找tag的。(不过在了解了机制之后，发现对于python来说，原来我的问题是落在1里面的，这个是后话)

在Emacs里面，所有和`find-tag`函数相关的东西都定义在`etags.el`里面，这个也是提供和etags对接的一个Emacs库。

在调用`find-tag`的时候，就会执行下面的语句：

{% highlight cl linenos %}
(defun find-tag (tagname &optional next-p regexp-p)
  (interactive (find-tag-interactive "Find tag: "))
  (let* ((buf (find-tag-noselect tagname next-p regexp-p)) ;****
         (pos (with-current-buffer buf (point))))
    (condition-case nil
        (switch-to-buffer buf)
      (error (pop-to-buffer buf)))
    (goto-char pos))){% endhighlight %}

实际上调用的第3行的`find-tag-noselect`。那么就看看`find-tag-noselect`干了些什么。

下面是`find-tag-noselect`的定义：

{% highlight cl linenos %}
(defun find-tag-noselect (tagname &optional next-p regexp-p)
  (interactive (find-tag-interactive "Find tag: "))

  (setq find-tag-history (cons tagname find-tag-history))
  ;; Save the current buffer's value of `find-tag-hook' before
  ;; selecting the tags table buffer.  For the same reason, save value
  ;; of `tags-file-name' in case it has a buffer-local value.
  (let ((local-find-tag-hook find-tag-hook))
    (if (eq '- next-p)
        ;; Pop back to a previous location.
        (if (ring-empty-p tags-location-ring)
            (error "No previous tag locations")
          (let ((marker (ring-remove tags-location-ring )))
            (prog1
                ;; Move to the saved location.
                (set-buffer (or (marker-buffer marker)
                                (error "The marked buffer has been deleted")))
              (goto-char (marker-position marker))
              ;; Kill that marker so it doesn't slow down editing.
              (set-marker marker nil nil)
              ;; Run the user's hook.  Do we really want to do this for pop?
              (run-hooks 'local-find-tag-hook))))
      ;; Record whence we came.
      (ring-insert find-tag-marker-ring (point-marker))
      (if (and next-p last-tag)
          ;; Find the same table we last used.
          (visit-tags-table-buffer 'same)
        ;; Pick a table to use.
        (visit-tags-table-buffer)
        ;; Record TAGNAME for a future call with NEXT-P non-nil.
        (setq last-tag tagname))
      ;; Record the location so we can pop back to it later.
      (let ((marker (make-marker)))
        (with-current-buffer
            ;; find-tag-in-order does the real work.
            (find-tag-in-order
             (if (and next-p last-tag) last-tag tagname)
             (if regexp-p
                 find-tag-regexp-search-function
               find-tag-search-function)
             (if regexp-p
                 find-tag-regexp-tag-order
               find-tag-tag-order)
             (if regexp-p
                 find-tag-regexp-next-line-after-failure-p
               find-tag-next-line-after-failure-p)
             (if regexp-p "matching" "containing")
             (or (not next-p) (not last-tag)))
          (set-marker marker (point))
          (run-hooks 'local-find-tag-hook)
          (ring-insert tags-location-ring marker)
          (current-buffer)))))){% endhighlight %}

首先看到`find-tag-noselect`在`next-p`为负数的情况下是会跳回到之前的tag，而不是跳转到当前tag的位置。  
而在接下来的判断中，`find-tag-noselect`首先把tag-table打开，然后记录下当前的tag，以便在之后跳回到这个tag。

最重要的就是调用了`find-tag-in-order`。从名字可以看出，这个函数是从`tag-table`中逐个逐个的找tag。实际上，`find-tag-in-order`是首先利用一个general的search函数粗略的匹配tag，然后再用order参数(一个函数列表)里面的函数按照不同的标准来进行进一步的匹配。

下面是`find-tag-in-order`的定义：

{% highlight cl linenos %}
(defun find-tag-in-order (pattern
                          search-forward-func
                          order
                          next-line-after-failure-p
                          matching
                          first-search)
  ;; State is saved so that the loop can be continued.
  (let (file                            ;name of file containing tag
        tag-info                        ;where to find the tag in FILE
        (first-table t)
        (tag-order order)
        (match-marker (make-marker))
        goto-func
        (case-fold-search (if (memq tags-case-fold-search '(nil t))
                              tags-case-fold-search
                            case-fold-search))
        )
    (save-excursion

      (if first-search
          (setq tag-lines-already-matched nil)
        (visit-tags-table-buffer 'same))

      ;; Get a qualified match.
      (catch 'qualified-match-found

        ;; Iterate over the list of tags tables.
        (while (or first-table
                   (visit-tags-table-buffer t))

          (and first-search first-table
               ;; Start at beginning of tags file.
               (goto-char (point-min)))

          (setq first-table nil)

          ;; Iterate over the list of ordering predicates.
          (while order
            (while (funcall search-forward-func pattern nil t)
              ;; Naive match found.  Qualify the match.
              (and (funcall (car order) pattern)
                   ;; Make sure it is not a previous qualified match.
                   (not (member (set-marker match-marker (save-excursion
                                                           (beginning-of-line)
                                                           (point)))
                                tag-lines-already-matched))
                   (throw 'qualified-match-found nil))
              (if next-line-after-failure-p
                  (forward-line 1)))
            ;; Try the next flavor of match.
            (setq order (cdr order))
            (goto-char (point-min)))
          (setq order tag-order))
        ;; We throw out on match, so only get here if there were no matches.
        ;; Clear out the markers we use to avoid duplicate matches so they
        ;; don't slow down editting and are immediately available for GC.
        (while tag-lines-already-matched
          (set-marker (car tag-lines-already-matched) nil nil)
          (setq tag-lines-already-matched (cdr tag-lines-already-matched)))
        (set-marker match-marker nil nil)
        (error "No %stags %s %s" (if first-search "" "more ")
               matching pattern))

      ;; Found a tag; extract location info.
      (beginning-of-line)
      (setq tag-lines-already-matched (cons match-marker
                                            tag-lines-already-matched))
      ;; Expand the filename, using the tags table buffer's default-directory.
      ;; We should be able to search for file-name backwards in file-of-tag:
      ;; the beginning-of-line is ok except when positioned on a "file-name" tag.
      (setq file (expand-file-name
                  (if (memq (car order) '(tag-exact-file-name-match-p
                                          tag-file-name-match-p
                                          tag-partial-file-name-match-p))
                      (save-excursion (forward-line 1)
                                      (file-of-tag))
                    (file-of-tag)))
            tag-info (funcall snarf-tag-function))

      ;; Get the local value in the tags table buffer before switching buffers.
      (setq goto-func goto-tag-location-function)
      (tag-find-file-of-tag-noselect file)
      (widen)
      (push-mark)
      (funcall goto-func tag-info)

      ;; Return the buffer where the tag was found.
      (current-buffer)))){% endhighlight %}

其中最为重要的就是`while`的一部分。首先`order`不为空，然后用`search-forward-func`(默认是`search-forward`)来查找这个tag的pattern，如果找到了，就再用`(car order)`来进行仔细的匹配。所以`order`里面的函数就是匹配的关键，就看里面有些什么匹配函数了。`order`是在`find-tag-noselect`传进来的。

`order`在tag是正则表达式的时候是`find-tag-regexp-tag-order`，而在tag是普通的字符串的时候就是`find-tag-tag-order`。这里我着重看了一下`find-tag-tag-order`，它默认是下面的列表：

{% highlight cl linenos %}
(tag-exact-file-name-match-p
 tag-file-name-match-p
 tag-exact-match-p
 tag-implicit-name-match-p
 tag-symbol-match-p
 tag-word-match-p
 tag-partial-file-name-match-p
 tag-any-match-p){% endhighlight %}

所以你可以看到，他是先按tag是不是完全匹配文件名，然后再去匹配看看是不是匹配tag，如果还是找不到的话，就去部分的匹配。而一般来说只有在一个tag在精准匹配里面找不到的时候，才可能去部分的匹配。也还有一种可能，那就生成tag的时候本身就生成错了，导致一些不是tag的地方也变成了tag。而在Python中，etags生成的tag就真的不全是我们想要的！

用etags对python进行tag的生成的时候，会把import语句也当成是tag的一种，从而生成在TAGS文件里面，所以用emacs的`find-tag`跳转的时候就会发现，当我想找一个tag的定义的时候，他却经常的跳到了import的地方，就是这个原因。

这里也大概的说一下etags生成的tag-table格式(我还没有看过etags的源码，是通过看TAGS文件总结出来的)。

    x7F
    <filename>,<No1>
    <Matched tag line>x7F[<tag>x01]<Line num>,<No2>
    .....

每个tag文件都用x7F来分隔每个scan的文件。

后记：

通过查看了解etags.el的实现，我大致明白了Emacs里面一个library的构成和编写方式。也通过它明白了Emacs里面调试的一些小技巧，通过edebug能够比较好的了解一个elisp程序的运行状态。通过这个库，我明白其实Emacs里面的灵活性主要是通过elisp实现的，而其实它上面的插件都不是太成熟，但是由于elisp具有较好的可读性，而且用emacs的人都比较经得起折腾（或者说hack源码或者自己动手的能力较强），所以导致了emacs的用户都说Emacs是神器。事实上，我现在用的也还不是太熟，算是在进阶阶段，感觉Emacs有些功能还是不错的，至少在拓展性上我觉得比Vim要强，不过打字速度还真没有vim来的快，看个人用的怎么样了。

理解Emacs Lisp Library的编写，就需要看看`require`和`provide`这两个函数，还有一些`defgroup`和`defcustom`。


