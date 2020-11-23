# Commentary on Kara Erickson's talk on Angular Forms
This article is a commentary on the talk that was given by Angular core team developer Kara Erickson.
The talk concerns some advanced topics in relation to Angular forms.
Although it was given in 2017, the API has not changed substantially since then and so is still relevant.
These topics are not very well covered in Angular docs so it's very valuable to have a core member of the Angular developers talk about them.
The biggest shortcoming of the talk is that no code for it was (as far as I am aware) released for the examples given and that code that is on the slides is insufficent for a working implementation. This article is therefore mainly concerned with my endeavors to implement the things she talked about.
[link to talk](https://www.youtube.com/watch?v=CD_t3m2WMM8)

## Custom Form Controls (CCF)
After a few preliminaries and discussing the new `updateOn` config option, [Kara shows how to implement a Custom Form Control](https://youtu.be/CD_t3m2WMM8?t=550)

The important things to understand about a CCF are:
* It is a directive that knows how to integrate with Angular's Form API
* It can be used inside a form just like any native element
* They are built by implementing the `ControlValueAccessor` interface
* A CFC can be used in both Reactive Forms and Template Driven Forms.

### Why would we want to use them?
Kara gives some examples of CCF. These are:
* Non-native form control elements
* Custom styling / functionality
* Control wrapped with related elements
* Parser /formatter directive *

These generalise into the following reasons for creating a CCF
* to break up a template into smaller pieces 
* to enable encapsulation
* to faciliate code reuse.

Unfortunately, Angular's docs do not give much information about implementing the ControlValueAccessorInterface. Besides this talk that we are discussing, the best resource online that I have found is [this blog](https://jenniferwadella.com/blog/understanding-angulars-control-value-accessor-interface) by Jennifer Wadella. She also does some talks on the same subject that can be found on YouTube.


The `ControlValueAccessor` interface looks like this, comprising 3 required methods and 1 optional one:
```
 writeValue(value: any) {}
 registerOnChange(fn: (value: any) => void) {}
 registerOnTouched(fn: () => void) {}
 setDisabledState(isDisabled: boolean) {}

```
### Implementing the methods
#### writeValue()
This method allows the Forms API to set values into our component within the DOM.
In order to do this we need a reference to our input field. We get this out using a `@ViewChild` query.

```
  @ViewChild("thisInput") input: ElementRef;
```
Then we use this ref to set the value of the input element.
```
  writeValue(val: any) {
    if (this.input) {
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
When I tried this using Stackblitz, I got an error complaining `Property 'value' does not exist on type 'EventTarget'.`
After some investigation, it turned out that the problem was the kind of type checking that is being done within the template.
This issue is discussed on the Angular Gitub [here](https://github.com/angular/angular/issues/35293)
Whether or not the error occurs depends on the setting of the `strictDomEventTypes` compiler option.
The Angular docs has this to say about it:
> Whether $event will have the correct type for event bindings to DOM events. If disabled, it will be any.

Setting the `strictDomEventTypes` option to *false* fixes the problem. However, it doesn't appear to be possible to do this in Stackblitz. I have raised an issue addressing this problem: https://github.com/stackblitz/core/issues/1334

Casting `$event` to type of `any` within the template also fixes the problem:

```
<input (input)="onChange($any($event).target.value)"/>
```

We could also pass the $event object directly as an argument and have the component deal with casting, but Angular docs regards this as bad practice and [recommends using a template variable instead](https://angular.io/guide/user-input#passing-event-is-a-dubious-practice), and that was the solution that I settled on:

```
<input #thisInput (input)="onChange(thisInput.value)" />
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


### Examples

[my example](https://stackblitz.com/edit/angular-control-value-accessor-template-driven-example?file=src%2Fapp%2Ffoo%2Ffoo.component.css)
demonstrates the CFC being used both within a template driven and a reactive form

* [ControlValueAccessor implementation using template driven form](https://stackblitz.com/edit/angular-custom-form-control-1)
* [ControlValueAccessor implementation using reactive forms]()

ref
I don't know exactly what a parser/formatter directive is, and Kara does not go into greater detail. (I guess it's a directive that formats the input as its being entered?)
