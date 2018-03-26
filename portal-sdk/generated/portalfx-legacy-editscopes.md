
<a name="legacy-edit-scopes"></a>
## Legacy Edit Scopes

**NOTE**:  EditScopes are becoming obsolete.   It is recommended that extensions be developed without edit scopes, as specified in [portalfx-editscopeless-procedure.md](portalfx-editscopeless-procedure.md).

<!-- TODO:  Compare document with the pieces of the portalfx-editscopes*.md document -->
<!-- TODO:  These documents and the portalfx-editscopeless*.md documents are still in work -->

<a name="legacy-edit-scopes-working-with-edit-scopes"></a>
### Working with Edit Scopes

Edit scopes provide a standard way of managing edits over a collection of input fields, blades, and extensions. They provide many common functions that would otherwise be difficult to orchestrate, like the following:

  * Track changes in field values across a form
  * Track validation of all fields in a form
  * Provide a way to discard all changes in a form
  * Persist unsaved changes from the form to the cloud
  * Simplify merging changes from the server into the current edit

In some instances, development approaches that do not use `editScopes` are preferred.  For more information about forms without editScopes, see  [portalfx-editscopeless-overview.md](portalfx-editscopeless-overview.md) and [portalfx-controls-dropdown.md#migration-to the-new-dropdown.md](portalfx-controls-dropdown.md#migration-to the-new-dropdown.md).

In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>` is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK. 

Any edits that were made by the user and collected in an `EditScope` are saved in the storage for the browser session. This is managed by the shell. Parts or blades may request an `EditScope`, but the most common usage is in a blade. A blade defines a `BladeParameter` with a `Type` of `NewEditScope`. This informs the shell that a blade is asking for a new `editScope` object. Within the rest of the blade, that parameter can be attached to an `editScopeId` property on any part. Using this method, many parts and commands on the blade can all read from the same editScopeId. This is common when a command needs to save information about a part. After sending the `editScopeId` to the part as a property, the `viewModel`  loads the `editScope` from the cloud. 

For more information about edit scopes and  managing unsaved edits, watch the video located at 
[https://aka.ms/portalfx/editscopes](https://aka.ms/portalfx/editscopes).

1. Request an editscope from pdl
1. Load an edit scope
1. Load an edit scope from a data context
1. EditScopeAccessor
1. Create new objects and bind them to the editScope 

<a name="legacy-edit-scopes-request-an-editscope-from-pdl"></a>
### Request an editscope from pdl

The following sample code demonstrates requesting an `editScope` from PDL.  The sample is also located at `<dir>\Client\V1\MasterDetail\MasterDetailEdit\MasterDetailEdit.pdl`.  The `valid` is using the `section` object of the form to determine if the form is currently valid.  This is not directly related to the edit scope, but will be relevant in the section named []().


```xml
<!-- Display detail blade with an edit scope. Blade consists of a form and commands.-->
<Blade Name="DetailBlade"
       ViewModel="DetailBladeViewModel">
    <Blade.Parameters>
        <Parameter Name="currentItemId" Type="Key" />
        <Parameter Type="NewEditScope" />
        <Parameter Name="formValid" Type="Output" />
    </Blade.Parameters>

    <Lens Title="SamplesExtension.Resources.Strings.masterDetailEditDetailTitle">
        <CustomPart Name="DetailPart"
                    ViewModel="DetailPartViewModel"
                    Template="{Html Source=&#039;Templates\\WebsitesDetail.html&#039;}"
                    InitialSize="HeroWideFitHeight">
        <CustomPart.Properties>
            <!-- Generated by the shell. -->
            <Property Name="editScopeId"
                      Source="{BladeParameter editScopeId}" />
            <!-- Output parameter indicating whether the form is valid. -->
            <Property Name="valid"
                      Source="{BladeParameter formValid}"
                      Direction="Output" />
            <!-- Master passes an id of object that will be used to seed the edit scope. -->
            <Property Name="currentItemId"
                      Source="{BladeParameter currentItemId}" />
        </CustomPart.Properties>
      </CustomPart>
    </Lens>
</Blade>
```

<a name="legacy-edit-scopes-load-an-edit-scope"></a>
### Load an edit scope

The data in the `editScope` includes original values and saved edits. The method to access inputs on a part is the `onInputsSet` method. In the constructor, a new `MsPortalFx.Data.EditScopeView` object is created from the `dataContext`. The `EditScopeView` provides a stable observable reference to an `EditScope` object. The `editScopeId` will be sent in as a member of the `inputs` object when the part is bound.

For an example of loading an edit scope, view the following code.  The sample is also located at `<dir>\Client\V1\MasterDetail\MasterDetailEdit\ViewModels\DetailViewModels.ts`. 

<!-- TODO: Determine whether this inline code should be replaced when the samples code is replaced and linked in gitHub, or whether the links to the SDK samples are sufficient. -->
```ts
// create a new editScopeView
constructor(container: MsPortalFx.ViewModels.PartContainerContract,
            initialState: any,
            dataContext: DataContext) {
    super();
    ...
    this._editScopeView = dataContext.masterDetailEditSample.editScopeCache.createView(container);
    // Initialize editScope of the base class.
    this.editScope = this._editScopeView.editScope;
    ...
}
```

```ts
// update the editScopeView with a new id
public onInputsSet(inputs: any): MsPortalFx.Base.Promise {
    // Acquires edit scope seeded with an item with id currentItemId.
    return this._editScopeView.fetchForExistingData(inputs.editScopeId, inputs.currentItemId);
}
```

<!--TODO:  Determine which piece of code is associated with this paragraph.  -->

 The code that loads the `EditScope` is largely related to data loading, so the data context is the preferred location for the code. 

<a name="legacy-edit-scopes-load-an-edit-scope-from-a-data-context"></a>
### Load an edit scope from a data context

 <!--TODO:  The following sample code does not exist by this name.  -->
 For an example of loading an edit scope from a data context, view the sample that is located at `<dir>\Client\Data\MasterDetailEdit\MasterDetailEditData.ts`.
 <!--TODO:  The  previous sample code does not exist by this name.  -->

The following code creates a new `EditScopeCache`, which is bound to the `DataModels.WebsiteModel` object type. It uses an edit scope within a form.  The `fetchForExistingData()` method on the cache provides a promise, which informs the `ViewModel` that the `editScope` is loaded and available.

```ts
this.editScopeCache = MsPortalFx.Data.EditScopeCache.create<DataModels.WebsiteModel, number>({
    entityTypeName: DataModels.WebsiteModelType,
    supplyExistingData: (websiteId: number) => {
        var deferred = $.Deferred<JQueryDeferredV<DataModels.WebsiteModel>>();

        this.initializationPromise.then(() => {
            var website = this.getWebsite(websiteId);
            if (website) {
                deferred.resolve(website);
            } else {
                deferred.reject();
            }
        });
        return deferred;
    }
});
```

<a name="legacy-edit-scopes-editscopeaccessor"></a>
### EditScopeAccessor

Form fields require a binding to one or more `EditScope` observables. Consequently, they have two constructor overloads.  Extension developers can configure this binding by supplying a path from the root of the EditScope/Form model down to the observable to which the form field should bind. They can do this by selecting one of the two form field constructor variations. 

In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>`  is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK.

1. **EditScopeAccessor** This is the preferred, compile-time verified methodology. The form field ViewModel constructor accepts an EditScopeAccessor, wraps a compile-time verified lambda, and returns the EditScope observable to which the Form field should bind, as in the following code.

    `<dir>/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts`

    ```typescript

this.textBoxSimpleAccessor = new MsPortalFx.ViewModels.Forms.TextBox.ViewModel(
    container,
    this,
    this.createEditScopeAccessor<string>((data) => { return data.state; }),
    textBoxSimpleAccessorOptions);

``` 

    The EditScopeAccessor methodology is preferred for the following reasons.

    * The supplied lambda will be compile-time verified. This code is more maintainable, for example, when the property names on the Form model types are changed.
    * There are advanced variations of `EditScopeAccessor` that enable less-common scenarios like binding multiple `EditScope` observables to a single form field or translating form model data for presentation to the user, as in the following code.

      `<dir>/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts`
  
    ```typescript

this.textBoxReadWriteAccessor = new MsPortalFx.ViewModels.Forms.TextBox.ViewModel(
    container,
    this,
    this.createEditScopeAccessor<string>(<MsPortalFx.ViewModels.Forms.EditScopeAccessors.Options<FormIntegratedFormData.FormIntegratedFormData, string>>{
        readFromEditScope: (data: FormIntegratedFormData.FormIntegratedFormData): string => {
            return data.state2().toUpperCase();
        },
        writeToEditScope: (data: FormIntegratedFormData.FormIntegratedFormData, newValue: string): void => {
            data.state2(newValue);
        }
    }),
    textBoxReadWriteAccessorOptions);

```

1. **String-typed path** This methodology is discouraged because it is not compile-time verified. The form field ViewModel constructor accepts a string-typed path that contains the location of the EditScope observable to which the Form field should bind, as in the following code.

    `<dir>/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts`

  ```typescript

this.textBoxViewModel = new MsPortalFx.ViewModels.Forms.TextBox.ViewModel(container, this, "name", textBoxOptions);

``` 

<!-- TODO:  The following content seems to belong with editscopes instead of the form documents.  However, it is not properly formatted.  -->


### Create new objects and bind them to the editScope 

The following code creates a new set of form field objects and binds them to the `editScope`. The sample is also located at  `<dir>\Client\V1\MasterDetail\MasterDetailBrowse\ViewModels\DetailViewModels.ts`.  

```ts
private _initializeForm(): void {

        // Form fields.
        var websiteNameFieldOptions = <MsPortalFx.ViewModels.Forms.TextBoxOptions>{
            label: ko.observable(ClientResources.masterDetailEditWebsiteNameLabel),
            validations: ko.observableArray([
                new MsPortalFx.ViewModels.RequiredValidation(ClientResources.masterDetailEditWebsiteNameRequired),
                new MsPortalFx.ViewModels.RegExMatchValidation("^[a-zA-Z _]+$", ClientResources.masterDetailEditWebsiteNameValidation)
            ]),
            emptyValueText: ko.observable(ClientResources.masterDetailEditWebsiteNameInitial),
            labelPosition: ko.observable(MsPortalFx.ViewModels.Forms.LabelPosition.Left)
        };

        this.websiteNameField = new MsPortalFx.ViewModels.Forms.TextBox(this._container, this, "name", websiteNameFieldOptions);

        var isRunningFieldOptions = <MsPortalFx.ViewModels.Forms.OptionsGroupOptions<boolean>>{
            label: ko.observable(ClientResources.masterDetailEditRunningLabel),
            options: ko.observableArray([
                {
                    text: ko.observable(ClientResources.masterDetailEditRunningOn),
                    value: true
                },
                {
                    text: ko.observable(ClientResources.masterDetailEditRunningOff),
                    value: false
                }
            ]),
            labelPosition: ko.observable(MsPortalFx.ViewModels.Forms.LabelPosition.Left)
        };

        this.isRunningField = new MsPortalFx.ViewModels.Forms.OptionsGroup(this._container, this, "running", isRunningFieldOptions);

        var generalSectionOptions = <MsPortalFx.ViewModels.Forms.SectionOptions>{
            children: ko.observableArray([
                this.websiteNameField,
                this.isRunningField
            ]),
            style: ko.observable(MsPortalFx.ViewModels.Forms.SectionStyle.Wrapper),
        };

        this.generalSection = new MsPortalFx.ViewModels.Forms.Section(this._container, generalSectionOptions);
    }
```

For more information about form fields, see [portalfx-controls.md#forms](portalfx-controls.md#forms).

### Editable entity arrays

EditScope `EntityArray` objects were designed with a few requirements in mind.
* The UI edits are serialized so that Journey-switching works with unsaved Form edits. For editing large arrays, the FX should not serialize array edits by persisting two full copies of the potentially very large array.
* In the UI, the FX renders an indication of what array items were created/updated/deleted. In some cases, array removes are rendered with strike-through styling.
* Array adds/removes are revertable for some scenarios.

These factors made EditScope `EntityArrays` behave differently than regular JavaScript arrays, as follows.
* `Creates` are kept out-of-band
* `Deletes` are non-destructive

Rows can be added or removed from an editable grid, but the corresponding adds/removes may not be immediately viewable from the EditScope array. To conveniently see the actual state of an EditScope `EntityArray`, use the `getEntityArrayWithEdits` EditScope method. This returns the following types of arrays.
* An array that includes 'created' entities and does not include 'deleted' entities
* Discrete arrays that individually capture 'created', 'updated' and 'deleted' entities  

A pair of EditScope methods significantly simplifies working with EditScope `EntityArrays`.  

* The `getEntityArrayWithEdits` method is particularly useful in the `mapOutgoingDataForCollector` callback of the `ParameterProvider` when returning an edited array to some ParameterCollector, as in the following code.

    ```typescript

this.parameterProvider = new MsPortalFx.ViewModels.ParameterProvider<DataModels.ServerConfig[], KnockoutObservableArray<DataModels.ServerConfig>>(container, {
    editScopeMetadataType: DataModels.ServerConfigType,
    mapIncomingDataForEditScope: (incoming) => {
        return ko.observableArray(incoming);  // Editable grid can only bind to an observable array.
    },
    mapOutgoingDataForCollector: (outgoing) => {
        const editScope = this.parameterProvider.editScope();

        // Use EditScope's 'getEntityArrayWithEdits' to return an array with all created/updated/deleted items.
        return editScope.getEntityArrayWithEdits<DataModels.ServerConfig>(outgoing).arrayWithEdits;
    }
});

```

* There is a corresponding `applyArrayAsEdits` EditScope method that simplifies applying edits to an existing EditScope `EntityArray`. This is often done in a ParameterCollector's `receiveResult` callback, as in the following example.

    ```typescript

this.itemsCollector = new MsPortalFx.ViewModels.ParameterCollector<DataModels.ServerConfig[]>(container, {
    selectable: this.itemsSelector.selectable,
    supplyInitialData: () => {
        const editScope = this._editScopeView.editScope();

        // Use EditScope's 'getEntityArrayWithEdits' to develop an array with all created/updated/deleted items
        // in this entity array.
        return editScope.getEntityArrayWithEdits<DataModels.ServerConfig>(editScope.root.serverConfigs).arrayWithEdits;
    },
    receiveResult: (result: DataModels.ServerConfig[]) => {
        const editScope = this._editScopeView.editScope();

        // Use EditScope's 'applyArrayWithEdits' to examine the array returned from the Provider Blade
        // and apply any differences to our EditScope entity array in terms of created/updated/deleted entities.
        editScope.applyArrayAsEdits(result, editScope.root.serverConfigs);
    }
});

```

* * *

<a name="legacy-edit-scopes-key-value-pairs"></a>
### Key-value pairs

<!-- TODO: Determine whether this section remains here, or should be moved to the edit grid document -->

***Q: My Form data is just key/value-pairs. How do I model a Dictionary/StringMap in EditScope? Why can't I just use a JavaScript object like a property bag?***

Like all models and view models treated by the Azure Portal FX, after the object/array is instantiated and returned to the FX in the process of rendering the UI, any subsequent mutation of that object/array should be done by changing/mutating **Knockout** observables. Portal controls and Knockout HTML template rendering both subscribe to these observables and re-render only when these observables change value.

This poses a challenge to the common JavaScript programming technique of treating a JavaScript object as a property bag of key-value-pairs. If an extension only adds or removes keys from a model or ViewModel object, the FX will not be aware of these changes. EditScope will not recognize these changes as user edits and therefore such key adds/removes will put EditScope edit-tracking in an inconsistent state.

So, how does an extension model a Dictionary/StringMap/property bag when using the Portal FX and EditScope?  

The pattern is to develop the Dictionary/StringMap/property bag as an observable array of key/value-pairs, like `KnockoutObservableArray<{ key: string; value: TValue; }>`. 

Often, additionally, it is important to let users edit the Dictionary/StringMap/property bag using *an editable grid*. In such cases, it is important to describe the array of key/value-pairs as an 'entity' array since editable grid can only be bound to an EditScope 'entity' array. For more information about how to develop type metadata to use the array with editable grid, see [#editable-grid](#editable-grid).

Here's a sample that does something similar, converting - in this case - an array of strings into an 'entity' array for consumption by editable grid.  

* * *
<a name="legacy-edit-scopes-q-what-should-be-returned-from-saveeditscopechanges-i-don-t-understand-the-different-values-of-the-accepteditscopechangesaction-enum"></a>
### Q: What should be returned from &#39;saveEditScopeChanges&#39;? I don&#39;t understand the different values of the <code>AcceptEditScopeChangesAction</code> enum.

<!-- TODO:  Move this to the EditScope document -->

SOLUTION: 
When creating an EditScopeCache, the `saveEditScopeChanges` callback supplied by the extension is called to push EditScope edits to a server/backend. This callback returns a `Promise` that should be resolved when the 'save' AJAX call completes, which occurs after the server/backend accepts the user's edits. 

When the extension resolves this `Promise`, it can supply a value that instructs the EditScope to  reset itself to a clean/unedited state. If no such value is returned during Promise resolution, then the EditScope is reset by default.  This means taking the user's client-side edits and considering these values to be the new, clean/unedited EditScope state. This works for many scenarios.  

There are scenarios where the default `saveEditScopeChanges` behavior does not work, like: the following. 
* During 'save', the server/backend produces new data values that need to be merged into the EditScope.
* For "create new record" scenarios, after 'save', the extension should clear the form, so that the user can enter a new record.

For these cases, the extension will resolve the `saveEditScopeChanges` Promise with a value from the `AcceptEditScopeChangesAction` enum. These values allow the extension to specify conditions like the following.
* The EditScopeCache is to implicitly reload the EditScope data as part of completing the 'save' operation.
* The EditScope's data is to be reverted/cleared as part of completing the 'save' operation (the "create new record" UX scenario).

For more information about each enum value, see the jsdoc comments around `MsPortalFx.Data.AcceptEditScopeChangesAction` or in `MsPortalFxDocs.js` in Visual Studio or any code editor.

<a name="legacy-edit-scopes-q-what-should-be-returned-from-saveeditscopechanges-i-don-t-understand-the-different-values-of-the-accepteditscopechangesaction-enum-caveat-anti-pattern"></a>
#### Caveat / anti-pattern

There is an important anti-pattern to avoid here re: '`saveEditScopeChanges`'. If the AJAX call that saves the user's edits fails, the extension should merely **reject** the '`saveEditScopeChanges`' Promise (which is natural to do with Q Promise-chaining/piping). The extension *should not* resolve their Promise with '`AcceptEditScopeChangesAction.DiscardClientChanges`', since this will lose the user's Form edits (a data-loss bug).
  
* * *

<a name="legacy-edit-scopes-common-error-entity-typed-object-array-is-not-known-to-this-edit-scope"></a>
### Common error: &quot;Entity-typed object/array is not known to this edit scope...&quot;

SOLUTION: 
EditScope data follows a particular data model. In short, the EditScope is a hierarchy of 'entity' objects. By default, when the EditScope's '`root`' is an object, this object is considered an 'entity'. The EditScope becomes a hierarchy of 'entities' when:
* the EditScope includes an array of 'entity' objects
* some EditScope object includes a property that is 'entity'-typed  

An object is treated by EditScope as an 'entity' when type metadata associated with the object is marked as an 'entity' type (see [here](#entity-type) and the EditScope video located at [here](portalfx-legacy-editscopes.md) for more details).

Every 'entity' object is tracked by the EditScope as being created/updated/deleted. Extension developers define 'entities' at a granularity that suit their scenario, making it easy to determine what in their larger EditScope/Form data model has been user-edited.  

Now, regarding the error above, once the EditScope is initialized/loaded, 'entities' can be introduced and removed from the EditScope only via EditScope APIs. It is an unfortunate design limitation of EditScope that extensions cannot simply make an observable change to add a new 'entity' object or remove an existing 'entity' from the EditScope. If an extension tries to do an observable add (make an observable change that introduces an 'entity' object into the EditScope), they'll encounter the error discussed here.  

To correctly (according to the EditScope design) add or remove an 'entity' object into/out of an EditScope, here are APIs that are useful:
* ['`applyArrayAsEdits`'](#apply-array-as-edits) - This API accepts a new array of 'entity' objects. The EditScope proceeds to diff this new array against the existing EditScope array items, determine which 'entity' objects are created/updated/deleted, and then records the corresponding user edits.
* '`getCreated/addCreated`' - These APIs allow for the addition of new, 'created' entity objects. The '`getCreated`' method returns a distinct, out-of-band array that collects all 'created' entities corresponding to a given 'entity' array. The '`addCreated`' method is merely a helper method that places a new 'entity' object in this '`getCreated`' array.
* '`markForDelete`' - 'Deleting' an 'entity' from the EditScope is treated as a non-destructive operation. This is so that - if the extension chooses - they can render 'deleted' (but unsaved) edits with strike-through styling (or equivalent). Calling this '`markForDelete`' method merely puts the associated 'entity' in a 'deleted' state.  

* * *

<a name="legacy-edit-scopes-common-error-entity-typed-object-array-is-not-known-to-this-edit-scope-common-error-scenario"></a>
#### Common error scenario

SOLUTION: 
Often, extensions encounter this "Entity-typed object/array is not known to this edit scope..." error as a side-effect of modeling their data as 'entities' binding with [editable grid](#editable-grid) in some ParameterProvider Blade.  Then, commonly, the error is encountered when applying the array edits in a corresponding ParameterCollector Blade.  Here are two schemes that can be useful to avoid this error:
* Use the [`applyArrayAsEdits'](#apply-array-as-edits) EditScope method mentioned above to commit array edits to an EditScope.
* Define type metadata for this array *twice* - once only for editing the data in an editable grid (array items typed as 'entities'), and separately for committing to an EditScope in the ParameterCollector Blade (array items typed as not 'entities').  
  
* * *

<a name="legacy-edit-scopes-editable-object-not-present"></a>
### Editable object not present

***"Encountered a property 'foo' on an editable object that is not present on the original object..."***

SOLUTION: 
As discussed in [portalfx-extensions-faq-forms.md#key-value-pairs](portalfx-extensions-faq-forms.md#key-value-pairs), the extension should mutate the EditScope/Form model by making observable changes and by calling EditScope APIs. For any object residing in the `EditScope`, merely adding and removing keys cannot be detected by `EditScope` or by the FX at large and, consequently, edits cannot be tracked. When an extension attempts to add or remove keys from an `EditScope` object, this puts the `EditScope` edit-tracking in an inconsistent state. When the `EditScope` detects such an inconsistency, it issues the `Encountered a property...` error to encourage the extension developer to use only observable changes and `EditScope` APIs to mutate/change the EditScope/Form model.  

* * *