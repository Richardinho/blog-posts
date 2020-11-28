# Commentary on Kara Erickson's talk on Angular Forms
This article is a commentary on a [talk](https://www.youtube.com/watch?v=CD_t3m2WMM8) that was given by Angular Core developer Kara Erickson at Angular Connect in 2017.

This talk concerned some advanced topics in relation to Angular forms, such as Custom Form Controls, nested forms, and form projection, important subjects which, by and large, aren't covered particularly well in the Angular docs themselves. Although the talk was given a few years ago, it is still relevant as the API has not changed substantially since then.

Whilst the information is good, unfortunately the code shown in the demos was never released, as far as I can ascertain. In this article I will discuss my efforts to recreate the code and also clarify a few things from the talk that I initially found confusing.

The prerequisite for reading this article is that you have watched this video (and presumably know something about Angular).

The first nine minutes of the talk comprises a refresher on Angular forms and an introduction to the (then) new `updateOn` option. It's fairly uncontroversial so I'm not going to discuss that.

## Custom Form Controls
[9:02](https://youtu.be/CD_t3m2WMM8?t=542)

A Custom Form Control is a directive that implements the `ControlValueAccessor` interface. This results in the directive being able to integrate with Angular's Form API. It can be used in an Angular form just as any native input element can be. It also works interchangeably with both Reactive and Template Driven forms. So once you have created a Custom Form Control, it can be used with both of these form modules. 

### Why would you want to use them?
[9:15](https://youtu.be/CD_t3m2WMM8?t=555)

The reason you would create a Custom Form Control is the same as for components in general:
* to break up a template into smaller pieces 
* to enable encapsulation
* to facilitate code reuse.

Some specific examples are:
* Non-native form control elements
* Custom styling / functionality
* Control wrapped with related elements
* Parser / formatter directive

### Implementing the ControlValueAccessor interface
[12:25](https://youtu.be/CD_t3m2WMM8?t=745)

The `ControlValueAccessor` interface looks like this, comprising 3 required methods and 1 optional one:

```
  writeValue(value: any) {}
  registerOnChange(fn: (value: any) => void) {}
  registerOnTouched(fn: () => void) {}
  setDisabledState(isDisabled: boolean) {}
```
I'm going to go through some problems I had implementing the first two of these methods.
The two remaining methods, `registerOnTouched()` and `setDisabledState()` were straightforward to implement and didn't present me with any problems.

### writeValue()
[14:30](https://youtu.be/CD_t3m2WMM8?t=870)

The `writeValue()` method is called by the Forms API to set values into our component within the DOM.
In order to do this we need a reference to our input field. We can retrieve this from our template using a `@ViewChild` query.
```
  @ViewChild("input") input: ElementRef;
```
We can use this reference to set the value of the input element:
```
  writeValue(val: any) {
    if (this.input) {
      this.input.nativeElement.value = val;
    }
  }
```
This differs from Kara's example in that an extra check is needed to make sure that the `input` property exists before attempting to use it. This check is needed when the component is used in a Reactive form, but not when it's used in a Template Driven form. Apparently, the Angular Reactive Forms API calls `writeValue()` before  `input` is set.

### registerOnChange()
[14:45](https://youtu.be/CD_t3m2WMM8?t=885)

The `registerOnChange()` method is called by the Forms API to pass a callback to our code which we must then call whenever there is a change within our component.
```
  // in component

  registerOnChange(fn: (value: any) => void) {
    this.onChange = fn;
  }

  //  in template

  <input type="text" (input)="onChange($event.target.value)"/>
```
Depending on the environment, the above code can cause the following error:
```
Property 'value' does not exist on type 'EventTarget'.
```
This is a Typescript problem. Typescript for some reason can't determine the correct type of `$event`.
This is configurable using the `strictDOMEventTypes` option. The docs say this about it:
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

### Registering with the local injector
[15:41](https://youtu.be/CD_t3m2WMM8?t=941)

Having implemented a `ControlValueAccessor`, we need to let Angular's Form API know about it. We do this by registering it with the local injector using the `NG_VALUE_ACCESSOR` token.
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
`NG_VALUE_ACCESSOR` is a Dependency Injection token representing classes that implement the `ControlValueAccessor` interface.
The `multi` property means that we can register multiple providers with this token.
`useExisting` means that we use the existing instance of the `RequiredText` component that the injector has already created.

[code example](https://stackblitz.com/edit/angular-required-text-component)

### Validation
[16:12](https://youtu.be/CD_t3m2WMM8?t=972)

Validation is achieved by implementing the Validator interface and registering the component using the `NG_VALIDATORS` token in the local injector. 
```
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      multi: true,
      useExisting: RequiredTextComponent,
    },
    {
      provide: NG_VALIDATORS,
      multi: true,
      useExisting: RequiredTextComponent,
    }
  ],
```

[code example](https://stackblitz.com/edit/angular-required-text-component-with-validation)

### Error Messages
[19:10](https://youtu.be/CD_t3m2WMM8?t=1150)

In order to show error messages within the component itself, we need to get a reference to the component's form control. How do we get this?
One approach is to provide it as an input to our component.
This is how we'd do it in a reactive form:
```
<required-text formControlName="three" [control]="form.get('three')"></required-text>
```

Clearly, though, it would be nicer to not have to do this. It's extra code that we have to write and extra "noise" when reading through the code.

Better is to use Dependency Injection and inject the form control into our component through its constructor function.

```
constructor(@Self() public controlDir: NgControl) {}
```
This works because when we add a form directive to an element the underlying object is registered with the local injector.

We specify `NgControl` as the provider as it is the super-type of all form directives that we might have set on our form element, such as `NgModel` and `FormControlName`, so we can use our component with all of these.

The `@Self` decorator is necessary so that we don't look beyond the local element injector for the NgControl instance.

However, now that we are injecting `NgControl`, we can have a circular dependency because `NgControl`, (or rather the instances of it, e.g. `NgModel`), is injecting both `NG_VALUE_ACCESSOR` and `NG_VALIDATOR`, therefore we need to not provide these in our component. This means we have to manually wire our component up with the Angular Forms API and add validators to it.

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
We can also remove the `validate()` method if we've got it as we're no longer doing validation that way.
The above code could actually go in the constructor, but it's considered best practice to keep as much logic out of the constructor as possible.

Now that we have a reference to the formControl, we can now set up the error messages within our component. There's nothing too complicated about this:
We just query the form control for properties such as `valid` and `pristine` and render an error message accordingly.
```
  <div *ngIf="controlDir && !controlDir.control.pristine &&!controlDir.control.valid" class="error">There was an error</div>
```
[code example](https://stackblitz.com/edit/angular-custom-form-control-3)

## Nested Forms
[25:22 ](https://youtu.be/CD_t3m2WMM8?t=1522)

A *nested form* is a component that comprises part of a form. A common example is an address component that contains separate fields for street, city, postcode etc. The motivation for having a nested form is similar to that of a Custom Form Component - to group related behaviour, for ease of re-use etc.

There are two kinds of nested form components that Kara talks about.
* *Composite ControlValueAccessor Component*: A `ControlValueAccessor` component which contains an arbitrary number of form controls instead of just one.
* *Sub Form Component*: A Component that contains a form fragment but does not implement the `ControlValueAccessor` interface.

### Composite ControlValueAccessor Component (CCC)
[26:25](https://youtu.be/CD_t3m2WMM8?t=1585)

Implementing a Composite `ControlValueAccessor` is largely the same as implementing the kind that we've already seen. 

This seems to me to be the best way to do nested forms: It gives the greatest amount of flexibility and reusability. The component works in the same way as a native input element, making it easier to compose complex forms out of them.

Whilst validation and error messages can be self-contained within the component, and we don't have to pass in the form control, it's important to implement the `validate()` method so that the component's valid status stays in sync with the containing form.

[code example](https://stackblitz.com/edit/angular-composite-control-value-accessor)

### Sub Form Component
[30:19](https://youtu.be/CD_t3m2WMM8?t=1819)

The main difference between this and a Composite `ControlValueAccessor` is that this does not implement that interface. Unfortunately, this doesn't make it less complicated: Arguably, it's more so.

Complicating things further is the fact that there's a couple of mistakes in the slides, which I'll point out.
The first of these is at around 32:50 where the component is called 'RequiredText' instead of 'AddressComponent', as in the example. The released slides correct this error.

Let's now have a closer look at what we mean by a *Sub Form Component*.

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
We might just expect this to work since we haven't changed the actual HTML in anyway, but instead we get an error:
```
Error: NodeInjector: NOT_FOUND [ControlContainer]
```

So what's a `ControlContainer`?
Angular docs says this of it:
> A base class for directives that contain multiple registered instances of NgControl.

If we look at the list of subclasses we see `NgForm` amongst them. From this we can deduce that the `ControlContainer` is in fact our form.
What is happening is that our component is looking for `NgForm` in the injector but can't find it.
The reason is that `NgForm` is configured that it will only inject an instance of ControlContainer if it exists in an injector that is not higher in the hierarchy of element injectors than that for the template of the host component. 

The way to fix this is to provide `NgForm` within the `viewProviders` array of `AddressComponent`:
```
 viewProviders: [
    { provide: ControlContainer, useExisting: NgForm}
  ],
```

But what if the `ControlContainer` isn't `NgForm`? After all, as we saw earlier, `ControlContainer` has many subclasses.

Imagine we change our form so that the address component has as its parent an `ngModelGroup` instead of an `ngModelForm`:
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
We will find that our form data now looks like this:
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
Contrary to what we might expect, `address` is not nested within `home`.
What has happened is that the Address Component is still configured to see `NgForm` as its `ControlContainer` and cannot see the `NgModelGroup` at all.

If we change `viewProviders` so that `ControlContainer` looks for an `NgModelGroup` instead:
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
This illustrates an important constraint of Sub Form Components: You have to be careful about how you nest them as you may experience unexpected behaviour.
This same constraint is why Sub Forms are not interchangeable with Reactive and Template Driven forms.

[code example](https://stackblitz.com/edit/angular-sub-form-component)

Now lets look at a Sub Form as implemented in a Reactive form.

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
As before, we register the `ControlContainer` in the `viewProviders` array, this time assigning it to be a `FormGroupDirective` instead of `NgModelGroup`:
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
This is because the `address` `FormGroup` does not exist. We need to create it within `AddressComponent`. 

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
In the slides for this there's another mistake.`this.form` within the constructor wont work because parent.form is null at this point.
There's a discussion about this [here](https://github.com/angular/angular/issues/23914)
 
[code example](https://stackblitz.com/edit/angular-sub-form-component-reactive-form)

## Form Projection
[36:54](https://youtu.be/CD_t3m2WMM8?t=2214)

Form projection is where you project content into a form element. 
E.g, you have a wrapper component that looks something like this:
```
<form>
  <ng-content></ng-content>
</form>
```
And it's used like this:
```
<wrapper-component>
  <input />
</wrapper-component>
```
The `<input/>` element will be *projected* into the form so that the final DOM looks like:
```
<form>
  <input/>
</form>
```
This approach is somewhat complex to get working and Kara advises against it, nonetheless she shows a way of making it work.

In the demo, this is the content that is being projected into a form:
```
  <div ngModelGroup="address">
    <input name="street" ngModel/>
    <input name="city" ngModel/>
  </div>
```
[code example](https://stackblitz.com/edit/angular-form-projection-1) is a demo that shows this.

Note the error message:
```
NodeInjector: NOT_FOUND [ControlContainer]
```
This is the same error we got with Sub Form components.
Once again, `ngModelGroup` cannot find it's `ControlContainer`: the `NgForm` directive. 
The same solution will not work here though as the situation is a little different:
For the `FormStepper` component we have a View Child, in which the `<form>` exists, and a Content Child, where we find the `<input/>` elements.
The View Child and the Content Child are not able to directly contact each other: they must do so through the component.

Within the component, we obtain the `NgForm` directive using a `@ViewChild` query.
We then provide this to the Content Child by adding it to the `Providers` array using `useFactory`. This should allow `ngModelGroup` to get a reference to `NgForm` through dependency injection.

[code example](https://stackblitz.com/edit/angular-form-projection-2)

But this too is insufficient. The reason is that `useFactory()` runs before the template view is created, thus, the form reference it gets is still undefined.
What we need to do is get this factory function to run *after* the view has been created.

Kara discusses a solution using `ngTemplateOutlet` which I have tried to implement, but I have been unable to get it to work.

[code example](https://stackblitz.com/edit/angular-form-projection-3)

It's not clear to me that it's possible for it to work.

I can't honestly see why `ngTemplateOutlet` rather than `ng-content` for inserting the Content Children should make any difference to when the factory is called.
[This experiment](https://stackblitz.com/edit/angular-projection-experiment) shows that the factory runs before the view is initialised and so the form that it attempts to register in the injector is still undefined. 

I have posted a [question](https://stackoverflow.com/questions/65008417/how-to-implement-form-projection-in-angular) on Stackoverflow to see if anyone else has an answer to this problem, but for the moment I'm stuck.

## Conclusion
This is by far the best resource I've managed to find on advanced Angular form techniques. It's frustrating that the code for it was never released. I have tried my best to create this code myself but unfortunately I was unable to do so for form projection. I hope that readers of this are able to benefit from where I was successful and are able to assist me in those places where I wasn't.

## Resources
* Another good resource regarding the `ControlValueAccessor` interface is [this blog](https://jenniferwadella.com/blog/understanding-angulars-control-value-accessor-interface) by Jennifer Wadella. She also does some talks on the same subject that can be found on YouTube.

* Ward Bell, wrote an [article](https://medium.com/@wardbell/wow-you-know-i-love-your-articles-we-are-almost-always-on-the-same-wave-length-11e0d53f7da3) about Reactive forms that mentions this talk. He remarks that he's somewhat confused by the section on nested forms.
