---
title: "Some project notes: Use React and webpack"
categories: ["tech"]
date: 2015-08-12
tags: ["javascript"]
---

I've been doing my current project for two months, and I feel it is time for some notes.
<!--more-->

## Try out react in frontend
The original front end is django template + jQuery + bootstrap. Without any framework to manage the code structure, there were nearly no reusable UI components. We had to implement every page from scratch. More importantly, without certain patterns, the implementations of front end logic differed greatly from one another even for similar logics.

The project is a business management site, the pages are mainly the CRUD of different business objects, eg. a list for all the objects, a modal dialog for detail of an object, a modal for update or create object, a modal for delete confirm. It's clear that UI elements can be reused somehow. For example, the boostrap modal:
```html
<div class="modal fade">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <button type="button" class="close" data-dismiss="modal" aria-label="Close"><span aria-hidden="true">&times;</span></button>
        <h4 class="modal-title">Modal title</h4>
      </div>
      <div class="modal-body">
        <p>One fine body&hellip;</p>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-default" data-dismiss="modal">Close</button>
        <button type="button" class="btn btn-primary">Save changes</button>
      </div>
    </div><!-- /.modal-content -->
  </div><!-- /.modal-dialog -->
</div><!-- /.modal -->
```
This structure is common among all objects, the difference is the title, the content and footer. The old way will be copy this structure, then write the contents in title, content and footer. Consider after copied this structure for near a hundred times, then one day we need to add a new css class in every modal... What a desperate work.. How about we write only one copy of this structure and generate the contents dynamicaly. With react, this is a very trival work.

Another problem is that, as UI elements get more and more complex, the relation between elements will be hard to manage. For example, when radio button A is checked, textinput B should appear, button C should hide, button D should appear... And when button A lost its check, we should perform the reverse actions. This will get tedious and hard to understand.

#### data binding
To solve this problem, I think first we need to use data to drive the UI, instead of considering the relationship between UI elements. For the example above, bind radio A to a variable, then make related UI listen for state of variable and make change themselves. If one day, we deside to add new UI when A is checked, just make this UI listen for vairable, saving us the trouble of modifying the listeners of A. This way we removed the complex relations between UIs, as all the UIs are now bind to data. For our project, we use state of react components to hold the data, and update the underlying UI accordingly. We won't care which children will get changed when state changes, the children is responsible for themselves.

#### components
Another thing I learned is that, when updatin UI, prefer function call to update DOM directly. For example, when click 'Detail' button, we load some data and show a modal with the data displayed. A naive way is to use lots of ```$(xxx).value(yyy)``` or ```$(xxx).find(xxx).prop('checked', true)``` to update DOM. As we can see, this is not easy to understand at application level. A prefer way is, we create a class that represents the modal for detail. Then, we pass this class the data, and say:"Hey, update your self". Although under the hood of this class, it does the same ugly things as the first approach, the application itself gets a lot cleaner, and the change of UI in modal won't break the application. With react, this get done natrually by passing properties to child component.

After used react, our modal looks like this.
```html
React.createClass({
  render: function() {
    return (
      <div class="modal fade">
	<div class="modal-dialog">
	  <div class="modal-content">
	    <div class="modal-header">
              <button type="button" class="close" data-dismiss="modal" aria-label="Close"><span aria-hidden="true">&times;</span></button>
              <h4 class="modal-title">{this.props.title}</h4>
	    </div>
	    <div class="modal-body">
	      <ChildContent data={this.state.data} />
	    </div>
	    <div class="modal-footer">
              <button type="button" class="btn btn-default" data-dismiss="modal">Close</button>
              <button type="button" class="btn btn-primary" onClick={this.handleSubmit}>Save changes</button>
	    </div>
	  </div><!-- /.modal-content -->
	</div><!-- /.modal-dialog -->
      </div><!-- /.modal -->
    )
  }
});
```
We just pass data to child, and child will make the view done according to specific needs.

### some cavets when using react

#### How to coexist with bootstrap-modal
Our project use bootstrap-modal, which comes with a ModalManager, so that modal can be stackable. But when doing this, it moves the modals away from original position and attach them to document. If we use react to generate modal, when it is shown, the react will fail with `Invariant Violation: findComponentRoot(..., .0.1.0): Unable to find element. This probably means the DOM was unexpectedly mutated.` It turns out that, outer program should not manipulate the DOM of react component directly. After some search, people say that, it doesn't matter if parents of react component is moved around. So instead of generating the whold modal with react, we first write ```<div class="modal"></div>``` part with plain html, then we use react to generate ```<div class="modal-dialog"></div>```part, and mount it into modal. With this approach, the error of react gone away.

#### key and generated list
In react [document](https://facebook.github.io/react/docs/multiple-components.html#dynamic-children), it is said that `key` is needed to determine the child should be reused or destoryed. I didn't quite understand it until I ran into some wierd problem. My UI is a list of input representing a list of data eg. [a, b,c ,d]. We can delete a row, or add a new row.
The werid part is that, when I delete the row with value c, the resulting list in UI will be [a, b, c] which is wrong, but the data of the state is correct with data [a, b, d]. That is always the last UI element gets deleted. When I look at the row, each row is with something like `data-reactid=".0.0.1.0.0.2.0"`,  the last digit in the id is key(maybe..). It seems that if we don't allocate key, react will simple allocate 1, 2, 3, 4  as keys. When we delete c, the state changes so the UI repaints, again key 1, 2, 3 is allocated for remaining elements. React compares key, and think that element with 1, 2, 3 is unchanged and they are reused, only element with key 4 is removed. As the result, c remained while d went away. So I tried to allocate brand new keys for each state change, eg. [1, 2, 3, 4], then [5, 6, 7, 8]. So that there were no duplicate keys between state change. Then the id will looks like `data-reactid=".0.0.1.0.0.2.$1`, where $1 seems the key 1. And this solves the problem. But actually, it is a big waste as all the UI elements got repainted event they didn't change.

### TODO: automate babel with webpack
