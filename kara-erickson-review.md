# Commentary on Kara Erickson's talk on Angular Forms
This article is a commentary on a [talk](https://www.youtube.com/watch?v=CD_t3m2WMM8) that was given by Angular Core developer Kara Erickson at Angular Connect in 2017.

This talk concerned some advanced topics in relation to Angular forms, such as Custom Form Controls, nested forms, and form projection, important subjects which, by and large, aren't covered particularly well in the Angular docs themselves. Although the talk was given a few years ago, it is still relevant as the API has not changed substantially since then.

Whilst the information is good, unfortunately the code shown in the demos was never released, as far as I can ascertain. In this article I will discuss my efforts to recreate the code and also clarify a few things from the talk that I initially found confusing.

The prerequisite for reading this article is that you have watched this video (and presumably know something about Angular).

The first nine minutes of the talk comprises a refresher on Angular forms and an introduction to the (then) new `updateOn` option. It's fairly uncontroversial so I'm skipping that.

## Custom Form Controls
[9:02](https://youtu.be/CD_t3m2WMM8?t=542)

A Custom Form Control is a directive that implements the `ControlValueAccessor` interface. This results in the directive being able to integrate with Angular's Form API. It can be used in an Angular form just as any native input element can be. It also works interchangeably with both Reactive and Template Driven forms. So once you have created a Custom Form Control, it can be used with both of these forms modules. 

### Why would you want to use them?
[9:15](https://youtu.be/CD_t3m2WMM8?t=555)

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
[12:25](https://youtu.be/CD_t3m2WMM8?t=745)

The `ControlValueAccessor` interface looks like this, comprising 3 required methods and 1 optional one:
//todo: fix this interface
```
  writeValue(value: any) {}
  registerOnChange(fn: (value: any) => void) {}
  registerOnTouched(fn: () => void) {}
  setDisabledState(isDisabled: boolean) {}
```
I'm going to go through some problems I had implementing the first two of these methods.
The two remaining methods, `registerOnTouched()` and `setDisabledState()` were straightforward to implement and didn't present me with problems.

### writeValue()
[14:30](https://youtu.be/CD_t3m2WMM8?t=870)

The `writeValue()` method is called by the forms API to set values into our component within the DOM.
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
This differs from Kara's example in that an extra check is needed to make sure that the `input` property exists before attempting to use it. This check is needed when the component is used in a Reactive form, but not in a Template Driven form. Apparently, the Angular Reactive Forms API calls `writeValue()` before  `input` is set.

### registerOnChange()
[14:45](https://youtu.be/CD_t3m2WMM8?t=885)

The `registerOnChange()` method is called by the forms API to pass a callback to our code which we must call whenever there is a change within our component.
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

Having implemented a `ControlValueAccessor`, we need to let Angular's Form API know about it. We do this by registering it with the local injector using the NG_VALUE_ACCESSOR token.
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
The `NG_VALUE_ACCESSOR` is a Dependency Injection token representing classes that implement the `ControlValueAccessor` interface.
The `multi` property means that we can register multiple providers with the token.
`useExisting` means that we use the existing instance of MyComponent that the injector has already created. Obviously, RequiredText is already in the injector so we are aliasing it here.

[code example](https://stackblitz.com/edit/angular-required-text-component)

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
[25:22 ](https://youtu.be/CD_t3m2WMM8?t=1522)

A *nested form* is a component that comprises part of a form. A common example is of an address component that contains separate fields for street, city, postcode etc. The motivation for having a nested form is similar to that of a Custom Form Component - to group related behaviour, for ease of re-use etc.

There are two kinds of nested form components (NFCs) that Kara talks about.
* *Composite ControlValueAccessor Component*: A ControlValueAccessor component which contains an arbitrary number of form controls instead of just one.
* *Sub Form Component*: A Component that contains a form fragment but does not implement the ControlValueAccessor interface.

### Composite ControlValueAccessor Component (CCC)
[26:25](https://youtu.be/CD_t3m2WMM8?t=1585)
 
[stackblitz example](https://stackblitz.com/edit/angular-composite-control-value-accessor)

Implementing a CCC is largely the same as implementing a CFC. Kara doesn't say anything about validation. Whilst validation and error messages can be self-contained within the component, it's important to implement the `validate()` method so that the component's valid status stays in sync with the containing form.

This seems to me to be the best way to do nested forms: It gives the greatest amount of flexibility and reusability. The component works in the same way as a native input element making it easier to compose complex forms out of them.

### Sub Form Component (SFC)
[30:19](https://youtu.be/CD_t3m2WMM8?t=1819)

[Sub Form Component using template driven form](https://stackblitz.com/edit/angular-sub-form-component)
[Sub Form Component using reactive form](https://stackblitz.com/edit/angular-sub-form-component-reactive-form)

The main difference between this an a CCC is that this does not implement the `ControlValueAccessor` interface. Unfortunately, this doesn't make it less complicated to implement: In fact, it's probably it's more complicated. Ward Bell, in an [article](https://medium.com/@wardbell/wow-you-know-i-love-your-articles-we-are-almost-always-on-the-same-wave-length-11e0d53f7da3) about Reactive forms that mentions this talk, comments about being confused by this section.

It should be pointed out that there's a mistake in the slide shown at around 32:50 where the component is called 'RequiredText' instead of 'AddressComponent' as in the example. The released slides correct this error, but this could easily confuse people, as it did me.

Let's have a closer look at what we mean by a *Sub Form Component*.

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
We might just expect this to work since we haven't changed the actual HTML in anyway, but in fact we end up with an error:
```
Error: NodeInjector: NOT_FOUND [ControlContainer]
```

So what's a `ControlContainer`?
Angular docs says this:
> A base class for directives that contain multiple registered instances of NgControl.

If we look at the list of subclasses we see `NgForm` amongst them. From this we can deduce that the `ControlContainer` is in fact our form.
What is happening is that our component is looking for NgForm in the injector but can't find it for some reason.
The reason is that `NgForm` is configured that it will only inject an instance of ControlContainer if it exists in an injector that is not higher in the hierarchy of element injectors than that for the template of the host component. 

The way to fix this is to provide NgForm within the viewProviders array of AddressComponent.
```
 viewProviders: [
    { provide: ControlContainer, useExisting: NgForm}
  ],
```

But what if the ControlContainer isn't NgForm? After all, as we saw earlier, ControlContainer has many subclasses.

Let's investigate. Imagine we change our form so that the address component has as its parent an ngModelGroup instead of an ngModelForm:
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
What has happened is that AddressComponent is still configured to see NgForm as its ControlContainer and cannot see the NgModelGroup at all.

If we change viewProviders so that ControlContainer looks for an NgModelGroup instead:
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

Lets look at a Sub Form as implemented for Reactive forms.

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
[36:54](https://youtu.be/CD_t3m2WMM8?t=2214)

Form projection is where you project content into a form element. 
E.g, you have a wrapper component something like this:
```
<form>
  <ng-content></ng-content>
</form>
```
And it's used something like this:
```
<wrapper-component>
  <input />
</wrapper-component>
```
This approach is somewhat complex to get working and Kara advises against it, nonetheless she shows a way of making it work.

In the demo, this is the content that is being projected into a form:
```
    <div ngModelGroup="address">
       <input name="street" ngModel/>
       <input name="city" ngModel/>
    </div>
```
[here](https://stackblitz.com/edit/angular-form-projection-1) is a demo that shows this.

Note the error message:
```
NodeInjector: NOT_FOUND [ControlContainer]
```
This is similar to the error we got with Sub Form components.
The problem is that `ngModelGroup` cannot find it's container. Given what we know about component boundaries, this is not surprising. Obviously, it's complicated by the fact that we're dealing with projected content. The Content Child is looking for the form directive that lives in the View Child, but these entities are able to directly communicate with each other. 

Now we provide the ControlContainer in a factory. This gives us a different error
If we can register the form in the host component's injector, the Content Child will have access to it. So the trick is to obtain the form from the view child using a @ViewChild query, then make this available to the Content Child by registering it in the component's providers array using a factory. 
[here](https://stackblitz.com/edit/angular-form-projection-2) is a demo of this.

This gives us a different error.
```
Error:
ngModelGroup cannot be used with a parent formGroup directive.
```
I don't fully understand this error, but the problem with the code as it stands is that the factory function runs *before* the view is initialised. Thus it gets a reference to the form that is `undefined`.
Kara discusses a solution using `ngTemplateOutlet` which I have implemented [here](https://stackblitz.com/edit/angular-form-projection-3), but I have been unable to get it to work.

It's not clear to me that it's possible for it to work.
[This experiment](https://stackblitz.com/edit/angular-projection-experiment) shows that the factory runs before the view is initialised and so the form that it attempts to register in the injector is still undefined. Why should using `ngTemplateOutlet` make the factory run later?

It seems to me that the `ngModelGroup` directive should be instantiated when the content template is created. At that point, the injector should call the factory and inject the form into the directive. But the form at this point is undefined. What we need is to be able to instantiate the Content Child after the View, but I don't know how to do this.

I have posted a question on Stackoverflow to see if anyone else has an answer to this problem, but for the moment I do not know how to solve it.

## Conclusion
This is by far the best resource I've managed to find on advanced Angular form techniques. It's sad that it has some serious shortcomings. The main one is that no code was ever supplied for it. I have tried my best to create this code myself but unfortunately I was unable to do so for form projection. I hope that readers of this are able to benefit from where I was successful and are able to assist me in those places where I wasn't.

## Resources
Another good resource regarding the `ControlValueAccessor` interface is [this blog](https://jenniferwadella.com/blog/understanding-angulars-control-value-accessor-interface) by Jennifer Wadella. She also does some talks on the same subject that can be found on YouTube.



