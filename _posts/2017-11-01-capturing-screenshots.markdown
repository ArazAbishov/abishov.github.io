---
layout: post
title: >
  A graceful way for capturing screenshots in UI tests
date: 2017-11-01 15:00:00 +0000
---

If you had experience of writing instrumentation tests for android, you probably heard of tools like Spoon or Composer. Along with orchestration of tests execution, they provide APIs for capturing screenshots which later are placed into generated HTML report. All of these is great, but there are certain associated shortcomings:
 - Screenshot capturing logic clutters tests
 - Hard to replace one library by another

Here is the example of an instrumentation test for a simple form screen:

```java
@Test
public void clickOnSubmitMustNavigateToHome() {
  Spoon.screenshot(testRule.getActivity(), "state_before_edit");

  onView(withId(R.id.title)).perform(typeText("test_title"));
  onView(withId(R.id.description)).perform(typeText("test_description"));
  onView(withId(R.id.submit)).perform(click());

  Spoon.screenshot(testRule.getActivity(), "state_after_edit");
}
```

Here I am using `Spoon` to capture screenshots which represent the state before and after user actions. This looks fine, but screenshot capturing logic shouldn't really be a part of the test body. Moreover, it simply becomes tedious to enter tags by hand, when it can be derived from the test name. This is what we can have instead:

```java

@RunWith(AndroidJUnit4.class)
public class FormScreenTest {

  @Rule
  public ScreenshotsRule<FormActivity> screenshotsRule =
          new ScreenshotsRule<>(FormActivity.class);
  /**
   * Tags in this example are automatically
   * derived from the test name.
   */
  @Test
  @CaptureScreenshots
  public void clickOnSubmitMustNavigateToHome() {
    onView(withId(R.id.title)).perform(replaceText("test_title"));
    onView(withId(R.id.description)).perform(replaceText("test_description"));
    onView(withId(R.id.submit)).perform(click());
  }

  /**
   * In case if you want to customize tags,
   * you can explicitly specify them.
   */
  @Test
  @CaptureScreenshots(before = "tag_before_test", after = "tag_after_test")
  public void clickOnSubmitMustNavigateToHome() {
    onView(withId(R.id.title)).perform(replaceText(""));
    onView(withId(R.id.description)).perform(replaceText(""));

    // capture screenshot within test
    screenshotsRule.screenshot("tag_intermediate_state");

    onView(withId(R.id.submit)).perform(click());
  }  
}

```

Looks better huh? This can be done by writing a jUnit rule which knows where to grab an instance of `Activity` and when to invoke `Spoon`. First of all, we will need an annotation for marking tests which we want to "screenshot":

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
> - We had to supply our own version of `beforeActivityFinished()` callback by overriding `finishActivity()`. I have created a [feature request](https://issuetracker.google.com/issues/68897841) to add official support for it in `ActivityTestRule`.

Within `apply` method we are capturing `description` of the test method which is under execution. It contains useful metadata like the name of test method, as well as the test class name. Those properties are useful both for generation of the screenshot tag and calling `Spoon`. Now we can already apply this rule to instrumentation tests and it will do its magic.


TODOs:
 - mention that this example is based on espresso 3
 - good examples of espresso tests
