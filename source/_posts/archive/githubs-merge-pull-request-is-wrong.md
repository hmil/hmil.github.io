---
title: 'GitHub''s "merge pull request" is wrong'
tags:
  - git
  - github
id: 209
categories:
  - tools
date: 2015-11-21 14:49:06
---

This is the story of Bob. Bob is working on some cool project on github. He's written tests and runs them continuously on [travis](https://travis-ci.org). Bob asks his contributors to write tests for new features they introduce. Using GitHub's pull requests, he can make sure no bugs are introduced by looking for the green tick next to the commit number. If you look at the commit tree in Bob's repository, you'll probably find something like [this](http://nvie.com/posts/a-successful-git-branching-model/).

Bob states in his contribution guide that people can fork _master_ to work on new features. He guarantees that master's tests always pass and therefore, individual contributors don't need to worry that much about errors made by other developers.

If you're familiar with github then you are probably familiar with this workflow and you'll think that it's perfectly reasonable.

But you'll be wrong. Bob got overly enthusiastic about github's UI and will soon discover that **he's been wrong all this time**.

## A catastrophic scenario

Say two contributors, Alice and Charlie, are writing code for Bob's project in their respective branches. In this project, we find the following code for operating Bob's car:
```java

private void doSomething(String action) {
  switch (action) {
    case "OPEN_DOOR":
       open_door();
       break;
  }
}
```

Now Alice thinks it would be better if Bob used an **enum** instead of a string. She changes the function to the following and opens a pull request on Bob's project.
```java

private void doSomething(Action action) {
  switch (action) {
    case Action.OPEN_DOOR:
       open_door();
       break;
  }
}
```

Bob opens his project page and sees a pull request. He thinks Alice's modification makes sense and, since all tests passed, he clicks on _the green button_.

[caption id="attachment_221" align="alignnone" width="776"][![The button of doom](/assets/archive/2015/11/button_of_doom.png)](/assets/archive/2015/11/button_of_doom.png) The button of doom[/caption]

Meanwhile, Charlie thinks Bob will need to close his car's door at some point. Charlie adds a new case to the switch:

```java

private void doSomething(String action) {
  switch (action) {
    case "OPEN_DOOR":
       open_door();
       break;
    case "CLOSE_DOOR":
       close_door();
       break;
  }
}
```

The next day, Bob opens his project on GitHub and sees Charlie's pull request. All tests passed, no merge conflict, that's a clear <acronym title="Looks Good To Me">LGTM</acronym> and Bob merges the code.

A few days later, Bob receives complaints from multiple developers, they forked master and it doesn't build!

## What could go wrong?

The problem lies in a common misunderstanding of the following assumptions. 

*   Different PRs must be completely independent
*   No merge conflicts &ne; No conflicts

If you accept one of these to be violated, then expect your main branch to fail tests!

If we speak in terms of diff, then Alice's diff is the following:
```java

- private void doSomething(String action) {
+ private void doSomething(Action action) {
    switch (action) {
-     case "OPEN_DOOR":
+     case Action.OPEN_DOOR:
         open_door();
...
```
While Bob's diff looks like this:
```java

...
         break;
+     case "CLOSE_DOOR":
+        close_door();
+        break;
  }
...
```

Put together we get
```java

- private void doSomething(String action) {
+ private void doSomething(Action action) {
    switch (action) {
-     case "OPEN_DOOR":
+     case Action.OPEN_DOOR:
         open_door();
         break;
+     case "CLOSE_DOOR":
+        close_door();
+        break;
...
```

This diff **does not cause a merge conflict** but it is actually in conflict!

## --no-ff and the button of doom

The flaw resides in that big old "merge pull request" button. While the green checks above the button and its fat style make pressing it a rewarding thing, it hides the real danger behind it: a blind merge!

The above example looks like this in terms of commit history:
```

        +-PR_Charlie-+
       /              \
      +--PR_Alice--+   \
     /              \   \
 ---+----------------x---x
                         |
                        master
```
All changes introduced by Alice and Charlie on their respective PRs have been tested, but the merge commits (marked 'x' above) were not! This is called a "non fast-forward" merge (--no-ff in git) and is basically like committing directly on master.

## The right way to do it

Assuming you are like Bob and want your main branch to always pass tests, then here is the right way to do this: Merge master into the PR fisrt, and merge only when fast forward. It would look like that:

```


Initial state: both PRs are unmerged:

        +-PR_Charlie-+
       /              
      +--PR_Alice--+  
     /                 
 ---+
    |
  master

on master: git merge --ff-only PR_Alice

      +-PR_Charlie-+
     /                              
 ---+--PR_Alice--+
                 |
               master

on PR_Charlie: git merge master

                  merge branch master into PR_Charlie   
                     |
      +-PR_Charlie-+-+
     /              /                  
 ---+--PR_Alice----+
                   |
                 master

Then on master: git merge --ff-only PR_Charlie

      +-PR_Charlie-+
     /              \                  
 ---+--PR_Alice----+-+
                     |
                   master
```

The end result is exactly the same as before EXCEPT that master was merged into the PR first, and this (if properly pushed to github) allows the automated tools to test the last commit. This ensures that master builds because master will always be on a commit that has been tested before.

Many beginner (and even advanced) programmers get fooled into thinking that merging a green pull request cannot harm. In an ideal world, clicking the green button would: merge master into the branch, wait for the tests to pass, and on success, fast-forward master to the merge commit. But unfortunately I don't think this is coming anytime soon. In the meantime, think about it twice before you press the _button of doom_. And when in doubt, run the merge sequence manually in the right order. This may save you a few hours of debugging.