---
layout: post
title:  "Dropped Styles in Aphrodite"
date:   2018-12-06 18:34:39 -0800
categories: jekyll update
---
Do you use [Aphrodite][aphrodite-gh] as your go-to CSS in JS framework? Have you ever had styles disappear on you? If so, you're in the right place! Today I will be describing an issue I encountered where styles would seemingly be dropped or disappear from an aphrodite generated style tag.

To begin, let's start with a really basic example.
{% highlight jsx %}
const Example = () => (
    <div className={css(styles.example)}>
        Example Div
    </div>
);

const styles = StyleSheet.create({
    example: {
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        width: '100px',
        height: '100px',
        backgroundColor: 'red',
    },
});
{% endhighlight %}

In the example above, a 100x100 pixel div is created with the words "Example Div" centered both vertically and horizontally. If you're reading this post, you're likely already familiar with the fact that aphrodite performs auto-prefixing for vendor specific properties. In this case, `display: flex` will expand to `display: -webkit-box`, `display: -moz-box`, `display: -ms-flexbox`, `display: -webkit-flex`, and `display: flex`. The same idea is also used to apply vendor prefixes to `alignItems` and `justifyContent`.

Great right? Flexbox works in other browsers without us having to specify the vendor prefix. But, that's not always the case and was actually the source of an hours worth of frustration for me. So where did I go wrong?

My case involved the property `-webkit-overflow-scrolling`. If you're not familiar, [here][webkit-overflow-scrolling] is the MDN doc for it. It's basically a property for controlling smooth scrolling on elements besides the document body on iOS. This was my first time writing a feature in [Aphrodite][aphrodite-gh] and I naively wrote something like the code below:

{% highlight jsx %}
const Example = () => (
    <div className={css(styles.example)}>
        <div className={css(styles.largeContent)}>
            Some div with a lot of content that scrolls
        </div>
    </div>
);

const styles = StyleSheet.create({
    example: {
        height: '100%',
        overflow: 'scroll',
        overflowScrolling: 'touch',
    },
    largeContent: {
        height: '9999px'
    }
});
{% endhighlight %}

Looks good right? Who needs vendor prefixes if they're automatically added (spoiler alert: I did need the prefix). Anyways, the above code did not work as I expected. I examined the element in Chrome devtools and sure enough, I noticed the stylesheet aphrodite had generated was missing the `overflow-scrolling` property.

After 30 minutes of digging through Aphrodite source code and cursing about how dumb it was for dropping my property, I found the piece of code responsible for injecting the style tag (see below).

{% highlight jsx %}
const injectStyleTag = (cssRules /* : string[] */) => {
    if (styleTag == null) {
        // Try to find a style tag with the `data-aphrodite` attribute first.
        styleTag = ((document.querySelector("style[data-aphrodite]") /* : any */) /* : ?HTMLStyleElement */);

        // If that doesn't work, generate a new style tag.
        if (styleTag == null) {
            // Taken from
            // http://stackoverflow.com/questions/524696/how-to-create-a-style-tag-with-javascript
            const head = document.head || document.getElementsByTagName('head')[0];
            styleTag = document.createElement('style');

            styleTag.type = 'text/css';
            styleTag.setAttribute("data-aphrodite", "");
            head.appendChild(styleTag);
        }
    }

    // $FlowFixMe
    const sheet = ((styleTag.styleSheet || styleTag.sheet /* : any */) /* : CSSStyleSheet */);

    if (sheet.insertRule) {
        let numRules = sheet.cssRules.length;
        cssRules.forEach((rule) => {
            try {
                sheet.insertRule(rule, numRules);
                numRules += 1;
            } catch(e) {
                // The selector for this rule wasn't compatible with the browser
            }
        });
    } else {
        styleTag.innerText = (styleTag.innerText || '') + cssRules.join('');
    }
};
{% endhighlight %}

I stepped through the code, line by line and watched as the `overflow-scrolling` property got inserted into the stylesheet. I checked the final `<style>` tag that got inserted into the DOM, and the property was knowhere to be found. As a sanity check, I performed the same sequence of code in my browser console, but isolated everything to just that one rule. Same result again. At this point it clicked for me. I tried a couple of other bogus named properties and came to the conclusion that the [StyleSheet API][stylesheet-api] itself was responsbile for dropping properties. In regular CSS stylesheets, if a browser doesn't recognize a rule, it will simply ignore it. Here the browser doesn't just ignore it, but instead *completely tosses* the rule.

At this point, the problem was clear to me. If I wanted this to work, I needed to make sure the property got fed into the [StyleSheet API][stylesheet-api] correctly. As you may have noticed, Aphrodite didn't apply any vendor prefixing to `overflowScrolling`. This was my next point of interest. Why wasn't aphrodite adding any vendor prefixes? I tried digging through the source code to try and understand why it wasn't getting prefixed, but ultimately got burned out on trying to track it down. The conclusion I came to is that because the property is specific to webkit, aphrodite doesn't recognize it without the prefix to begin with.

Okay, so the solution is easy now. Aphrodite allows you to pass property names as their exact string literal in the stylesheet creation:

{% highlight jsx %}
    ...

    example: {
        height: '100%',
        overflow: 'scroll',
        '-webkit-overflow-scrolling': 'touch',
    },

    ...
{% endhighlight %}

This *is* in fact the correct solution, but I'm embarassed to say that even with all this in place, I still chased my tail for a bit longer. As I mentioned before, the [StyleSheet API][stylesheet-api] drops properties it doesn't recognize. I made the mistake of trying to verify this solution on Chrome (because -webkit, right?), but I forgot the property only exists for iOS. Once again the style was being dropped! But, like I said, that's expected since Chrome doesn't recognize the property. Once I validated the solution on iOS, all was good again.

So the moral of the story is, if your styles are being dropped in your aphrodite stylesheets:
* Don't blame Aphrodite right away
* Verify the property has a non-vendor prefix name
* If all else fails, use the **exact** property name.
* Validate on the target browser.

[aphrodite-gh]: https://github.com/Khan/aphrodite
[webkit-overflow-scrolling]: https://developer.mozilla.org/en-US/docs/Web/CSS/-webkit-overflow-scrolling
[stylesheet-api]: https://developer.mozilla.org/en-US/docs/Web/API/StyleSheet

