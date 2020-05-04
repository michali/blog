---
layout: post
title: "Trigger validation of a dropdown box when tabbing over it in KnockoutJS"
date: 2020-05-05 08:00
tags:
- KnockoutJS
excerpt: When tabbing over a dropdown box in a form, it's tougher to trigger its validations than it is with empty text boxes. Here's a trick on how to enable that.
---

I had a problem to solve recently which was to change a form's validation triggers in a front-end that uses KnockoutJS. When a user tabs through the required input elements of a form, the validation visuals of said elements get highlighted. That was not the case, however, with the dropdown boxes in the form as those didn't highlight as invalid. The would still show the caption entry on the top ("Please select from the below") which obviously is a prompt and that doesn't meet the "required" criteria, but the validation would only trigger if a value was selected and then the caption entry on the top was selected again, which isn't a usual user journey to start with.

The invalid dropdown entries would still be caught upon form submission. However, a quick feedback would provide consistency in the UI instead of some fields being highlighted when passed on with the tab key and some others not being hightlighted. More importantly, it's also an accessibility issue as screen readers will wait until the end instead of picking up the invalid field as soon as they do with the text boxes.

Let's look at an example. Here is a dropdown box whose markup and validation markers look like this:

```html
<div data-bind="css: { 'field-error': stateValid }">
    <label id="lblState" for="selectState" aria-required="true">
        State/Territory
        <abbr class="required-indicator" title="Required"> *</abbr>
    </label>
    <div>
        <div>
            <select aria-invalid="false" name="expiry-month" id="selectState" 
            aria-describedby="selectStateErr" 
            data-bind="event: {blur: forceValidation.bind($data, state) }, value: state">
            <option value="" selected>-Select-</option>
                <option value="ACT">ACT</option>
                <option value="NSW">NSW</option>
                <option value="QLD">QLD</option>
                <option value="VIC">VIC</option>
                <option value="TAS">TAS</option>
                <option value="NT">NT</option>
                <option value="WA">WA</option>
                <option value="SA">SA</option>
        </select>
        </div>
        <ul class="error" aria-live="polite" aria-atomic="true" id="selectStateErr">
            <li data-bind="if: stateValid">
                <span class="icon icon-error"></span>
                <span class="sr-only">State</span>
                <span data-bind="validationMessage: state"></span>
            </li>
        </ul>
    </div>
</div>
```

Validation rules for the dropdown:

```javascript
this.state = ko.observable("")
    .extend({
        required: {
            message: "You must select a state or territory",
        }
     });  
```

The view model property `state` is set up with an empty string value, *same as the placeholder entry's value*. Were this not the case, the "required" validation of the dropdown would kick in immediately. This is because while the observable is initialized with another value, when Knockout evaluates the value binding it sets the `state` field to the currently selected empty string value. Because undefined != "", validation fails as knockout thinks that the property has been modified.

Then create a property with an observable that will trigger validation visuals in the DOM:

```javascript
self.stateValid = ko.computed(function(){
    return validateField(self.state);
});
```

Where `validateField` is a generic validation function you would use with text fields too:

```javascript
function validateField(field){
    return !field.isValid() && field.isModified();
}
```

An additional step is required to trigger the validations of a dropdown when focusing and unfocusing from it.

The `blur` event is fired when a user clicks out or tabs out from an input element. We need to let knockout know that it's a event it should watch out for:

```javascript
this.valueUpdate = ['blur']
```

Edit the `selectState` opening dropdown tag as follows:

```html
 <select aria-invalid="false" name="expiry-month" id="selectState" 
            aria-describedby="selectStateErr" 
            data-bind="event:{blur: forceValidation.bind($data, state) }, value: state">

```
And add the following function:
```javascript
self.forceValidation = function(field){
    field.valueHasMutated(); 
};    
```

Now when the dropdown loses focus its bound observable will have its `valueHasMutated` function fired. `valueHasMutated` normally notifies the observable's subscribers that the value has changed but manually invoking it will have the same effect.

With that in place, you have a dropdown whose validation is also triggered by it losing focus.