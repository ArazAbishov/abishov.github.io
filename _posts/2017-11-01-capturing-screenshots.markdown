---
layout: post
title: >
  A graceful way to capture screenshots in UI tests
date: 2017-11-01 15:00:00 +0000
---

If you had experience of writing UI tests for android, you probably heard of tools like Spoon or Composer. Along with orchestration of tests execution, they provide APIs for capturing screenshots which later are placed into generated HTML report. All of these is great, but there are certain associated shortcomings:
 - Screenshot capturing logic clutters tests
 - Hard to replace one library by another

Here is the example of an instrumentation test for a simple form screen:

```java
@RunWith(AndroidJUnit4.class)
public class FormScreenTest {

  @Rule
  public ActivityTestRule<FormActivity> activityTestRule =
          new ActivityTestRule<>(FormActivity.class);

  @Test
  public void clickOnSubmitMustHighlightErrors() {
    Spoon.screenshot(testRule.getActivity(), "state_before_edit");

    onView(withId(R.id.title)).perform(replaceText("test_title"));
    onView(withId(R.id.description)).perform(replaceText("test_description"));
    onView(withId(R.id.submit)).perform(click());

    Spoon.screenshot(testRule.getActivity(), "state_after_edit");
  }
}
```

Here I am using `Spoon` to capture screenshots which represent the state before and after user actions. This looks fine, but screenshot capturing logic shouldn't really be a part of the test body. Moreover, it simply becomes tedious to enter tags by hand, when it can be derived from the test name. This is what we can have instead:

```java
@RunWith(AndroidJUnit4.class)
public class FormScreenTest {

  @Rule
  public ScreenshotsRule<FormActivity> screenshotsRule =
          new ScreenshotsRule<>(FormActivity.class);

  @Test
  @CaptureScreenshots
  public void clickOnSubmitMustHighlightErrors() {
    onView(withId(R.id.title)).perform(replaceText("test_title"));
    onView(withId(R.id.description)).perform(replaceText("test_description"));
    onView(withId(R.id.submit)).perform(click());
  }
}
```

Looks better huh? Notice that tags are derived from a test name automatically. Here is another example where tags are specified explicitly and the state of activity is captured within test using the `screenshot("tag")` method:

```java
@RunWith(AndroidJUnit4.class)
public class FormScreenTest {

  @Rule
  public ScreenshotsRule<FormActivity> screenshotsRule =
          new ScreenshotsRule<>(FormActivity.class);

  @Test
  @CaptureScreenshots(before = "before_state", after = "after_state")  
  public void clickOnSubmitMustHighlightErrors() {
    onView(withId(R.id.title)).perform(replaceText("test_title"));
    onView(withId(R.id.description)).perform(replaceText("test_description"));

    // capture screenshot within test
    screenshotsRule.screenshot("intermediate_state");
    onView(withId(R.id.submit)).perform(click());
  }
}
```

----
So how do we get there? By writing a jUnit rule which knows where to grab an instance of `Activity` and when to invoke `Spoon`. First of all, we will need an annotation for marking tests for which we want to capture screenshots:

```java
@Target(METHOD)
@Retention(RUNTIME)
public @interface CaptureScreenshots {
  String before() default "";
  String after() default "";
}
```

Since we have to follow `ActivityTestRule`'s lifecycle, we will have to build our rule upon it:

```java
public class ScreenshotsRule<T extends Activity> extends ActivityTestRule<T> {
  private Description description;

  public ScreenshotsRule(Class<T> clazz) {
    super(clazz);
  }

  @Override
  public Statement apply(Statement base, Description description) {
    this.description = description;
    return super.apply(base, description);
  }

  @Override
  public void finishActivity() {
    if (getActivity() != null) {
      beforeActivityFinished();
    }
    super.finishActivity();
  }

  public void screenshot(String tag) {
    if (getActivity() == null) {
      throw new IllegalStateException("Activity has not been started yet" +
              " or has already been killed!");
    }

    Spoon.screenshot(getActivity(), tag, description.getClassName(),
            description.getMethodName());
  }

  Override
  protected void afterActivityLaunched() {
    CaptureScreenshots captureScreenshots =
            description.getAnnotation(CaptureScreenshots.class);
    if (captureScreenshots != null) {
      String tag = captureScreenshots.before();
      if (TextUtils.isEmpty(tag)) {
        tag = "before_" + description.getMethodName();
      }
      screenshot(tag);
    }
  }

  protected void beforeActivityFinished() {
    CaptureScreenshots captureScreenshots =
            description.getAnnotation(CaptureScreenshots.class);
    if (captureScreenshots != null) {
      String tag = captureScreenshots.after();
      if (TextUtils.isEmpty(tag)) {
        tag = "after_" + description.getMethodName();
      }
      screenshot(tag);
    }
  }
}
```

> - In sake of simplicity and brevity of example, I have not overriden all of the parent constructors.
> - We had to supply our own version of `beforeActivityFinished()` callback by overriding `finishActivity()`, which is not the cleanest solution. I have created a [feature request](https://issuetracker.google.com/issues/68897841) for adding it to `ActivityTestRule` in support library.

Within `apply` method we are storing a reference to `description` of the test method which is under execution. It contains metadata like names of a test method and class, which later will be useful both for generation of the screenshot tag and invocation of `Spoon`.

TODOs:
 - Improve description of logic within ScreenshotsRule
 - A few lines regarding application of the rule
 - Write a conclusion where you address two points from introduction: less boilerplate + abstraction which allows you to change plugin without sweating 
