---
layout: post
title: >
  Cutting boilerplate in instrumentation tests
date: 2017-11-01 15:00:00 +0000
---
// ToDo: complete code snippets  
// ToDo: consider adding example with Intents  
// ToDo: rename annotation to be @CaptureScreenshots?

If you had experience of writing instrumentation tests for android, you probably heard of tools like Spoon or Composer. Along with orchestration of tests execution, they provide APIs for capturing screenshots which later are placed into generated HTML report. All of these is great, but there are certain associated shortcomings:
 - Adds complexity to tests which makes code less readable
 - Hard to switch out one library by another

Here is a typical example of an instrumentation test:

```java
@Test
public void clickOnLoginButtonMustNavigateToHome() {
  Spoon.screenshot(activityTestRule.getActivity(), "state_before_login");

  loginRobot.typeUsername("username")
    .typePassword("password")
    .login();  

  Spoon.screenshot(activityTestRule.getActivity(), "state_after_login");
}
```

Here I am using `Spoon` to capture screenshots which represent the state before and after user actions. This looks fine, but screenshot capturing logic shouldn't really be a part of the test body. Moreover, it simply becomes tedious to enter tags by hand, when the test name is a perfect candidate for it. This is what we can have instead:

```java
@Test
@CaptureScreenshot
public void clickOnLoginButtonMustNavigateToHome() {
  loginRobot.typeUsername("username")
    .typePassword("password")
    .login();
}
```

Looks better huh?
