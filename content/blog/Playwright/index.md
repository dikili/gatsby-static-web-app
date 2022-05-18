---
title: Playwright - is this the way to play right?
date: 2022-04-27 21:40:00 +0300
description: # Add post description (optional)
img: ./playwright.PNG # Add image post (optional)
tags: [] # add tag
---

## Playwright Deep Dive

Playwright has been the latest tool on the market to be offered for writing automation tests, however question I can not keep to ask myself is; we have c# selenium, java selenium, xx selenium, webdriverio, cypress and now Playwright, why would I even bother to think of analysing , assessing and working with another tool anyways? 

This article will be about my experience with the product so not a tutorial or a technical implementation guide alot , if you are looking for this pls visit https://playwright.dev

## What is diffentiating Playwright from the rest

Well, this is the critical question to answer in my mind, because for me it all came down to it. so let me talk about few things I observed in the automation tool and tried to summarize as below;

### API names makes more sense and feels natural

In the past frameworks one issue was that api names or the way to call them did not really feel very natural. 

e.g cy.visit('xxx') - well this is kind of understandable but when I look at this , it just feels more natural and explains what it does page.goto('xxx');

or calling page.locator('text:xxx'); feels much more natural and easier to do (developer friendly) and looks for the element with the 'xxx' text in it , rather than going through all different kinds of selectors and try to work with it. 

another example could be simply, .press('Enter') , it writes simple , reads simple, executes fast (thank you Playwright NodeJS!)

page.click('selector'); also makes alot of sense and is available in playwright. we are not forced to first use the locator and then from that locator grab the element and then apply a click, simply telling the framework i want the page to click, and pass a parameter ,plain and simple!

### Limitations are overcome from the Cypress

To name few, easier to deal with iframes, can now switch tabs, pages ,widgets etc.. in a single test whereas this has been an issue with the Cypress as it works natively inside the executing browser itself. 

### Auto-waiting

Auto-waiting capability was something shining on the product as well. In the evolution of various test frameworks , so far all of them has been somehow less smart than playwright just because of this feature by itself. 

Selenium based c#,java etc... ---> Need to utilize explicit ,implicit waits on all quite extensively which is a massive distraction from writing tests.

Selenium based JS e.g webdriverio ---> JS runs faster compare to c#,java backend tech stack languages, however still element waitings are quite common, and we just add waits to the code until whatever that element or page we are waiting for to interact, and for some reason it is a bit delayed (can be for many many reasons), then there you go, you have a list of flaky tests, now stop automating new tests and make time to fix existing tests so that existing functionality wont break and so that you can just delay failures due to the fact that team could not make time to write coverage on the new features.

Cypress like frameworks which operate natively --> waiting mechanisim has not evolved much still, due to its ability to have a selection of browser and advanced debugging UI screens, and running fast, eliminates many issues others had and more developer friendly. However in order to interact with an element, it is consistently trying to and sending commands to the element and executes by luck when the element is loaded (provided that nothing has timeout)

Playwright --> waiting mechanism much improved, refer to this documentation here;

https://playwright.dev/docs/actionability

It smartly waits for the element to be loaded and then sending the commands, it is not doing this for all the elements however doing it for quite a few and helps with test writing and not ending up with flaky tests alot. For more details as to this area ,pls refer to the link above. 

In a nutshell though, for example, for page.click(selector[, options]), Playwright will ensure that:

element is Attached to the DOM
element is Visible
element is Stable, as in not animating or completed animation
element Receives Events, as in not obscured by other elements
element is Enabled

This was a painpoint in the previous frameworks , because you needed to have all kinds of 'IFS' to make sure, element is visible, enabled , attached, interactable etc.., now it is not the testers' job but the framework will handle it as it should do, and our focus then would be to make sure we have the coverage for that specific area.


### Component testing for SPAs

In the latest version of the playwright 1.22 to be precise, Playwright introduces the ability to work with frontend components and able to interact with them on their component lifecycles e.g mounting, demounting etc.. and looks to be a promising competitor to also cover the areas where React testing lib. , jest and other would do before, for more details pls see the below;

https://playwright.dev/docs/test-components

Note that this is in experimental state as of now.


### What about SNAPSHOTS ?

Well, this is also available readily and easily after installing playwright. And again, you can see in the example below ,how from an english point of view, it reads seamlessly.
We simply expect tha page has the same snapshot of that of landing.png


```
test('example test', async ({ page }) => {
  await page.goto('https://playwright.dev');
  await expect(page).toHaveScreenshot('landing.png');
});
```


Conclusion
----------
 
 I believe this tool looks promising and provides ease of setup and use. Also it can serve as a unifying tool for all kinds of different testing requirements from mobile to API tests. Microsoft owns the tool so there will be a good level of backup too is my guess. It is also able to emulate devices which is yet another great feature with simplicity.