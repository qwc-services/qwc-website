# Editing

QWC2 offers comprehensive editing support through a variety of plugins:

- The [`Editing`](../references/qwc2_plugins.md#editing) plugin allows creating, editing and removing features of an editable vector layer. It supports editing both geometry and attributes, displaying customizeable attribute forms.
- The [`AttributeTable`](../references/qwc2_plugins.md#attributetable) plugin also allows creating, editing and removing features of an editable vector layer. It displays all features of the editable layer in a tabularized view, and allows editing attributes, but not geometries. Noteably, it will allow editing geometryless datasets.
- The [`FeatureForm`](../references/qwc2_plugins.md#featureform) works similarly to the feature-info, but will display the feature form according to the QGIS form configuration, and also allows editing the attributes and geometry of a picked feature. It can configured as `identifyTool` instead of the standard `Identify` plugin in `config.json`.


## Quick start <a name="quick-start"></a>

The easiest way to use the editing functionality is by using the pre-configured `qwc-docker` with the [`qwc-data-service`](https://github.com/qwc-services/qwc-data-service) and [`qwc-config-generator`](https://github.com/qwc-services/qwc-config-generator).

To make a layer editable, follow these steps:

* The datasource of the layer needs to be a PostGIS database. In particular, make sure that a primary key is configured for your dataset!
* Configure the QGIS PostgreSQL connection using a service name, add the corresponding service definition to your host `pg_service.conf` and to `qwc-docker/pg_service-write.conf`. Make sure your database host is reachable within the docker containers!
* Especially when your primary key field type is `serial`, you'll want to mark the corresponding field widget type as `Hidden` in the QGIS Attributes Form settings.
* Create a `Data` resource as child of the corresponding `Map` resource in the administration backend, and create a new permission for the `Map` and `Data` resources for the roles which should be allowed to edit the layer.
* *Note:* if you leave the "Write" checkbox in the `Data` resource permission unchecked, the dataset will be available as read-only, which can be useful if you want to use the `AttributeTable` and/or `FeatureForm` to just display the dataset without allowing any edits.
* Run the config generator from the administration backend to update service configuration.

## Designing the edit forms<a name="edit-forms"></a>

Much of the power of the QWC2 editing functionality resides in its fully customizeable forms, providing support for different input widget types, file uploads and 1:N relations.

The [`qwc-config-generator`](https://github.com/qwc-services/qwc-config-generator) will automatically generate forms based on the configuration specified in the QGIS Layer Properties &rarr; Attributes Form. If `Autogenerate` or `Drag and Drop Designer` is chosen, a corresponding Qt UI form is automatically generated for QWC2 in `assets/forms/autogen`. If `Provide ui-file` is chosen, the specified UI form will copied to `assets/forms/autogen`.

Localized translation forms are supported. To this end, place a Qt Translation file called `<form_basename>_<lang>.ts` next to the designer form `<form_basename>.ui`, where `lang` is a language or language/country code, i.e. `en` or `en-US`. There is a [`translateui.sh`](https://github.com/qgis/qwc2/tree/master/scripts/translateui.sh) script to help generate the translation files. Example:
```bash
./translateui.sh .../qwc2/assets/forms/form.ui de it fr
```
### File uploads

You can configure a *text-like* field of your dataset as an upload field as follows:

- For `Autogenerated` and `Drag and Drop Designer` forms configuration, set the widget type to Attachment. You can set the file type filter in the widget configuration under `Display button to open file dialog -> Filter`, in the format `*.ext1, *.ext2`.
- For manually created Qt Designed Ui forms, use a `QLineEdit` widget named `<fieldname>__upload`, and optionally as the text value of the `QLineEdit` set a comma separated list of suggested file extensions.

Attachments are stored on disk below the `attachments_base_dir` defined in the data service configuration, and the path to the attachments stored in the dataset.

*Note:*

- If you set the format constraint to `*.jpeg` and your browser has access to a camera, QWC2 will allow you to directly upload images captured from the camera.
- You can set the allowed attachment extensions and maximum file sizes globally by setting `allowed_attachment_extensions` and `max_attachment_file_size` in the data service configuration. You may also need to set/increase `client_max_body_size` in `qwc-docker/api-gateway/nginx.conf`.
- You can also set the allowed attachment extensions and maximum file sizes per dataset by setting `max_attachment_file_size_per_dataset` and `allowed_extensions_per_dataset` in the data service configuration. If you set the per dataset values, the global settings will be disregarded (i.e. if an attachment satisfies the per dataset constraint it will be considered valid, even if it violates the global constraint).
- To ensure the uploaded files are properly rendered as download links in GetFeatureInfo responses, use the [`qwc-feature-info-service`](https://github.com/qwc-services/qwc-config-generator).

### Key-value relations (value mappings)

Value relations allow mapping technical values to a human readable display strings, displayed in a combo box.

For `Autogenerated` and `Drag and Drop Designer`, use widgets of type `Value Relation`.

In a manually created Qt-Designer Ui form, you can use key-value relations for combo box entries by naming the `QComboBox` widget according to the following pattern: `kvrel__<fieldname>__<kvtablename>__<kvtable_valuefield>__<kvtable_labelfield>`. `<kvtablename>` refers to a table containing a field called `<kvtable_valuefield>` for the value of the entry and a field `<kvtable_labelfield>` for the label of the entry. For key-value relations inside a 1:N relation, use `kvrel__<reltablename>__<fieldname>__<kvtablename>__<kvtable_valuefield>__<kvtable_labelfield>`.

*Note:* The relation table needs to be added as a (geometryless) table to the QGIS Project. You also need to set appropriate permissions for the relation table dataset in the QWC admin backend. Alternatively, you can set `autogen_keyvaltable_datasets`to `true` in the config generator configuration, to automatically generate resources and read-only permissions as required.


### 1:N relations

1:N relations allow associating multiple child records to the target feature, displayed in a table.

For `Autogenerated` and `Drag and Drop Designer` forms, configure the 1:N relation in QGIS Project &rarr; Properties &rarr; Relations. Note that the child table foreign key must refer to parent primary key.

By default, a table widget similar to an attribute table will be generated to manage the relation values. If you set `generate_nested_nrel_forms` to `true` in the config generator config, the relation values will be displayed as a list of buttons which open the record in a nested form. The button label is chosen according to the following rules:

* The display name (Layer properties &rarr; Display &rarr; Display Name) of the referencing layer, if the expression is a single field name.
* The primary key value of the referencing layer.


In a manually created Qt-Designer Ui form, create a widget of type `QWidget`, `QGroupBox` or `QFrame` named according to the pattern `nrel__<reltablename>__<foreignkeyfield>`, where `<reltablename>` is the name of the relation table and `<foreignkeyfield>` the name of the foreign key field in the relation table. Inside this widget, add the edit widgets for the values of the relation table. Name the widgets `<reltablename>__<fieldname>`. These edit widgets will be replicated for each relation record.

*Notes*:

- In a manually created Qt-Designer Ui form, you can also specify a sort column for the 1:N relation in the form `nrel__<reltablename>__<foreignkeyfield>__<sortcol>`. If a sort-column is specified, QWC2 will display sort arrows for each entry in the relation widget.
- The relation table needs to be added as a (geometryless) table to the QGIS Project. You also need to set appropriate permissions for the relation table dataset in the QWC admin backend.


### Special form widgets

In manually created Qt-Designer Ui forms, there are a number of special widgets you can use:

* *Images*: To display attribute values which contain an image URL as an inline image, use a `QLabel` named `img__<fieldname>`.
* *Linked features*: To display a button to choose a linked feature and edit its attributes in a nested edit form, create a `QPushButton` named `featurelink__<linkdataset>__<fieldname>` (simple join) or `featurelink__<linkdataset>__<reltable>__<fieldname>` in a 1:N relation. In a 1:N relation, `linkdataset` can be equal to `reltable` to edit the relation record itself in the nested form. `fieldname` will contain the `id` of the linked feature.
* *External fields*: Some times it is useful to display information from an external source in the edit form. This can be accomplished by creating a `QWidget` with name `ext__<fieldname>` and using a form preprocessor hook (see `registerFormPreprocessor` in [`QtDesignerForm.jsx`](https://github.com/qgis/qwc2/blob/master/components/QtDesignerForm.jsx) to populate the field by assigning a React fragment to `formData.externalFields.<fieldname>`.
* *Buttons*: To add a button with a custom action, add a `QPushButton` with name `btn__<buttonname>`, and use a form preprocessor hook to set the custom function to `formData.buttons.buttonname.onClick`.


## Logging mutations

The `qwc-data-service` offers some basic functionality for logging mutations:

- If you set `upload_user_field_suffix` in the data service config, the username of the last user who performed a mutation to `<fieldname>` will be logged to `<fieldname>__<upload_user_field_suffix>`.
- If you set `edit_user_field` in the data service config, the username of the last user who performed a mutation to a record with be logged to the `<edit_user_field>` field of the record.
- If you set `edit_timestamp_field` in the data service config, the timestamp of the last mutation to a record will be logged to the `<edit_timestamp_field>` field of the record.

*Note*: for these fields to be written, ensure the qgis project is also up-to-date, i.e. that contain the up-to-date table schemas. You can set the respective field types to hidden in the QGIS layer properties to avoid them showing up in the autogenerated edit forms.


## Using a custom editing backend

You can also use a custom editing backend instead of the `qwc-data-service` by following these steps:

- Implement the custom editing interface, taking [default `EditingInterface.js`](https://github.com/qgis/qwc2/blob/master/utils/EditingInterface.js) as a template.
- Enable the desired editing plugins in [`js/appConfig.js`](https://github.com/qgis/qwc2-demo-app/blob/master/js/appConfig.js), passing your custom editing interface to `Editing`, `AttributeTable` and `FeatureForm`.
- Set up an editing backend.
- If you are using the [`qwc-config-generator`](https://github.com/qwc-services/qwc-config-generator), the edit configuration will be automatically generated from the QGIS project. Otherwise, you need to write a custom `editConfig` in `themesConfig.json` as follows:

| Entry                                | Description                                                   |
|--------------------------------------|---------------------------------------------------------------|
| `{`                                  |                                                               |
| `⁣  <LayerId>: {`                     | A WMS layer ID. Should be a theme WMS layer name, to ensure the WMS is correctly refreshed. |
| `⁣    "layerName": "<LayerName>",`    | The layer name to show in the selection combo box. |
| `⁣    "editDataset": "<DatasetName>",`| The name of the edit dataset passed to the editing interface. |
| `⁣    "geomType": "<GeomType>",`      | The geometry type, either `Point`, `LineString` or `Polygon`. |
| `⁣    "displayField": "<FieldId>",`  | The ID of the field to use in the feature selection menu.     |
| `⁣    "permissions": {`               | A list of different write permissions to specify rights and buttons. |
| `⁣      "creatable": <boolean>,`      | If `true`, `Draw` button will appear in Editing interface and `Add` button in Attribute Table. |
| `⁣      "updatable": <boolean>,`      | If `true`, `Pick` button will appear in Editing interface.    |
| `⁣      "deletable": <boolean>,`      | If `true`, `Delete` button will appear in Editing interface and Attribute Table. |
| `⁣    },`                           |                                                               |
| `⁣    "fields": [{`                   | A list of field definitions, for each exposed attribute.      |
| `⁣      "id": "<FieldID>",`           | The field ID.                                                 |
| `⁣      "name": "<FieldName>",`       | The field name, as displayed in the editing form.             |
| `⁣      "type": "<FieldType>",`       | A field type. Either `bool`, `list` or a regular [HTML input element type](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input). |
| `⁣      "constraints": {`             | Constraints for the input field.                              |
| `⁣        "values": [<Entries>],`     | Only if `type` is `list`: an array of arbitrary strings.      |
| `⁣        ...`                        | For regular HTML input types, the ReactJS API name of any applicable [HTML input constraint](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input), i.e. `maxLength` or `readOnly`. |
| `⁣      }`                            |                                                               |
| `⁣    }],`                            |                                                               |
| `⁣    "form": "<PathToUiFile>",`      | Optional, a QtDesigner UI file.                               |
| `⁣  }`                                |                                                               |
| `}`                                  |                                                               |

* If you specify just `fields`, a simple form is autogenerated based on the field definitions.
* Alternatively you can specify the URL to a Qt Designer UI form in `form` (use `:/<path>` to specify a path below the `assets` folder).