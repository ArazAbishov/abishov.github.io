---
layout: post
title: >
  Cutting boilerplate in instrumentation tests
date: 2017-11-01 15:00:00 +0000
---
// ToDo: complete code snippets  
// ToDo: consider adding example with Intents  
// ToDo: rename annotation to be @CaptureScreenshots?
// ToDo: figure out how to properly capture screenshots within Rule.
// ToDo: figure out the impact of capturing screenshot after activity has been finished.

If you had experience of writing instrumentation tests for android, you probably heard of tools like Spoon or Composer. Along with orchestration of tests execution, they provide APIs for capturing screenshots which later are placed into generated HTML report. All of these is great, but there are certain associated shortcomings:
 - Screenshot capturing logic adds complexity to tests
 - Hard to replace one library by another

Here is the example of an instrumentation test for a simple form:

```java

@RunWith(AndroidJUnit.class) // ToDo: check the name?
final class FormScreenTest {

  @Rule
  public ActivityTestRule<FormActivity> activityRule =
        new ActivityTestRule(FormActivity.class, false, false);

  private FormRobot formRobot;

  @Before
  public void setup() {
    formRobot = new FormRobot();
  }

  @Test
  public void clickOnSubmitMustNavigateToHome() {
    Spoon.screenshot(activityTestRule.getActivity(), "state_before_edit");

    formRobot.typeTitle("test_title")
        .typeDescription("test_description")
        .submit();

    Spoon.screenshot(activityTestRule.getActivity(), "state_after_edit");
  }

  final class FormRobot {
    FormRobot typeTitle(String title) {
      onView(withId(R.id.title)).perform(typeText(title));
      return this;    
    }  

    FormRobot typeDescription(String description) {
      onView(withId(R.id.description)).perform(typeText(description));
      return this;
    }

    FormRobot submit() {
      onView(withId(R.id.submit)).perform(click());    
    }
  }
}
```

Here I am using `Spoon` to capture screenshots which represent the state before and after user actions. This looks fine, but screenshot capturing logic shouldn't really be a part of the test body. Moreover, it simply becomes tedious to enter tags by hand, when the test name is a perfect candidate for it. This is what we can have instead:

```java
@Test
@CaptureScreenshot
public void clickOnLoginButtonMustNavigateToHome() {
  onView(withId(R.id.title)).perform(typeText("test_title"));
  onView(withId(R.id.description)).perform(typeText("test_description"));
  onView(withId(R.id.submit)).perform(click());
}
```

Looks better huh?
