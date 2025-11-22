```ic-metadata
{
  "name": "A Framework-Agnostic and Source-Aware Approach to E2E Test Generation With AI",
  "series": null,
  "date": "2025-11-22",
  "lastModifiedDate": "2025-11-22",
  "author": "Volodymyr Yepishev",
  "tags": ["ai", "tutorial", "e2e"],
  "canonicalLink": ""
}
```

# A Framework-Agnostic and Source-Aware Approach to E2E Test Generation With AI

## Well, what is it all about?

In this article we will be discussing how real user workflows can be transformed into living documentation, tutorials and e2e test in a framework of your choice.

Yet, let us start with the why. AI tools have significantly accelerated code generation, which, though of questionable quality, still often achieves business goals and in this way deserves to exist. More shipped code require more testing, which puts a strain on manual testers and testers writing e2e tests using various framework, like Selenium, Playwright or others. This is usually either done by writing code or generating specs from actions in a browser using the appropriate framework's tools, like Selenium IDE extension, Playwright codegen, Cypress browser, etc. This allows decoupling of e2e tests from the codebase and development itself.

Yet, when e2e testing is completely decoupled from development itself, it can sometimes produce flaky tests, when something breaks just because component tree was changed, css class got renamed or a placeholder on an input received an update. Moreover, writing test is additional work on top of work: during the development process a software engineer manually tests a feature, which is then manually tested during the demo, manually tested by QA, often even before the e2e test is written. Essentially we end up with multiple people running same test on and on, which goes to waste. Then if it breaks at some point, it casts shadow on everyone in the testing chain with questions why the issue was not found. By that time no one remembers what was tested, or how it was tested, maybe even people who tested or developed it do not work anymore.

Additionally, there is training to be done to teach clients how to use new features, documentations, wikis, word of mouth in the development team. Documentation is something undoubtedly no one likes to maintain.

How could it be improved? Let us consider how it can be added into the development and testing process, how we could be contributing to generating test scenarios, documentation and training materials without working on those as separate items.

Here is the idea: the process of manual testing needs to be recorded and use to generate a living documentation, contribute to writing documentation and e2e unit testing. However, the type of recording would have to be not the video, but incremental changes done to the app as the user interacts with it. The record is further used for e2e test case generation, tutorial, documentation or any other purpose. If made accessible, this recording process could be used by anyone, be it QA, PO or even a Frontend Developer. For the e2e test case generation the pipeline would be the following

```text
use records session => the record sent to developer => developer verifies the record => e2e test case is generated from the record
```

## Good on paper, but how?

For this approach to work, session recording should be something simple and accessible, the user should not be technically savvy to be able to produce the record, otherwise it would be too daunting. This leaves out all testing framework, as they have entry barrier. Yet, recording a session is fairly easy task with proper tools and could be oversimplified with a bookmarklet using `rrweb` to record user action.

Consider the following bookmarklet that on initial click dybamically loads `rrweb` and starts recording user actions, and on the second click exports the record as `JSON` file for the user to save. It requires no technical knowledge apart of how to add a bookmark to the browser and could be used on any website in any environment. Are you a developer who wants to run the final set of tests on a feature prior to creating a PR? You will do it manually anyway, why not create a `JSON` record of it? More on how the produced `JSON` can be used further.

```javascript
javascript: (function () {
  // --- Prevent duplicate script injection ---
  function loadRRWebOnce(cb) {
    if (window.__rrwebLoading) {
      alert("rrweb is still loading…");
      return;
    }
    if (window.rrweb) {
      return cb(true);
    }

    window.__rrwebLoading = true;
    alert("Installing rrweb…");

    const urls = [
      "https://unpkg.com/rrweb/dist/rrweb-all.min.js",
      "https://cdn.jsdelivr.net/npm/rrweb/dist/rrweb-all.min.js",
    ];

    function loadScript(url, next) {
      if (document.querySelector('script[data-rrweb-script="' + url + '"]')) {
        return next(); // already added
      }
      const s = document.createElement("script");
      s.src = url;
      s.dataset.rrwebScript = url;
      s.onload = next;
      s.onerror = next;
      document.head.appendChild(s);
    }

    (function tryLoad(i) {
      if (i >= urls.length) {
        window.__rrwebLoading = false;
        return cb(false);
      }
      loadScript(urls[i], function () {
        if (window.rrweb) {
          window.__rrwebLoading = false;
          alert("rrweb installed!");
          return cb(true);
        }
        tryLoad(i + 1);
      });
    })(0);
  }

  // --- If recording already active: STOP & DOWNLOAD ---
  if (window.rrwebStopFn) {
    alert("Stopping rrweb recording…");

    const data = JSON.stringify(window.rrwebEvents || [], null, 2);
    const blob = new Blob([data], { type: "application/json" });
    const url = URL.createObjectURL(blob);

    const a = document.createElement("a");
    a.href = url;
    a.download =
      "rrweb-events-" + new Date().toISOString().replace(/[:]/g, "-") + ".json";
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);

    window.rrwebStopFn();
    window.rrwebStopFn = null;

    alert("Recording stopped & file downloaded.");
    return;
  }

  // --- Otherwise: START recording ---
  loadRRWebOnce(function (ok) {
    if (!ok) {
      alert("Could not load rrweb (blocked by browser or extension).");
      return;
    }

    window.rrwebEvents = [];
    window.rrwebStopFn = rrweb.record({
      emit: function (ev) {
        window.rrwebEvents.push(ev);
      },
    });

    alert(
      "rrweb recording started.\nRun bookmarklet again to stop & download."
    );
  });
})();
```

The produced `JSON` file contains all the incremental changes to the web page, it documents all interactions with the page and how it was changed. It can be replayed in a special player, as if it was a video. This already has a value because you produced learning material for the users without any screen capturing software.

The next step would be to pass the record to a developer with appropriate notes on what is tested there. It would be difficult to overstate the value of this record, as it can be passed to AI with appropriate instructions to generate an e2e test, to write a tutorial or documentation page.

Now here is what is special about this approach: you can pass the record to AI along with reference to the codebase and ask it to generate appropriate data-testid attributes where necessary, to avoid using fragile selectors for buttons, placeholders, etc. Yet, it gives the developer something to iterate on until the test is green. Moreover, this approach is test framework agnostic, the collected user interactions record is a completely separate thing, not tied to framework shenanigans.

What is even more interesting, it is not tied to any particular AI provider. The model does not really matter, as long as it is capable of generating E2E tests using the framework of choice. AI usage make it more accessible even to developers, not particularly familiar with the e2e testing.

## Examples

I used the approach in this [repo](https://github.com/Bwca/testing_e2e-test-code-generation-with-ai) to generate e2e tests for Cypress and Playwright from `rrweb` screen recordings using AI to analyze them and add `data-qa` attributes to the codebase to be used as selectors, opposite to using XPath, classes, ids or etc. By examining the record and the codebase AI manages to navigate even React codebase (which does not have explicit component selectors like Angular).
I had 4 recordings: add employee, use navigation, try to delete employee and not confirm, and the last one to delete the employee.

## Conclusion

With some imagination, e2e testing and documentation could be an exciting team effort, where everyone contributes to quality and maintenance.
