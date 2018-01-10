---
published: true
---
## Mô hình phân nhánh cho dự án phần mềm sử dụng GIT

Trong bài viết này, tôi sẽ giới thiệu đến các bạn một mô hình phát triển mà tôi sử dụng trong các dự án của tôi (trong công việc và dự án cá nhân). Tôi sẽ không nói về bất kỳ chi tiết nào của dự án, chỉ đơn thuần là về chiến lược phân nhánh và quản lý phát hành.

![Git branching model](/images/git/git-model@2x.png "Git branching model")


Create the branch on your local machine and switch in this branch :

<pre>$ git checkout -b [name_of_your_new_branch]</pre>

Change working branch :

<pre>$ git checkout [name_of_your_new_branch]</pre>

Push the branch on github :

<pre>$ git push origin [name_of_your_new_branch]</pre>

When you want to commit something in your branch, be sure to be in your branch. Add -u parameter to set upstream.

You can see all branches created by using :

<pre>$ git branch</pre>

Which will show :

<pre>* approval_messages
  master
  master_clean</pre>

Add a new remote for your branch :

<pre>$ git remote add [name_of_your_remote] <url></pre>

Push changes from your commit into your branch :

<pre>$ git push [name_of_your_new_remote] [name_of_your_branch]</pre>

Update your branch when the original branch from official repository has been updated :

```sh
$ git fetch [name_of_your_remote]
```
Then you need to apply to merge changes, if your branch is derivated from develop you need to do :

<pre>$ git merge [name_of_your_remote]/develop</pre>

Delete a branch on your local filesystem :

<pre>$ git branch -d [name_of_your_new_branch]</pre>

To force the deletion of local branch on your filesystem :

<pre>$ git branch -D [name_of_your_new_branch]</pre>

Delete the branch on github :

<pre>$ git push origin :[name_of_your_new_branch]</pre>

The only difference is the : to say delete, you can do it too by using github interface to remove branch : https://help.github.com/articles/deleting-unused-branches.

If you want to change default branch, it's so easy with github, in your fork go into Admin and in the drop-down list default branch choose what you want.
