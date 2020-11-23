YouTube link
https://www.youtube.com/watch?v=CD_t3m2WMM8

I was reading an article by Ward Bell in which he argued for the superiority of templated driven forms to Reactive ones in Angular.

I have to admit that the first time I watched this video I was somewhat confused and I've got a lot of experience with working with Angular and Angular forms..

Having watched it several time and investigated what she is talking about, I've reached the conclusion that this is one of these instances where the reason somethings sounds complicated is because it *is* complicated! I feel a similar feeling as I remember having when first reading the documentation for AngularJS's directives. In those days, they had a comments section at the bottom of the page and it was full of similarly bewildered people venting their frustration! (I don't know if that's why they got rid of the comments.)o

1. Angular forms primer
2. updateOn config property 

## 3. Custom Form Controls
[timestamp](https://youtu.be/CD_t3m2WMM8?t=550)
> A custom form control is a directive that knows how to integrate with Angular's Form API

Mostly it seems more sensible for a CFC to be a component and not just a directive.

### What is the benefit of a Custom Form control? 

In general, the reasons why you would create a CFC are the same as why you would create a Component:

* to break up a template into smaller pieces 
* to enable encapsulation
* to faciliate code reuse.

Kara gives some specific examples of CFCs

> A custom form control can be used inside a form just like any native element

CFCs can be used in both Reactive Forms and Template Driven Forms.
 
Angular's docs do not explain how to create CFCs.

The best explanation of how this works is probably by Jennifer Wadella.
https://jenniferwadella.com/blog/understanding-angulars-control-value-accessor-interface

This is fairly straightforward although the full code is not supplied so there are a few gaps that require to be filled in order to get this working. My implementation consists of a custom form control with a single input field with validation and an error message. I show it being used both within a template driven form and in a reactive form.

To build a CFC, you need to implement the `ControlValueAccessor` interface.
This has 3 required methods and 1 optional one

## Examples

[my example](https://stackblitz.com/edit/angular-control-value-accessor-template-driven-example?file=src%2Fapp%2Ffoo%2Ffoo.component.css)
demonstrates the CFC being used both within a template driven and a reactive form

[ControlValueAccessor implementation using template driven form](https://stackblitz.com/edit/angular-custom-form-control-1)
```

// write value into dom
//  model -> view
writeValue(value: any) {}


// callback for when there is a change in the DOM
//  view -> model
registerOnChange(fn: (value: any) => void) {}

// callback for touched event
//  so the forms API can properly toggle the touched property (usually on blur)
//  view -> model
registerOnTouched(fn: () => void) {}

//  optional method disable element in view
// so the form control can be enabled and disabled programmatically
//  model -> view
setDisabledState(isDisabled: boolean) {}


```

Within the CFC, we obtain a reference to the input field itself using a `@ViewChild` query.

```
  @ViewChild("thisInput") input: ElementRef;
```

We use this reference to set the value property of the input element, using the value that is passed to the writeValue method.
```
  writeValue(val: any) {
    if(this.input) {
      this.input.nativeElement.value = val;
    }
  }
```

An extra check is needed that `this.input` exists before attempting to use it. This check is not needed when using the CFC in a TD form, but is when used in a Reactive form.
// I should investigate this


#### registerOnChange()

`registerOnChange()` is called by the forms API to pass to our code a callback which we will call whenever there is some change within our component.

In Kara's example, the callback is saved as a property of the component and then called within the template statement that is assigned to the input event of our input element

```
// in component

registerOnChange(fn: (value: any) => void) {
  this.onChange = fn;
}

//  in template

<input type="text" (input)="onChange($event.target.value)"/>


```
Stackblitz seems to see $event as of type `InputEvent` with value:
```
{isTrusted: true}
```

When I tried this using Stackblitz, I got an error complaining `Property 'value' does not exist on type 'EventTarget'.`
There seems to be some confusion about what type `$event` is and there are plenty of StackOverflow questions concerning problems related to this.

This issue in the Angular Github discusses the problem https://github.com/angular/angular/issues/35293

The Angular docs has this to say about this config option
> strictDomEventTypes	: Whether $event will have the correct type for event bindings to DOM events. If disabled, it will be any.

Setting the strictDomEventTypes option to false fixes the problem. However, it doesn't appear to be possible to do this in Stackblitz. I have raised an issue addressing this problem.https://github.com/stackblitz/core/issues/1334

casting it to type of any within the template also fixes the problem.

```
<input (input)="onChange($any($event).target.value)"/>


```


We could pass the $event object directly as an argument and have the component deal with casting, but Angular docs regards this as bad practice and recommends using a template variable instead.
https://angular.io/guide/user-input#passing-event-is-a-dubious-practice

```
<input
  #thisInput
  (input)="onChange(thisInput.value)"
/>
```

### registerOnTouched()
callback for touched event
so the forms API can properly toggle the touched property (usually on blur)
```
  //  in template

  <input (blur)="onTouched()" />

  // in component

  registerOnTouched(fn: () => void) {
    this.onTouched = fn;
  }
```
I didn't encounter any problems implementing this.

### setDisabledState()
model -> view
So the forms API can programmatically disable the component.
```
  //  in template
  <input [disabled]="disabled" />

  // in component

  setDisabledState(isDisabled: boolean) {
    this.disabled = isDisabled;
  }
```

Having implemented the ControlValueAccessor interface, there is another step to make to make a CFC work and that is to register this component in the local injector. As often with Angular, there are two ways of doing this (something I call "Angular's rule of 2") and Kara describes both of them.

The first is to register this component class a provider for the NG_VALUE_ACESSOR token in the local element injector.

``` 
providers: [

  {
    provide: NG_VALUE_ACCESSOR,
    multi: true,
    useExisting: FooComponent
  }
]

```
The NG_VALUE_ACCESSOR is a token that provides classes that implement ControlValueAccessor to forms.

## Nested Forms
The third thing she talks about.
This is the part that Ward Bell was confused by, and I don't think that's unreasonable as I found it hard to follow as well.

What is meant by *nested forms* are components which represent a part of a form, perhaps a set of input components that are grouped conceptually in some ways, e.g. an address.

The first solution given is a nested form component created as a Custom Form Control as was previously described, but possibly containing several input fields rather than just one. This is called a *Composite ControlValueAccessor*. The obvious benefit of this is that the component becomes much easier to use and reuse within the Angular forms framework, and can be used as both part of a template driven form and a reactive form.

This seems to me to be the best of all the solutions given for nested form components and also is the most straightforwards, particularly if you've mastered the single field ControlValueAccessor component. For those who really got lost in the subsequent explanations, I'd recommend just ignoring them and sticking with this one.


### Validation and Error Messages
She doesn't talk about how to implement validation and error messages for the Composite ControlValueAccessor component, however, so I'm going to discuss how to go about doing this.


Because you have access to the formControl elements within the address component, you can query their status and show or hide error messages accordingly. The problem is that this component can be invalid whilst the form as a whole is valid. We would probably want the entire form to be invalid if any component part of it was invalid, regardless of how many levels nested it was. So how do we do that?


## Form Projection
[talk here](https://youtu.be/CD_t3m2WMM8?t=2213)

What is *form projection*? 

What is an *error aggregator*?

"Some kind of form that wraps this error aggregation logic"
