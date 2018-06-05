---
layout: post
title: "Hacking Atom: Adding a tab in the \"Settings\" Panel"
date:   2017-02-26 09:55:05 +0200
categories: atom hacking
author: giulio
---

Atom is known to everybody as the "Hackable to the core" editor. And it really is. Provided that you understand a little bit of CoffeeScript, JavaScript and CSS (actually LESS), and that you have the patience to untangle its source code, you have at your hands one of the most powerful editors in the open-source community. In this post I will explain how to add a new lateral tab in the classic settings panel. This option may be very useful if your package needs a more advanced configuration page, which may be too complex to be rendered in the default package settings.

{% include image.html url="/assets/images/atom-settings-tab.png" %}

Anyway be aware that there's a reason why Atom developers haven't made such a feature available trough public API, in fact a clogged settings view UI is certainly something that we shouldn't want.

### Looking at the code

To add a new tab in the settings view two approaches are possible:

- directly modifying the dom and inserting a new element in the panel;
- trying to exploit some function in the settings-view package code.

Since the first option is too complicate to implement and often too unreliable, and since the fact that we can easily explore how the code works, what we're trying to do is the second option.

Looking at the source code we can find in the [settings-view.js](https://github.com/atom/settings-view/blob/master/lib/settings-view.js) file the function "addCorePanel", which is used in the constructor to populate the left tabs. So if we're able to call this from outside the settings view, we can add as many additional panels as we want.

The settings-view module provides the item that is going to be placed in the new workspace panel, so we can try to intercept it with the [atom.workspace.observeActivePaneItem()](https://atom.io/docs/api/v1.17.2/Workspace#instance-observeActivePaneItem). This function invokes the given callback with the current active pane item and with all future active pane items in the workspace. This enables us to intercept any item being displayed, which then we need to filter by URI:

{% highlight javascript linenos %}
// Inject the new panel in settings-view when we open the settings
atom.workspace.observeActivePaneItem((item) => {
  // Avoid any possible undefined errors
  if (item && item.uri && item.uri == 'atom://config') {
    // DO SOME HACKS HERE
  }
});
{% endhighlight %}

Once we've determined that our item is a settings-view item, we can call on it the addCorePanel method, which takes as arguments the panel name, the icon name:

{% highlight javascript linenos %}
// Inject the new panel in settings-view when we open the settings
atom.workspace.observeActivePaneItem((item) => {
  // Avoid any possible undefined errors
  if (item && item.uri && item.uri == 'atom://config') {
    item.addCorePanel("Demo", 'database', () => new Panel());
  }
});
{% endhighlight %}

the Panel class is the actual new element, (a etch component) to be displayed in the settings view. The base class is the following:

{% highlight javascript linenos %}
'use babel'
// Necessary for render syntax to work
/** @jsx etch.dom */

import {CompositeDisposable} from 'atom'
import etch from 'etch';

export default class Panel {

  constructor () {
    // Initialize etch
    etch.initialize(this);
  }

  /**
   * Default etch api destroy
   */
  destroy () {
    return etch.destroy(this);
  }

  /**
   * Default etch api update
   */
  update (props, children) {
    return etch.update(this);
  }

  /**
   * Default etch api render
   * @return {Object} HTML dom element
   */
  render () {
    return (
      <div className='panels-item menu-editor' tabIndex='-1'>
        <section className='section'>
          <div className='section-container'>
            <div className='section-heading icon icon-database'>Hello world</div>
            <div>Hello world</div>
          </div>
        </section>
      </div>
    );
  }

  /****************************************************
   * Default functions present in settings-view panes *
   ****************************************************/

  focus () {
    this.element.focus()
  }

  show () {
    this.element.style.display = ''
  }

  scrollUp () {
    this.element.scrollTop -= document.body.offsetHeight / 20
  }

  scrollDown () {
    this.element.scrollTop += document.body.offsetHeight / 20
  }

  pageUp () {
    this.element.scrollTop -= this.element.offsetHeight
  }

  pageDown () {
    this.element.scrollTop += this.element.offsetHeight
  }

  scrollToTop () {
    this.element.scrollTop = 0
  }

  scrollToBottom () {
    this.element.scrollTop = this.element.scrollHeight
  }
}
{% endhighlight %}

There's just one last thing to do: since we inject this new panel every time it becomes active, in order not to have multiple copies of it we have to use a flag to avoid that:

{% highlight javascript linenos %}
// Inject the new panel in settings-view when we open the settings
atom.workspace.observeActivePaneItem((item) => {
  // Avoid any possible undefined errors
  if (item && item.uri && item.uri == 'atom://config') {
    if (!item.panelAdded) {
      item.addCorePanel("Demo", 'database', () => new Panel());
      item.panelAdded = true;
    }
  }
});
{% endhighlight %}
