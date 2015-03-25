---
layout: post
title: "Using Roboelectric for JVM-based Android Unit Testing"
date: 2015-03-25 10:28:00
comments: true
---

One of my biggest aggravations with Android development is the length of time it takes to run simple unit tests on a device or emulator.  The compile/package/deploy/test cycle just takes too long to run on a regular basis, and so I tend to drift away from good unit testing practice, and towards more integration or acceptance test process.  This means that I frequently end up navigating the application to the area that I actually want to test, slowing down the development process significantly.

Recently I decided to actively improve my development practices, including implementing Continuous Integration and finding a way of returning to a more test-driven approach.

Enter [Roboelectric](http://robolectric.org).  From the website:

>Running tests on an Android emulator or device is slow! Building, deploying, and launching the app often takes a minute or more. That’s no way to do TDD. There must be a better way.

>Wouldn’t it be nice to run your Android tests directly from inside your IDE? Perhaps you’ve tried, and been thwarted by the dreaded java.lang.RuntimeException: Stub!?

>Robolectric is a unit test framework that de-fangs the Android SDK jar so you can test-drive the development of your Android app. Tests run inside the JVM on your workstation in seconds. With Robolectric you can write tests like this:

```java
// Test class for MyActivity
@RunWith(RobolectricTestRunner.class)
public class MyActivityTest {
  @Test
  public void clickingButton_shouldChangeResultsViewText() throws Exception {
    Activity activity = Robolectric.buildActivity(MyActivity.class).create().get();

    Button pressMeButton = (Button) activity.findViewById(R.id.press_me_button);
    TextView results = (TextView) activity.findViewById(R.id.results_text_view);

    pressMeButton.performClick();
    String resultsText = results.getText().toString();
    assertThat(resultsText, equalTo("Testing Android Rocks!"));
  }
}
```

This was just what I was looking for, so I set about implementing it into the [test project](https://github.com/jasongoff/AndroidMaven) I created for my [Continuous Integration]({% post_url 2015-03-23-Building Android In The Cloud 1 %}) posts.

The first step was to add the required dependencies to my Maven POM

```xml
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.robolectric</groupId>
            <artifactId>robolectric</artifactId>
            <version>2.4</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.android.support</groupId>
            <artifactId>support-v4</artifactId>
            <version>19.0.1</version>
            <scope>test</scope>
        </dependency>

```
Robolectric requires the Google APIs for Android (specifically, the maps JAR) and Android support-v4 library. I used the SDK manager to ensure these were downloaded onto my development machine, and then installed them into my local Maven repository:

```
mvn install:install-file -DgroupId=com.google.android.maps \
  -DartifactId=maps \
  -Dversion=18_r3 \
  -Dpackaging=jar \
  -Dfile="$ANDROID_HOME/add-ons/addon-google_apis-google-18/libs/maps.jar"

mvn install:install-file -DgroupId=com.android.support \
  -DartifactId=support-v4 \
  -Dversion=19.0.1 \
  -Dpackaging=jar \
  -Dfile="$ANDROID_HOME/extras/android/m2repository/com/android/support/support-v4/19.0.1/support-v4-19.0.1.jar"
```
I also added these as Pre-Build tasks to my [cloud Jenkins server]({% post_url 2015-03-23-Building Android In The Cloud 3 %}) after downloading the relevant packages.

```
sudo ./tools/android update sdk -u -a -t 26,93
```

Once installed, I created a simple test class in the project to hold my unit tests.

```java
@RunWith(RobolectricTestRunner.class)
@Config(manifest = "./AndroidManifest.xml", emulateSdk = 18)
public class HelloAndroidActivityTest {

  private Activity activity;
  private Button button3;
  private Button button;

  @Before
  public void setUp() throws Exception {
    activity = Robolectric.setupActivity(HelloAndroidActivity.class);
    button3 = (Button) activity.findViewById(R.id.button3);
    button = (Button) activity.findViewById(R.id.button);
  }

  @Test
  public void testJunit() {
    assertTrue(true);
  }

  @Test
  public void titleIsCorrect() {
    assertTrue(activity.getTitle().toString().equals("Hello Android!"));
  }

  @Test
  public void buttonCaptionIsCorrect() {
    assertEquals("Activity 2", button.getText());
  }

  @Test
  public void thatButton3CaptionIsCorrect() {
    assertEquals("Activity 3", button3.getText());
  }

  @Test
  public void thatButtonStartsSecondScreen() {
    button.performClick();
    Intent expectedIntent = new Intent(activity, SecondActivity.class);
    assertThat(shadowOf(activity).peekNextStartedActivity(), is(expectedIntent));
  }

  @Test
  public void thatButton3StartsThirdScreen() {
    button3.performClick();
    Intent expectedIntent = new Intent(activity, ThirdActivity.class);
    assertThat(shadowOf(activity).peekNextStartedActivity(), is(expectedIntent));
  }
}
```
Here I used Roboelectric's `setupActivity` method to create a mocked version of my main application activity.  Within the click tests, I used Roboelectric's `shadowOf` method to get the shadow object created by Roboelectric.  These shadows allow my tests to access state that normally would not be available.  I then determine whether the intent created by the shadow is the one I would expect the application to create when I clicked a button.

## API Support
The only drawback with Roboelectric that I have encountered so far is that it does not support all Android API levels.

According to the [source on Github](https://github.com/robolectric/robolectric/blob/master/robolectric/src/main/java/org/robolectric/internal/SdkConfig.java), the following APIs are supported:

```java
  static {
    SUPPORTED_APIS = new HashMap<Integer, SdkVersion>();
    SUPPORTED_APIS.put(Build.VERSION_CODES.JELLY_BEAN, new SdkVersion("4.1.2_r1", "0"));
    SUPPORTED_APIS.put(Build.VERSION_CODES.JELLY_BEAN_MR1, new SdkVersion("4.2.2_r1.2", "0"));
    SUPPORTED_APIS.put(Build.VERSION_CODES.JELLY_BEAN_MR2, new SdkVersion("4.3_r2", "0"));
    SUPPORTED_APIS.put(Build.VERSION_CODES.KITKAT, new SdkVersion("4.4_r1", "1"));
    SUPPORTED_APIS.put(Build.VERSION_CODES.LOLLIPOP, new SdkVersion("5.0.0_r2", "1"));
  }
```
However I have been unable to get it working with any API beyond 18.  To override the `targetSdkVersion` in the AndroidManifest, I used the` emulateSdk = 18` in the Roboelectric class annotations.

```java
@RunWith(RobolectricTestRunner.class)
@Config(manifest = "./AndroidManifest.xml", emulateSdk = 18)
public class HelloAndroidActivityTest {
```

Using Roboelectric's test runner in this way has enabled me to write JVM-based unit tests, and also to include test execution and coverage reporting in my Jenkins & Maven setup.

