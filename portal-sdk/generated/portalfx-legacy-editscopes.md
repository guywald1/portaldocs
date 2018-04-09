
<a name="legacy-edit-scopes"></a>
## Legacy Edit Scopes

**NOTE**:  EditScopes are becoming obsolete.  It is recommended that extensions be developed without edit scopes, as specified in [portalfx-editscopeless-procedure.md](portalfx-editscopeless-procedure.md). For more information about forms without editScopes, see  [portalfx-editscopeless-overview.md](portalfx-editscopeless-overview.md) and [portalfx-controls-dropdown.md#migration-to-the-new-dropdown](portalfx-controls-dropdown.md#migration-to-the-new-dropdown).

<!-- TODO:  Compare document with the pieces of the portalfx-editscopes*.md document -->
<!-- TODO:  These documents and the portalfx-editscopeless*.md documents are still in work -->

Edit scopes provide a standard way of managing edits over a collection of input fields, blades, and extensions. They provide many common functions that would otherwise be difficult to orchestrate, like the following:

  * Track changes in field values across a form
  * Track validation of all fields in a form
  * Provide a way to discard all changes in a form
  * Persist unsaved changes from the form to the cloud
  * Simplify merging changes from the server into the current edit
 
In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>` is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK. 

<a name="legacy-edit-scopes-data-model"></a>
### Data Model

<a name="legacy-edit-scopes-editscope-entity-arrays"></a>
### EditScope entity arrays

An EditScope is a hierarchy of 'entity' objects.  EditScope 'entity' arrays were designed with a few requirements in mind.

* The user's edits need to be serialized so that journey-switching works with unsaved Form edits. For editing large arrays, the FX should not serialize array edits by persisting two full copies of the array.

* In the UI, the FX will indicate which array items were created/updated/deleted by the user. In some cases, array removes are rendered with strike-through styling.

* Array adds/removes need to be revertable for some scenarios.

Consequently, EditScope 'entity' arrays behave differently than regular JavaScript arrays. Importantly:

* 'Creates' are kept out-of-band
* 'Deletes' are non-destructive

By default, when the EditScope's `root` is an object, this object is considered an 'entity'. The EditScope becomes a hierarchy of 'entities' when:

* the EditScope includes an array of 'entity' objects

* some EditScope object includes a property that is 'entity'-typed  

An object is treated by EditScope as an 'entity' when type metadata associated with the object is marked as an 'entity' type.

(see [The getEntityArrayWithEdits method](#the-getentityarraywithedits-method) and the EditScope video located at [https://aka.ms/portalfx/editscopes](https://aka.ms/portalfx/editscopes) for more details).

Every 'entity' object is tracked by the EditScope as being created/updated/deleted. Extension developers define 'entities' at a granularity that suit the scenario, making it easy to determine what data in the  EditScope/Form data model was edited by the user.

After the `EditScope` is initialized and loaded, entities can be introduced and removed from the EditScope only by using EditScope APIs. Unfortunately, extensions cannot simply make an observable change to add a new 'entity' object or remove an existing 'entity' from the EditScope. If an extension tries to make an observable change that introduces an 'entity' object into the EditScope, it will  encounter the error message discussed in [portalfx-extensions-status-codes.md#unknown-entity-typed-object-array](portalfx-extensions-status-codes.md#unknown-entity-typed-object-array).

An `editScope` entity array is an array where created/updated/deleted items are tracked individually by `EditScope`. Array adds/removes are revertable for some scenarios. Rows can be added or removed from an editable grid, but the corresponding adds/removes may not be immediately viewable from the `EditScope` array. To grant this treatment to an array in the `EditScope`/`Form` model, the extension supplies type metadata for the type of the array items. For example, the  `T` in `KnockoutObservableArray<T>` contains the type. 

Any edits that were made by the user and collected in an `EditScope` are saved in the storage for the browser session. This is managed by the shell. Parts or blades may request an `EditScope`, but the most common usage is in a blade. A blade defines a `BladeParameter` with a `Type` of `NewEditScope`. This informs the shell that a blade is asking for a new `editScope` object. Within the rest of the blade, that parameter can be attached to an `editScopeId` property on any part. Using this method, many parts and commands on the blade can all read from the same editScopeId. This is common when a command needs to save information about a part. After sending the `editScopeId` to the part as a property, the `viewModel`  loads the `editScope` from the cloud. 

A pair of EditScope methods significantly simplifies working with EditScope `EntityArrays`.

* [The getEntityArrayWithEdits method](#the-getentityarraywithedits-method)

* [The applyArrayAsEdits method](#the-applyarrayasedits-method)

<a name="legacy-edit-scopes-editscope-entity-arrays-the-getentityarraywithedits-method"></a>
#### The getEntityArrayWithEdits method

To see the actual state of an EditScope `EntityArray`, use the `getEntityArrayWithEdits` EditScope method. The `getEntityArrayWithEdits` EditScope method returns the following types of arrays.

* An array that includes 'created' entities and does not include 'deleted' entities

* Discrete arrays that individually capture 'created', 'updated' and 'deleted' entities

This method is particularly useful in the `mapOutgoingDataForCollector` callback of the `ParameterProvider` when returning an edited array to some ParameterCollector, as in the following code.

gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/ParameterCollection/ParameterProviders/ViewModels/ProviderViewModels.ts", "section": "formsEditScopeFaq#getEntityArrayWithEdits"}

In the UI, the FX renders an indication of what array items were created/updated/deleted. 

The following example converts an array of strings into an 'entity' array for consumption by an editable grid.  The editable grid can only be bound to an `EditScope` 'entity' array.

Modeling your data as an 'entity' array
gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/ParameterCollection/ParameterProviders/ViewModels/ProviderViewModels.ts", "section": "formsEditScopeFaq#makeEntityForEditableGrid"}

Converting your data to an 'entity' array for consumption by an editable grid
gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/ParameterCollection/ParameterProviders/ViewModels/ProviderViewModels.ts", "section": "formsEditScopeFaq#makeEntityForEditableGrid2"}

<a name="legacy-edit-scopes-editscope-entity-arrays-the-applyarrayasedits-method"></a>
#### The applyArrayAsEdits method

The `applyArrayAsEdits` method simplifies applying edits to an existing `EditScope` entity array. This API accepts a new array of 'entity' objects. The `EditScope` will compare  this new array to  the existing `EditScope` array items, determine which 'entity' objects are created/updated/deleted, and then records the corresponding user edits.

This is often done in a ParameterCollector's `receiveResult` callback, as in the following examples.

gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/ParameterCollection/FormWithCollectors/ViewModels/FormWithCollectorsBladeViewModel.ts", "section": "formsEditScopeFaq#applyArrayAsEdits"}

gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/ParameterCollection/ParameterProviders/ViewModels/ProviderViewModels.ts", "section": "formsEditScopeFaq#getEntityArrayWithEdits"}

* * *

* * *

 
* [The EditScopeAccessor](#the-editscopeaccessor)

Some patterns for of using the editScope are as follows.

* [Request an editscope from pdl](#request-an-editscope-from-pdl)

* [Associating the `EditScopeView` object with the `EditScope` for data](#the-editscopeview-object).

* [Load an edit scope from a data context](#load-an-edit-scope-from-a-data-context)

1. [Create new objects and bind them to the editScope](#create-new-objects-and-bind-them-to-the-editscope)


For more information about edit scopes and  managing unsaved edits, watch the video located at 
[https://aka.ms/portalfx/editscopes](https://aka.ms/portalfx/editscopes).

* * *
* * *

<a name="legacy-edit-scopes-editscope-and-ajax"></a>
### EditScope and AJAX

This sample reads and writes data to the server directly via `ajax()` calls. It loads and saves data by creating an `EditScopeCache` object and defining two functions. The `supplyExistingData` function reads the data from the server, and the `saveEditScopeChanges` function writes it back.

In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>` is the `SamplesExtension\` directory. Links to the Dogfood environment will display working copies of the samples that were made available with the SDK.

The code for this example is associated with the basic form sample. It is located at 
* `<dir>\Client\V1\Forms\Samples\Basic\FormsSampleBasic.pdl`
* `<dir>\Client\V1\Forms\Samples\Basic\Templates\FormSampleBasic.html`
* `<dir>\Client\V1\Forms\Samples\Basic\ViewModels\FormsSampleBasicBlade.ts`

The code instantiates an `EditScope` via a `MsPortalFx.Data.EditScopeView` object. When the data to manipulate is already located on the client, an edit scope view can also be obtained from other data cache objects, as in the following example.

gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/Forms/Samples/Basic/ViewModels/FormsSampleBasicBlade.ts", "section": "forms#editScopeCache"}

Then, the code transforms the data to make it match the model type. The server returns strings, but the `WebsiteModel` type that is used is defined in the following code.

  ```ts
   interface WebsiteModel {
      id: KnockoutObservable<number>;
      name: KnockoutObservable<string>;
      running: KnockoutObservable<boolean>;
  }
  ```

Therefore, the save and load functions have to transform the data to make it match the `WebsiteModel` model type. The control `viewModels` require a reference to a `Form.ViewModel`, so the code creates a form and sends the reference to the `editScope` to it, as in the following example.

gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/Forms/Samples/Basic/ViewModels/FormsSampleBasicBlade.ts", "section": "forms#formViewModel"}

This form displays one textbox that allows the user to edit the name of the website, as specified in the following code.

gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/Forms/Samples/Basic/ViewModels/FormsSampleBasicBlade.ts", "section": "forms#controls"}

The form is rendered using a section. The code loads all the controls that should be displayed into the `children` observable array of the section. This positions the controls sequentially on a blade, by default, so it is an easy way to standardize the look of forms in the Portal. An alternative to the default positioning is to manually author the HTML for the form by binding each control into an HTML template for the blade.

This sample includes two commands at the top of the blade that will save or discard data. Commands are used because this blade stays open after a save/discard operation. If the blade were to close after the operation, then there would be an action bar at the bottom of the blade to use instead. 

The commands check to make sure that the `EditScope` has been populated previous to enabling themselves via the `canExecute` portion of the `ko.pureComputed` code.

The commands also keep themselves disabled during save operations via a `_saving` observable that the blade maintains, as in the following code.

`<dir>\Client\V1\Forms\Samples\Basic\ViewModels\FormsSampleBasicBlade.ts`

<!--
gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/Forms/Samples/Basic/ViewModels/FormsSampleBasicBlade.ts", "section": "forms#commands"}
-->

Because the `EditScope` is being used, the save/discard commands can just call the `saveChanges()` or `revertAll()` methods on the edit scope to trigger the right action.

For more information, see [http://knockoutjs.com/documentation/computed-writable.html](http://knockoutjs.com/documentation/computed-writable.html).



<a name="legacy-edit-scopes-request-an-editscope-from-pdl"></a>
### Request an editscope from pdl

The following sample code demonstrates requesting an `editScope` from PDL.  The sample is also located at `<dir>\Client\V1\MasterDetail\MasterDetailEdit\MasterDetailEdit.pdl`.  The `valid` element is using the `section` object of the form to determine if the form is currently valid.  This is not directly related to the edit scope, but will be relevant in the section named []().

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

<a name="legacy-edit-scopes-the-editscopeview-object"></a>
### The editScopeView object

The data in the `editScope` includes original values and saved edits. The method to access inputs on a part is the `onInputsSet` method. In the constructor, a new `MsPortalFx.Data.EditScopeView` object is created from the `dataContext`. The `EditScopeView` provides a stable observable reference to an `EditScope` object. The `editScopeId` will be sent in as a member of the `inputs` object when the part is bound.

An example of loading an edit scope is in the following code.  The sample is also located at `<dir>\Client\V1\MasterDetail\MasterDetailEdit\ViewModels\DetailViewModels.ts`. 

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

In the following example, the `editScopeView` is refreshed with new data from the data context.

```ts
// update the editScopeView with a new id
public onInputsSet(inputs: any): MsPortalFx.Base.Promise {
    // Acquires edit scope seeded with an item with id currentItemId.
    return this._editScopeView.fetchForExistingData(inputs.editScopeId, inputs.currentItemId);
}
```

<!--TODO: Determine which piece of code is associated with this paragraph.  -->

 The code that loads the `EditScope` is largely related to data loading, so the data context is the preferred location for the code.

<a name="legacy-edit-scopes-load-an-edit-scope-from-a-data-context"></a>
### Load an edit scope from a data context

 <!--TODO:  The following sample code does not exist by this name.  -->
 For an example of loading an edit scope from a data context, view the sample that is located at `<dir>\Client\Data\MasterDetailEdit\MasterDetailEditData.ts`.
 <!--TODO:  The previous sample code does not exist by this name.  -->

The following code creates a new `EditScopeCache`, which is bound to the `DataModels.WebsiteModel` object type. It uses an `editScope` within a form.  The `fetchForExistingData()` method on the cache provides a promise, which informs the `ViewModel` that the `EditScope` is loaded and available.

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

<a name="legacy-edit-scopes-the-editscopeaccessor"></a>
### The editScopeAccessor

Form fields require a binding to one or more `EditScope` observables. Consequently, they have two constructor overloads.  Extension developers can configure this binding by supplying a path from the root of the EditScope/Form model down to the observable to which the form field should bind. They can do this by selecting one of the two form field constructor variations. 

**NOTE**: In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>`  is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK.

1. **EditScopeAccessor** This is the preferred, compile-time verified methodology. The form field ViewModel constructor accepts an EditScopeAccessor, wraps a compile-time verified lambda, and returns the EditScope observable to which the Form field should bind, as in the following code located at     `<dir>/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts`.  It is also in the following code.

    gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts", "section": "formsEditScopeFaq#editScopeAccessor"} 
    The EditScopeAccessor methodology is preferred for the following reasons.

    * The supplied lambda will be compile-time verified. This code is more maintainable, for example, when the property names on the Form model types are changed.
    * There are advanced variations of `EditScopeAccessor` that enable less-common scenarios like binding multiple `EditScope` observables to a single form field.  There are others that demonstrate translating form model data for presentation to the user, as in the code located at       `<dir>/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts`. It is also in the following code.
  
    gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts", "section": "formsEditScopeFaq#editScopeAccessorAdvanced"}

1. **String-typed path** This methodology is discouraged because it is not compile-time verified. The form field ViewModel constructor accepts a string-typed path that contains the location of the EditScope observable to which the Form field should bind, as in the code located at    `<dir>/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts`. It is also in the following code.

  gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/Forms/Scenarios/FormFields/ViewModels/FormFieldsFormIntegratedViewModels.ts", "section": "formsEditScopeFaq#editScopePath"} 

* * *

<a name="legacy-edit-scopes-"></a>
### 





The type is marked as an "entity type". 

These factors made EditScope `EntityArrays` behave differently than regular JavaScript arrays, as follows.

* `Creates` are kept out-of-band

* `Deletes` are non-destructive

The property/properties that constitute the entity's 'id' are specified in the following examples. 

**NOTE**: In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory, and  `<dirParent>`  is the `SamplesExtension\` directory, based on where the samples were installed when the developer set up the SDK.
 
* In TypeScript:

    The TypeScript sample is located at 
    `<dir>\Client\V1\Forms\Scenarios\ChangeTracking\Models\EditableFormData.ts`. This code is also included in the following working copy.

    gitdown": "include-section", "file":"../Samples/SamplesExtension/Extension/Client/V1/Forms/Scenarios/ChangeTracking/Models/EditableFormData.ts", "section": "formsEditScopeFaq#entityTypeMetadata"}

* In C#:

    The C# sample is located at 
    `<dirParent>\SamplesExtension.DataModels/Person.cs`. This code is also included in the following working copy.

    gitdown": "include-section", "file":"../Samples/SamplesExtension/SamplesExtension.DataModels/Person.cs", "section": "formsEditScopeFaq#entityTypeMetadataCsharp"}

The following properties significantly simplify working with `EditScope` entity arrays.

* [The trackEdits property](#the-trackedits-property)

* [The applyArrayAsEdits method](#the-applyarrayasedits-method)

<a name="legacy-edit-scopes--the-trackedits-property"></a>
#### The trackEdits property

 Some properties on the EditScope/Form model are only for presentation instead of for editing. In this situation, the extension can instruct `EditScope` to opt out of tracking user edits for the specified properties.

In TypeScript:  

    MsPortalFx.Data.Metadata.setTypeMetadata("Employee", {
        properties: {
            accruedVacationDays: { trackEdits: false },
            ...
        },
        ...
    });  

In C#:  

    [TypeMetadataModel(typeof(Employee))]
    public class Employee
    {
        [TrackEdits(false)]
        public int AccruedVacationDays { get; set; }

        ...
    }  

Extensions can supply type metadata to configure their `EditScope` as follows.

* When using `ParameterProvider`, supply the `editScopeMetadataType` option to the `ParameterProvider` constructor.

* When using `EditScopeCache`, supply the `entityTypeName` option to `MsPortalFx.Data.EditScopeCache.createNew`.

Extensions can pass the type name used to register type metadata to either of these options by using `MsPortalFx.Data.Metadata.setTypeMetadata`.

<a name="legacy-edit-scopes--the-getcreated-addcreated-method"></a>
#### The getCreated/addCreated method

The `getCreated/addCreated` APIs allow for the addition of new, 'created' entity objects. The `getCreated` method returns a distinct, out-of-band array that collects all 'created' entities corresponding to a given 'entity' array. The `addCreated` method is a helper method that places a new 'entity' object in this `getCreated` array.

<a name="legacy-edit-scopes--the-markfordelete-method"></a>
#### The markForDelete method

The `markForDelete` APIs allows an extension to delete an 'entity' from the `EditScope` as a non-destructive operation. This allows the extension to render 'deleted' edits with strike-through or similar styling. Calling this `markForDelete` method puts the associated 'entity' in a 'deleted' state, although the deletion is not yet saved.

<!-- TODO:  The following content seems to belong with editscopes instead of the form documents.  However, it is not properly formatted.  -->

<a name="legacy-edit-scopes-create-new-objects-and-bind-them-to-the-editscope"></a>
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

<a name="legacy-edit-scopes-q-what-should-be-returned-from-saveeditscopechanges-i-don-t-understand-the-different-values-of-the-accepteditscopechangesaction-enum"></a>
### Q: What should be returned from &#39;saveEditScopeChanges&#39;? I don&#39;t understand the different values of the <code>AcceptEditScopeChangesAction</code> enum.

<!-- TODO:  Move this to the EditScope document -->

SOLUTION: 
When creating an EditScopeCache, the `saveEditScopeChanges` callback supplied by the extension is called to push EditScope edits to a server. This callback returns a `Promise` that should be resolved when the 'save' AJAX call completes, which occurs after the server accepts the user's edits. 

When the extension resolves this `Promise`, it can supply a value that instructs the EditScope to  reset itself to a clean/unedited state. If no such value is returned during Promise resolution, then the EditScope is reset by default.  This means taking the user's client-side edits and considering these values to be the new, clean/unedited EditScope state. This works for many scenarios.  

There are scenarios where the default `saveEditScopeChanges` behavior does not work, like: the following. 
* During 'save', the server produces new data values that need to be merged into the EditScope.
* For "create new record" scenarios, after 'save', the extension should clear the form, so that the user can enter a new record.

For these cases, the extension will resolve the `saveEditScopeChanges` Promise with a value from the `AcceptEditScopeChangesAction` enum. These values allow the extension to specify conditions like the following.
* The EditScopeCache is to implicitly reload the EditScope data as part of completing the 'save' operation.
* The EditScope's data is to be reverted/cleared as part of completing the 'save' operation (the "create new record" UX scenario).

For more information about each enum value, see the jsdoc comments around `MsPortalFx.Data.AcceptEditScopeChangesAction` or in `MsPortalFxDocs.js` in Visual Studio or any code editor.

* * *