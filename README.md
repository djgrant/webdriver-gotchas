# Webdriver.io gotchas

### Selectors

Not all selectors will be recognised by every browser's webdriver. 
- CSS3 selectors probably won't run in old IE browsers. 
- Careful using selectors that return more than 1 element, you can't click both
- Special characters need to be escaped e.g. `deluxe:deluxe-box` -> `deluxe\\3A deluxe-box`, see https://mothereff.in/css-escapes

### Form state

Avoid using getAttribute to check form state e.g. `getAttribue('checked')` - not all browsers will add the attribute.

Instead use `isSelected()`.

### Waiting

Sometimes you need to wait before making an assertion or proceeding with the test suite. 

Prefer using a `wait` function over `pause()`. It's better to wait and proceed as soon as possible than to block the whole suite. 

Even better, you can set an implicit timeout in web driver config that determines how long the driver will for an element to be retrievable on the page before timing out ([see spec](https://w3c.github.io/webdriver/webdriver-spec.html#dfn-session-implicit-wait-timeout)).

Make sure you check that your wait functions work in remote browsers.

**Do not make `should()` assertions inside `waitUntil()`** as it will cause the test to fail if the predicate does not immediately return true.

For animations, typically you can interact with the element immediately even if it is still transitioning into view. If not, `waitForVisible`.

### Loading pages

In most browsers when you load a page (using `browser.element('#form').submit()`, `browser.element('#link').click()` etc.) the execution of the test is paused until the page has loaded. In most browsers.

In some browsers, particularly MS Edge, the test will continue so it is necessary to put a wait check in place immediately after the page load. (`browser.url()` seems to pause execution consistently across browsers).

Furthermore, especially in old IEs, pages will sometimes take long enough to load (or load, but not properly resolve) that the test times out. This is particularly awkward because rerunning the test will fail as the original page will not longer exist in the session. 

Recommended pattern: 
- If the page is being loaded at the end of the test suite, wait for the expected url. (This checks that the page you are testing has fulfilled it's responsibility of opening the url of the next page).
- If the page is being loaded at the beginning of the suite, wait for the DOM to be ready by checking for the presence of a given element. (This ensures that the rest of the suite can interact with that page).
- In end to end suites, load pages / wait for resolution in before/after blocks. (This avoids timeouts causing a test failure - if the page did actually fail to load the latter tests will fail).

Example:

```js
describe('Cart', function () {

  before('loads the page', function () {
    Cart.openWithDeluxeBook();
    Cart.checkoutButton.waitForVisible();
  });

  it('does something', function () {
    // things to check
  });

  after('goes to /checkout/details', function () {
    Cart
      .checkoutButton()
      .click()
      .waitUntil(() =>
        browser
          .getUrl()
          .includes(CheckoutDetails.url)
      );
  });
});

describe('Checkout', function () {
  // At the end of the end to end suite 
  // make sure you check the final page has actually loaded.
})
```

### Timeouts

Tests running on the cloud need lengthy timeouts to allow enough time for pages to load. A page might take up to 10 seconds to load so test timeouts are set at 20 seconds. With that in mind, only load 1 page per test.

Webdriver commands may still get executed after their test has timed out. 

Further reading: http://webdriver.io/guide/testrunner/timeouts.html

### Sync execution

Since v4 tests are written in a synchronous style. Using the [fibers](https://github.com/laverdet/node-fibers) package, wherever there is a promise the test runner will automagically return the resolved value. 

Instead of:
```js
browser
  .getTitle()
  .then(title => title.should.equal('Title'));

browser
  .element('#node')
  .isVisible()
  .then(isVisible => isVisible.should.be.true());
``` 

You can write:

```js
browser
  .getTitle()
  .should.equal('Title'));

browser
  .element('#node')
  .isVisible()
  .should.be.true());
``` 
See: http://webdriver.io/guide/getstarted/v4.html

If you would like to still write tests without the sync style, you can do so by naming test block functions `async`:

```js
it('should do a thing', function async() {
  // promises inside function named `async` not transformed by fibers
  return browser.waitUntil(function () {
      // inner function is not named `async`
      // so is transformed by fibers, reverting to sync behaviour
      return browser
        .getUrl()
        .includes(Cart.url);
    });
});
```



Note that async tests don't seem to be reported by the spec reporter.

See: http://webdriver.io/guide/getstarted/upgrade.html

### Chaining methods

Typically methods that perform an action will return the browser object and further methods can be chained to them. 

This can be counter-intuitive as you'd expect `Cart.button.click()` to return the button element (like jQuery), instead, you'll get the browser object.

Methods that return a value e.g. `getTitle`, `isVisible` cannot have further methods chained to them. Some methods you probably won't expect to return a value e.g. `waitUntil`, also cannot have further methods chained to them either.

## Resources 

Adding new browsers

* https://wiki.saucelabs.com/display/DOCS/Platform+Configurator#/

IE Woes 

* https://support.saucelabs.com/customer/en/portal/articles/2143851-tips-and-tricks-for-using-selenium-with-internet-explorer
* https://www.joecolantonio.com/2014/11/20/selenium-and-ie-getting-them-to-work-together/

Sauce/travis integration

* http://webdriver.io/guide/services/sauce.html 
* https://docs.travis-ci.com/user/sauce-connect/

General

* http://webdriver.io
* http://elementalselenium.com/tips
* https://github.com/webdriverio/webdriverio/tree/master/examples
