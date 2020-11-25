# Commentary on Kara Erickson's talk on Angular Forms
This article is a commentary on a talk that was given by Angular Framework Tech Lead Kara Erickson at Angular Connect in 2017.

That talk concerned some advanced topics in relation to Angular forms which, by and large, aren't covered particularly well in the Angular docs themselves, which makes it valuable. Although the talk was given a few years ago, it is still relevant as the API has not changed substantially since then.

The biggest shortcoming of the talk is that whilst it featured a number of code demos, the actual code examples were never released (see footnote).
This article will discuss my attempts to implement the demos from the talk and to clarify a few things which I myself found a little confusing the first time I listened through it.
The original talk can be found [here](https://www.youtube.com/watch?v=CD_t3m2WMM8).

(Kara promises to post the code on Twitter but whilst the slides were posted, the code does not appear to have been.)

## Custom Form Controls
After a few preliminaries, Kara demonstrates how to create a Custom Form Control (CCF).
[link to this item](https://youtu.be/CD_t3m2WMM8?t=550)

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
The `writeValue(`) method allows the Forms API to set values into our component within the DOM.
In order to do this we need a reference to our input field. We can retrieve this from our template using a `@ViewChild` query.

```
  @ViewChild("thisInput") input: ElementRef;
```
We can use this ref to set the value of the input element.
```
  writeValue(val: any) {
    if (this.input) {
      this.input.nativeElement.value = val;
    }
  }
```

An extra check is needed that `this.input` exists before attempting to use it. This check is not needed when using the CFC in a TD form, but is when used in a Reactive form.
// I should investigate this

`registerOnChange()` is called by the forms API to pass a callback to our code which we must call whenever there is some change within our component.

In Kara's example, the callback is saved as a property of the component and then called within the template statement that is assigned to the input event of our input element

```
// in component

registerOnChange(fn: (value: any) => void) {
  this.onChange = fn;
}

//  in template

<input type="text" (input)="onChange($event.target.value)"/>

```
When I tried this using Stackblitz, I got an error complaining that `Property 'value' does not exist on type 'EventTarget'.`
After some investigation, it turned out that the problem was the kind of type checking that is being done within the template.
This issue is discussed on [Angular's Gitub](https://github.com/angular/angular/issues/35293)
Whether or not the error occurs depends on the setting of the `strictDomEventTypes` compiler option.
The Angular docs has this to say about that:
> Whether $event will have the correct type for event bindings to DOM events. If disabled, it will be `any`.

Setting the `strictDomEventTypes` option to *false* fixes the problem. However, it doesn't appear to be possible to do this in Stackblitz. I have raised an [issue](https://github.com/stackblitz/core/issues/1334) addressing this problem.

Casting `$event` to type of `any` within the template also fixes the problem:

```
<input (input)="onChange($any($event).target.value)"/>
```

Another solution would be to pass the $event object directly as an argument and have the component deal with casting, but Angular docs regards this as bad practice and [recommends using a template variable instead](https://angular.io/guide/user-input#passing-event-is-a-dubious-practice).

Using a template variable was the solution that I settled on:

```
<input #thisInput (input)="onChange(thisInput.value)" />
```
The two remaining methods, `registerOnTouched()` and `setDisabledState()` were straightforward to implement and didn't present me with problems.

### Registering the ControlValueAccessor
Having implemented the ControlValueAccessor interface, it also has to be available to directives like `ngModel` and `formControlName` which will attempt to inject it. There are two ways to do this.

The first is to register our component class as a provider for the NG_VALUE_ACCESSOR token within the local element injector.

``` 
// in MyComponent
providers: [

  {
    provide: NG_VALUE_ACCESSOR,
    multi: true,
    useExisting: MyComponent
  }
]

```
The `NG_VALUE_ACCESSOR` is a Dependency Injection token representing classes that implement the ControlValueAccessor interface.
The `multi` property means that we can register multiple providers with the token.
`useExisting` means that we use the existing instance of MyComponent that the injector has already created.

This works for simple components, but there are circumstances in which a different approach is required.
Suppose we need a reference to formControl of this component. We might want to do this if we want to query its current status for the purpose of showing error messages. We don't have it directly within the component itself. We can either pass it in from the containing template, or else we can get it via dependency injection. The latter way is the one we will employ.
```
constructor(@Self() public controlDir: NgControl) {}

```
NgControl is the superclass of all form control directives, including NgModel and FormControlName. When we request an object for this class, we get whichever class in the injector is a subclass of this type. Depending on which directive has been inserted upon our component element, e.g. `ngModel` or `formControlName`, we will get an object of that corresponding class back.

So for example, if we configure our component like so:
```
<my-component ngModel name="firstName"></my-component>
```
Then `controlDir` in the above constructor will be an instance `NgModel`, although it's important to mention that we will only be able to use it as its declared type - `NgControl`, as this is how Typescript works.

The reason for the `@Self()` decorator is that we only want to look in the local element injector for an `NgControl`. If in the case where there has not been a directive applied to our component element, we don't want to reach out to an injector higher up in the injector hierarchy to look for an NgController. If there happens to be one, this could cause our application to work in unexpected and broken ways. The decorator is thus not essential, but its a useful safeguard.

Because NgControl also injects value accessors and validators, we can no longer use the providers array to add our component to the set of NG_VALUE_ACCESSORS within the local element injector. 

But recall that all that is required is that NgControl know about ControlValueAcessor. We can manually set this in the constructor.

```
constructor(@Self() public controlDir: NgControl) {
  controlDir.valueAccessor = controlDir;
}

```
Similarily, we can configure the validators within ngOnInit()
```
ngOnInit() {
 const control = this.controlDir.control;
 control.setValidators(Validators.required);
 control.updateValueAndValidity();
}
```
Kara does not explain why this code in put in the `ngOnInit()` rather than in the constructor. Putting it in the constructor also seems to work. I assume it's to do with putting as little logic as possible in the constructor, which Angular docs recommnend. (One reason for this is so that in unit tests you can separate out object creation from object behaviour).

## Nested Forms
[timestamp](https://youtu.be/CD_t3m2WMM8?t=1522)

A *nested form* is a component that comprises part of a form. A common example is of an address component that contains separate fields for street, city, postcode etc. The motivation for having a nested form is similar to that of a Custom Form Component - to group related behaviour, for ease of re-use etc.

There are two kinds of nested form components (NFCs) that Kara talks about.
* *Composite ControlValueAccessor Component*: A ControlValueAccessor component which contains an arbitrary number of form controls instead of just one.
* *Sub Form Component*: A Component that contains a form fragment but does not implement the ControlValueAccessor interface.

### Composite ControlValueAccessor Component (CCC)
[stackblitz example](https://stackblitz.com/edit/angular-composite-control-value-accessor)

Implementing a CCC is largely the same as implementing a CFC. Kara doesn't say anything about validation. Whilst validation and error messages can be self-contained within the component, it's important to implement the `validate()` method so that the component's valid status stays in sync with the containing form.

### Sub Form Component (SFC)
[stackblitz example](https://stackblitz.com/edit/angular-sub-form-component)

The main difference between this an a CCC is that this does not implement the `ControlValueAccessor` interface. Unfortunately, this doesn't make it less complicated to implement. Arguably, it's more complicated. In Ward Bell's article about Reactive Forms in which he mentions this talk, he complains about being confused by this section.

What I want to do here is explain the issues involved:

Supposing we have a form which contains a firstName field, and a group of fields that comprise the address. We might organise this as follows:
```
<form #form="ngForm">

  <label for="name">name</label>
  <input name="firstName" ngModel id="name" required />

  <div ngModelGroup="address">
    <input ngModel name="city"/>
    <input ngModel name="postcode"/>
  </div>

</form>
```
The address fields are grouped using the `ngModelGroup` directive.
Now supposing we want the address section to be its own component. We might create an Address component with a template that looks like this:

```
   <div ngModelGroup="address">
    <input ngModel name="city"/>
    <input ngModel name="postcode"/>
  </div>

```
As Kara explains, this results in a `Error: NodeInjector: NOT_FOUND [ControlContainer]` error.
In this case, the ControlContainer is the container of our subform, which should be the ngForm directive, but our local injector is not able to find it.
When our ngModelGroup was in the same template as the form element, it could find it, but now that it's in it's own component, it can't.

So why can't our local injector find the ngForm directive? The reason is that ngModelGroup is set up so that when it looks for its ControlContainer, (in this case `ngForm`), it will not look further than the host injector. This is the injector that is configured using the viewProviders array of our sub component.

```
 viewProviders: [
    { provide: ControlContainer, useExisting: NgForm}
  ],
```
Now, a problem here is what happens if the containing directive isn't ngForm?

Imagine we change our form so that the address component has as its parent an ngModelGroup instead of an ngModelForm:
```
<form #form="ngForm">

  <label for="name">name</label>
  <input name="firstName" ngModel id="name" required />
  
  <div ngModelGroup="home">
    <input name="telephone" ngModel/>
    <my-address></my-address>
  </div>

</form>
```
We will find that our form data actually now looks like this:
```
{
  "firstName": "",
  "home": {
    "telephone": ""
  },
  "address": {
    "city": "",
    "postcode": ""
  }
}
```
But why is address not within home?
The reason is that the address component still sees the ngForm as its parent because of how we configure its ControlContainer in viewProviders.
If we change viewProviders so that ControlContainer looks for an NgModelGroup
``` 
viewProviders: [
    { provide: ControlContainer, useExisting: NgModelGroup}
  ],

```
Then the form data looks like what we would like it to be:
```{
  "firstName": "",
  "home": {
    "telephone": "",
    "address": {
      "city": "",
      "postcode": ""
    }
  }
}

```
So this is one of the constraints of SFCs. You have to be careful about which components you nest them within and things can behave unexpectedly if you are not.

It should be pointed out that there's a mistake in the slide shown at around 32:50 where the component is called 'RequiredText' instead of 'AddressComponent' as in the example. The released slides correct this error, but this could easily confuse people, as it did me.

#### Sub Form Components with Reactive Forms

Once we create this template within `AddressComponent`:
```
   <div formGroupName="address">
     <input formControlName="street"/>
     <input formControlName="city"/>
   </div>
```

We get the following error 
```
Error: Cannot read property 'getFormGroup' of null
```
As before, we register the ControlContainer in the viewProviders array, this time assigning it to be a FormGroupDirective instead of NgModelGroup:
```
  viewProviders: [
   { provide: ControlContainer, useExisting: FormGroupDirective}
  ],
  ```
  
Now we get a different error:
```
ERROR
Error: Cannot find control with name: 'address'
```
This is because the `address` FormGroup does not exist. We need to create it within AddressComponent. 

```
  parent: FormGroupDirective;

  constructor(parent: FormGroupDirective) {
    //this.form = parent.form;
    this.parent = parent;
  }

  ngOnInit() {
    this.parent.form.addControl('address', new FormGroup({
      street: new FormControl(),
      city: new FormControl(),
    }))
  }

```
Note that there's a mistake in the slides. `this.form` within the constructor wont work because parent.form is null at this point.
Discussion [here](https://github.com/angular/angular/issues/23914)

## Form Projection
[talk here](https://youtu.be/CD_t3m2WMM8?t=2213)

Form projection is where you project content into a form element. So if you have a component `WrapperComponent` with a template:
```
<form #form="ngForm">
 <ng-content>
</form>

<pre>
  {{ form.value | json }}
</pre>
```
and you use it in another template as follows:
```
<my-address>
  <input name="firstName" ngModel />
</my-address>
```
We would say that the 'first-name' input element is being projected into the form element that lives within the WrapperComponent.

In the DOM it should look something like this:
```
 <form>
  <input name="firstName"/>
 </form>

```
But this doesn't work. We don't get an error message but our form template variable doesn't get populated either. Basically, it seems that the ngModel on our input field isn't communicating with the form directive.



Kara suggests that you shouldn't be doing this anyway, but it's interesting to try and make it work.

The question is: how do we provide the form that lives in the component view children to the ngModel that lives in the content children?

See an example [here](https://stackblitz.com/edit/angular-form-projection-1)
I've tried my best to get this working but I've been unsuccessful. I get the following error.
```
ERROR
Error:
ngModelGroup cannot be used with a parent formGroup directive.

Option 1: Use formGroupName instead of ngModelGroup (reactive strategy):


<div [formGroup]="myGroup">
<div formGroupName="person">
<input formControlName="firstName">
</div>
</div>

In your class:

this.myGroup = new FormGroup({
person: new FormGroup({ firstName: new FormControl() })
});

Option 2: Use a regular form tag instead of the formGroup directive (template-driven strategy):


<form>
<div ngModelGroup="person">
<input [(ngModel)]="person.name" name="firstName">
</div>
</form>
```
### Examples

[my example](https://stackblitz.com/edit/angular-control-value-accessor-template-driven-example?file=src%2Fapp%2Ffoo%2Ffoo.component.css)
demonstrates the CFC being used both within a template driven and a reactive form

* [ControlValueAccessor implementation using template driven form](https://stackblitz.com/edit/angular-custom-form-control-1)
* [ControlValueAccessor implementation using reactive forms]()

ref
I don't know exactly what a parser/formatter directive is, and Kara does not go into greater detail. (I guess it's a directive that formats the input as its being entered?)
