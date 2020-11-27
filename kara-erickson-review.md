# Commentary on Kara Erickson's talk on Angular Forms
This article is a commentary on a talk that was given by Angular Core developer Kara Erickson at Angular Connect in 2017.
Link [here](https://www.youtube.com/watch?v=CD_t3m2WMM8)

This talk concerned some advanced topics in relation to Angular forms, such as Custom Form Controls, nested forms, and form projection, important subjects which, by and large, aren't covered particularly well in the Angular docs themselves. Although the talk was given a few years ago, it is still relevant as the API has not changed substantially since then.

Whilst the information is good, unfortunately the code shown in the demos was never released, as far as I can ascertain. In this article I will discuss my efforts to recreate the code and also clarify a few things from the talk that I initially found confusing.

The first nine minutes of the talk comprises a refresher on Angular forms and an introduction to the (then) new `updateOn` option. Nothing here is particularly difficult.

## Custom Form Controls
[9:02](https://youtu.be/CD_t3m2WMM8?t=542)

A Custom Form Control (CFC) is a directive that implements the `ControlValueAccessor` interface. This results in the directive being able to integrate with Angular's Form API. It can be used in Angular just as any native input element can be and works with both Reactive and Template Driven forms

### Why would you want to use them?
The reason you would create a Custom Form Control is the same as for components in general:
* to break up a template into smaller pieces 
* to enable encapsulation
* to faciliate code reuse.

Some specific examples are:
* Non-native form control elements
* Custom styling / functionality
* Control wrapped with related elements
* Parser / formatter directive

### Implementing the ControlValueAccessor interface
The `ControlValueAccessor` interface looks like this, comprising 3 required methods and 1 optional one:
```
 writeValue(value: any) {}
 registerOnChange(fn: (value: any) => void) {}
 registerOnTouched(fn: () => void) {}
 setDisabledState(isDisabled: boolean) {}
```
I'm going to go through some problems I had implementing the first two of these methods.
The two remaining methods, `registerOnTouched()` and `setDisabledState()` were straightforward to implement and didn't present me with problems.

### writeValue()
The `writeValue(`) method is called by the forms API to set values into our component within the DOM.
In order to do this we need a reference to our input field. We can retrieve this from our template using a `@ViewChild` query.
```
  @ViewChild("input") input: ElementRef;
```
We can use this ref to set the value of the input element.
```
  writeValue(val: any) {
    if (this.input) {
      this.input.nativeElement.value = val;
    }
  }
```
This differs from Kara's example in that an extra check is needed that `this.input` exists before attempting to use it. This check is needed when the CFC is used in a Reactive form, but not when used in a Template Driven form. Apparently, the Angular Forms API calls writeValue() before the view has been queried for the input field.

### registerOnChange()
[14:45](https://youtu.be/CD_t3m2WMM8?t=885)

The `registerOnChange()` method is called by the forms API to pass a callback to our code which we then must call whenever there is some change within our component.
```
  // in component

  registerOnChange(fn: (value: any) => void) {
    this.onChange = fn;
  }

  //  in template

  <input type="text" (input)="onChange($event.target.value)"/>
```
Depending on the environment, this can cause the following error:
```
Property 'value' does not exist on type 'EventTarget'.
```
The problem is the kind of type-checking that is being done within the template. 
You configure this with the `strictDomEventTypes` compiler property.
> Whether $event will have the correct type for event bindings to DOM events. If disabled, it will be `any`.

One solution, therefore, is to set this property to `false`.

As the excerpt from the documentation above suggests, another solution is to cast `$event` to type `any`:
```
<input (input)="onChange($any($event).target.value)"/>
```
You could also pass `$event` as the argument to the `onChange()` method and let the component handle casting, 
but this is regarded as [bad practise](https://angular.io/guide/user-input#passing-event-is-a-dubious-practice).

The solution I settled on was to use a template variable:
```
<input #thisInput (input)="onChange(thisInput.value)" />
```
The `thisInput` template variable refers to the `<input/>` element and is given the correct type by Angular.

registerOnTouched()

setDisabledState()

### Registering with the local injector
[15:41](https://youtu.be/CD_t3m2WMM8?t=941)

Having implemented a ControlValueAccessor, we need to let Angular's Form API know about it. We do this by registering it with the local injector using the NG_VALUE_ACCESSOR token.
``` 
// in RequiredText
providers: [
  {
    provide: NG_VALUE_ACCESSOR,
    multi: true,
    useExisting: RequiredText
  }
]
``` 
The `NG_VALUE_ACCESSOR` is a Dependency Injection token representing classes that implement the ControlValueAccessor interface.
The `multi` property means that we can register multiple providers with the token.
`useExisting` means that we use the existing instance of MyComponent that the injector has already created. Obviously, RequiredText is already in the injector so we are aliasing it here.

### Validation
[16:12](https://youtu.be/CD_t3m2WMM8?t=972)

Validation is achieved by implementing the Validator interface and registering the component using NG_VALIDATORS with the local injector. 
// show code excerpts
// link to stackblitz 

### Error Messages
[19:10](https://youtu.be/CD_t3m2WMM8?t=1150)

How do we show error messages from within the component itself?
The problem is that we need to know the validation status of the form control within the component itself, but currently we do not have a reference to this form control. So how do we get this reference?
One approach is to provide it as an input to our component:
```
<required-text formControlName="three" [control]="form.get('three')"></required-text>
```
Clearly, though, it would be nicer to not have to do this. It's extra code that we have to write and extra "noise" when looking at the template.

Better is to use Dependency Injection and inject the form control into our component through its constructor function.

```
constructor(@Self() public controlDir: NgControl) {}
```
This works because when we add a form directive to an element the underlying object is registered with the local injector.

We specify `NgControl` as the provider as it is the super-type of all form directives that we might have set on our form element, such as `NgModel` and `FormControlName`, so we can use our component with all of these.

The @Self decorator is necessary so that we don't look beyond the local element injector for the NgControl instance.

However, now that we are injecting NgControl, we can have a circular dependency because NgControl, (or rather the instances of it, e.g. NgModel), is injecting both `NG_VALUE_ACCESSOR` and `NG_VALIDATOR`, therefore we need to not provide these in our component. This means we have to manually "wire" our component up with the Angular Forms API and add validators to it.

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
We can also remove the `validate()` method if we've got it as we're not longer doing validation that way.
The above code could actually go in the constructor, but it's considered best practice to keep as much logic out of the constructor as possible.

Now that we have a reference to the formControl, we can now set up the error messages within our component. There's nothing too complicated about this:
We just query the form control for properties such as `valid` and `pristine` and render an error message accordingly.
```
  <div *ngIf="controlDir && !controlDir.control.pristine &&!controlDir.control.valid" class="error">There was an error</div>
```

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
But this doesn't work. We don't get an error message but our form template variable doesn't get populated either. It seems that the ngModel on our input field isn't communicating with the form directive. This is somewhat similar to the problem we face with sub form components. The solution is that we need to find a way for our ngModelGroup to get a reference to the ngForm directive. 
The difficulty is that ngForm lives in the view child whilst ngModelGroup lives in the content child and these can't directly communicate with one another.
However, directives in The content child can inject providers that are set in the host component, and the host component can access directives in the view children, so it would seem that within the host component we should be able to extract the form from the view and then make this available to ngModel in the content child.
 My investigations suggest that it might not even be possible.
The reason is that we make the ngForm available in the injector using the useFactory() function. But this will run when the constructor for our ngModel directive runs. Because this is in the content children, it runs before the view is actually initialised. Thus, when useFactory runs, it uses a form that doesn't yet exist.

Kara seems to anticipate this problem, but I haven't been able to make her solution work.

I have posted a question on Stackoverflow to see if anyone else has an answer to this problem, but for the moment I do not know how to solve it.

## Resources
Besides this talk that we are discussing, the best resource online that I have found regarding the `ControlValueAccessor` interface is [this blog](https://jenniferwadella.com/blog/understanding-angulars-control-value-accessor-interface) by Jennifer Wadella. She also does some talks on the same subject that can be found on YouTube.

### Examples

[my example](https://stackblitz.com/edit/angular-control-value-accessor-template-driven-example?file=src%2Fapp%2Ffoo%2Ffoo.component.css)
demonstrates the CFC being used both within a template driven and a reactive form

* [ControlValueAccessor implementation using template driven form](https://stackblitz.com/edit/angular-custom-form-control-1)
* [ControlValueAccessor implementation using reactive forms]()


