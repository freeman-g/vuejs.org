---
title: Avoiding Memory Leaks
type: cookbook
order: 9
---
## Introduction

If you are developing applications with Vue, then you need to watch out for memory leaks. This issue is especially important in Single Page Applications (SPAs) because by design, users should not have to refresh their browser when using an SPA, so it is up to the Javascript application to clean up components and make sure that garbage collection takes place as expected.

Memory leaks in Vue applications do not typically come from Vue itself, rather they can happen when incorporating other libraries into an application.

## Simple Example

The following example shows a memory leak caused by using the Choices.js library in a Vue component (and not properly cleaning it up). Later, we will show how to remove the Choices.js footprint and avoid the memory leak.

In the example below, we load up a select with a lot of options and then we use a show/hide button with a `v-if` directive to add it and remove it from the virtual DOM. The problem with this example is that the v-if directive removes the parent element from the DOM, but we did not clean up the additional DOM pieces created by Choices.js, causing a memory leak. 

To see this memory leak in action, open this [CodePen example](https://codepen.io/freeman-g/pen/qobpxo) and then, using Chrome, open the Chrome Task Manager (Chrome Menu > More Tools > Task Manager). Now, click the show/hide button 50 or so times. You should see the memory usage in the Chrome Task Manager increase and never be reclaimed.

```
<link rel='stylesheet prefetch' href='https://joshuajohnson.co.uk/Choices/assets/styles/css/choices.min.css?version=3.0.3'>
<script src='https://joshuajohnson.co.uk/Choices/assets/scripts/dist/choices.min.js?version=3.0.3'></script>

<div id="app">
  <button v-if="showChoices" @click="hide">Hide</button>
  <button v-if="!showChoices" @click="show">Show</button>
  <div v-if="showChoices">
    <select id="choices-single-default"></select>
  </div>
</div>

const app = new Vue({
  el: "#app",
  data: function() {
    return {
      showChoices: true
    }
  },
  mounted: function() {
    this.initializeChoices()
  },
  methods: {
    initializeChoices() {
      let list = []
      // lets load up our select with a lot of choices so it will use a lot of memory
      for (let i = 0; i < 1000; i++) {
        list.push({
          label: "Item " + i,
          value: i
        })
      }
      new Choices("#choices-single-default", {
        searchEnabled: true,
        removeItemButton: true,
        choices: list
      })
    },
    show() {
      this.showChoices = true
      this.$nextTick(() => {
        this.initializeChoices()
      })
    },
    hide() {
      this.showChoices = false
    }
  }
})

```

In the above example, we can use our `hide()` method to do some clean up and solve the memory leak prior to removing the select from the DOM. To accomplish this, we will keep a property in our Vue instance’s data object and we will use the [Choices API’s](https://github.com/jshjohnson/Choices) `destroy()` method to perform the clean up.

Check the memory usage again with this [updated CodePen example](https://codepen.io/freeman-g/pen/mxWMor).

```
const app = new Vue({
  el: "#app",
  data: function() {
    return {
      showChoices: true,
      choicesSelect: null
    }
  },
  mounted: function() {
    this.initializeChoices()
  },
  methods: {
    initializeChoices() {
      let list = []
      for (let i = 0; i < 1000; i++) {
        list.push({
          label: "Item " + i,
          value: i
        })
      }
      // Set a reference to our choicesSelect in our Vue instance's data object
      this.choicesSelect = new Choices("#choices-single-default", {
        searchEnabled: true,
        removeItemButton: true,
        choices: list
      })
    },
    show() {
      this.showChoices = true
      this.$nextTick(() => {
        this.initializeChoices()
      })
    },
    hide() {
      // now we can use the reference to Choices to perform clean up here 
      // prior to removing the elements from the DOM
      this.choicesSelect.destroy()
      this.showChoices = false
    }
  }
})
```

## Additional Context

In the above example, we used a `v-if` directive to illustrate the memory leak, but a more common real-world example happens when using **vue-router** to route to components in a Single Page Application.

Just like the v-if directive, vue-router removes elements from the virtual DOM and replaces those with new elements when a user navigates around your application. The Vue `beforeDestroy()` [lifecycle hook](https://vuejs.org/v2/guide/instance.html#Lifecycle-Diagram) is a good place to solve the same sort of issue in a vue-router based application.

We could move our clean up into the `beforeDestroy()` hook like this:

```
mounted: function() {
  ...
},
beforeDestroy: function() {
    this.choicesSelect.destroy()
}
```
## Wrapping Up

Vue makes it very easy to develop amazing, reactive Javascript applications, but you still need to be careful about memory leaks. These leaks will often occur when using additional 3rd Party libraries that also manipulate the DOM outside of Vue. Make sure to test your application for memory leaks and take appropriate steps to clean up components where necessary.