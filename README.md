pytest-appium
=============

`pytest-appium` is a plugin for [pytest](https://docs.pytest.org/en/latest/) that provides support for running [Appium](http://appium.io/) based tests.

It is inspired by [pytest-selenium](https://github.com/pytest-dev/pytest-selenium)

This package has a suite of additional utilities

* HTML Reports
* Augment the base Appium `driver` with additional modular functionality with python mixins.
* Wait for Appium to be ready
* `pytest.mark.platform`
* Light python builders for Android [UiSelector](https://developer.android.com/reference/android/support/test/uiautomator/UiSelector.html)


Setup
-----

pip requirements file
```
    git+http://GIT.SERVER.LOCATION/pytest-appium.git@master#egg=pytest-appium
```

pytest.ini
```
    [pytest]
    addopts = -p pytest_appium
```

python test example
```python
    from pytest_appium.android.UIAutomator2 import UiSelector, UiScrollable
    from pytest_appium.driver.proxy._enums import Direction

    @pytest.mark.platform('ios')
    def test_example(appium_extended, platform):
        el = appium_extended.find_element_on_page(
            UiSelector().resourceIdMatches('.*awesome_item'),
            direction=Direction.LEFT,
        )
        assert el.text == 'Example'
```

commandline example
```bash
    $(PYTHON_VENV)/bin/py.test \
        $(TEST_PATH) \
        --html=$(REPORT_PATH)/report.html \
        --junitxml=$(REPORT_PATH)/report.xml \
        --variables variables/pytest/android_emulator_local.json \
        --capability app $(APPIUM_APK_PATH)
```
for more commandline details use `pytest --help` under the heading `appium`


Features
--------

### HTML reports

[pytest-html](https://pypi.python.org/pypi/pytest-html/) for Appium tests.

Includes screenshot, page `XML` and full `logcat` dumps.


### Wait for startup conditions

Orchestration of service startup order is sometimes problematic.
On startup we can wait for a varity of conditions.
`pytest --help` for details

```
    --appium_wait_for_condition={appium,android_device_available}
                            Type of appium condition to wait for before beginning
                            first test
    --appium_wait_for_seconds=APPIUM_WAIT_FOR_SECONDS
                            Seconds to wait for an Appium server to become
                            available before raising an error.
```

* `appium` - Waits for the Appium port to become active
* `android_device_available` - Trys to send a low level `wd` api call to launch an Android packagename "NOT-REAL". We know an android device is connected and available if the error message explicitly contains "NOT-REAL".

Example of use in a `docker-compose.yml` waiting for android device.
```yaml
    command:
      - pytest
      ...
      - --appium_host=android-container
      - --appium_wait_for_seconds=90
      - --appium_wait_for_condition=android_device_available
      ...
```

Exampe of use in a `bash` script waiting for appium service to become active.
```bash
    ...
    (nohup node $(APPIUM_PATH) $(APPIUM_ARGS) > $(REPORT_FOLDER)/appium.log &)
    ...

    _env/bin/py.test \
        tests \
        ...
        --appium_host=localhost \
        --appium_wait_for_condition appium \
        --appium_wait_for_seconds 30 \
        ...

```

### pytest Markers for platforms

```python
    @pytest.mark.platform('ios')
    def test_ios(appium_extended):
        pass

    @pytest.mark.platform('android')
    def test_android(appium_extended):
        pass

    def test_both_ios_and_android(appium_extended):
        pass
```

### Python wrapper for expressing UiSelector syntax in python

Handling long android strings to compose UiSelectors is inflexible. A lightweight UiSelector python builder is provided

```python
    from pytest_appium.android.UIAutomator2 import UiSelector, UiScrollable

    # Create a UiSelector python builder
    selector = UiSelector().resourceIdMatches('.*filmstrip_list').childSelector(UiSelector().index(1))
    assert str(selector) == 'new UiSelector().resourceIdMatches(".*filmstrip_list").childSelector(new UiSelector().index(1))'

    # UiSelector objects can be used directly
    el = self.driver.find_element_by_android_uiautomator(selector)

    # UiSelectors can have segments modified/appended after creation
    selector.ui_objects[-1].childSelector(UiSelector().className('my_class'))
    assert str(selector) == 'new UiSelector().resourceIdMatches(".*filmstrip_list").childSelector(new UiSelector().index(1).childSelector(new UiSelector().className("my_class")))'
    self.driver.find_element_by_android_uiautomator(selector)
```

### Appium Driver Extension Framework

The base Appium [python-client](https://github.com/appium/python-client) `driver` supports base functions.
We sometimes want to add extra functionality to this `driver` for individual platforms.

We can transparently overlay extra Mixin's over the base `driver` object. This is available as the pytest fixture `appium_extended`.

```python
    from pytest_appium.driver.proxy.proxy_mixin import register_proxy_mixin

    @register_proxy_mixin(name='android')
    class MyAndroidDriverMixin():
        def new_thing(self, text):
            log.debug('my mixin method for android')
            # modify the selector in some cool way
            return self.find_element(*my_cool_selector_with_text)

    @register_proxy_mixin(name='ios')
    class MyIOSDriverMixin():
        def new_thing(self, text):
            log.debug('my mixin method for ios')
            # modify the selector in some cool way
            return self.find_element(*my_cool_selector_with_text)

    def test_my_mixin(appium_extended):
        el = appium_extended.new_thing('text of awesome')

    # The `appium` fixture can be used to get the base `driver` without mixin augmentation.
    @pytest.mark.xfail(strict=True)
    def test_my_mixin_without_extension(appium):
        el = appium.new_thing('text of awesome')

```

Omit the `name='platform'` argument to allow the mixin to augment all platforms.

Note: `dir(appium_extended)` will *NOT* reveal your additional mixin methods. They are invisible. (This could be improved in a future version)


### `appium_extended` Driver Extensions Summary

| platform | method | description |
|----------|--------|-------------|
| all | .platform | return a string of the platform name (android, ios) |
| all | .wait_for() | wait for element to become visible |
| all | .find_element_safe | same as find_element but returns None rather than throw exception |
| all | .get_element_bounds | return dict of derived location information of element |
| all | .swipe_element | swipe in a direction |
| all | .find_element_on_page | swipe up/down left/right looking for an element. Similar to android UiScrollable but platform dependent |
| android | .find_element_by_android_uiautomator | accepts UiSelector python objects |
| android | .scroll_to_element_by_android_uiautomator | similar to find_element_on_page |
| android | .back, .home, .app_switcher, .background_app | send android keycodes |
| android | .wait_for_webview | wait for a webview to become active and populated |
| ios | | in progress |
