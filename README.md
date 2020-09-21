# xsd_to_django_model
Generate Django models from an XSD schema description (and a bunch of hints)

## TODO
* More examples.
* More heuristics.
* `xs:complexType/(xs:simpleContent|xs:complexContent)/xs:restriction` support.
* `xs:simpleType/xs:union` support.
* ...?

## Getting started

```
Usage:
    xsd_to_django_model.py [-m <models_filename>] [-f <fields_filename>] [-j <mapping_filename>] <xsd_filename> <xsd_type>...
    xsd_to_django_model.py -h | --help

Options:
    -h --help              Show this screen.
    -m <models_filename>   Output models filename [default: models.py].
    -f <fields_filename>   Output fields filename [default: fields.py].
    -j <mapping_filename>  Output JSON mapping filename [default: mapping.json].
    <xsd_filename>         Input XSD schema filename.
    <xsd_type>             XSD type (or an XPath query for XSD type) for which a Django model should be generated.

If you have xsd_to_django_model_settings.py in your PYTHONPATH or in the current directory, it will be imported.
```

## Examples

See the `examples` subdirectory.

## Settings

If you have `xsd_to_django_model_settings.py` in your `PYTHONPATH` or in the current directory, it will be imported.
It may define the following module-level variables:

* `TYPE_MODEL_MAP` is a `collections.OrderedDict` (you can use `dict` but then processing order is not guaranteed) which maps XSD `xs:complexType` names (including ones autogenerated for anonymous `xs:complexType`s) to Django model class names, using regular expressions, e.g., if the XSD types are all called like `tMyStrangeTasteType`, this will generate model names like `MyStrangeTaste`:
  ```python
  from collections import OrderedDict
  
  TYPE_MODEL_MAP = OrderedDict([
      (r't([A-Z].*)Type', r'\1'),
  ])
  ```
  If you are mapping different XSD types to a single Django model, prefix the Django model's name with `+` to make sure model merging logic works well.

* `MODEL_OPTIONS` is a `dict` mapping model names to `dict`s of options applied when processing this model:
  ```python
  MODEL_OPTIONS = {
      'MyStrangeTaste': {
          'abstract': True,
      },
  }
  ```
  Supported model options:
  * `abstract` - when `True`, enforce `abstract = True` in Django model`s `Meta`.
  * `add_fields` - a `list` of extra fields which should be included in the final Django model class. Most of the options should be set in these definitions, overrides won't work. Example:
    ```python
    'add_fields': [
        {
            'name': 'content_md5',                # Django model field name
            'dotted_name': 'a.any',               # path to the field in XML
            'django_field': 'models.UUIDField',   # Django field class
            'doc': 'A hash to speed up lookups',  # Django field's verbose_name
            'options': ['db_index=True'],         # Django field's options
        },
    ],
    ```
  * `add_json_attrs` - a mapping of extra JSON attributes to their documentation.
  * `array_fields` - a `list` of XSD fields for which `django.contrip.postgres.fields.ArrayField`s should be generated. The corresponding `xs:element` should either have `maxOccurs="unbounded"` or contain an `xs:complexType` whose only member has `maxOccurs="unbounded"`.
  * `coalesce_fields` - a `dict` mapping generated field names to actual field names in Django models, using regular expressions, e.g.:
    ```python
    'coalesce_fields': {
        r'a_very_long_autogenerated_prefix_(.+)': r'myprefix_\1',
    },
    ```
  * `custom` - when `True`, treat the model as a custom model which does not have a corresponding `xs:complexType` in XSD schema. Makes sense to use at least `add_fields` as well in such case.
  * `drop_fields` - a `list` of field names to be ignored when encountered, e.g.:
    ```python
    'drop_fields': ['digitalSignature2KBInSize'],
    ```
  * `field_docs` - a `dict` which allows to override `verbose_name` Django model field option for given fields:
    ```python
    'field_docs': {
        'objectName': 'Fully qualified object name',
    }
    ```
  * `field_options` - a `dict` mapping final Django model field names to a list of their overridden options, e.g.:
    ```python
    'field_options': {
        'schemeVersion': ['max_length=7'],
    }
    ```
  * `flatten_fields` - a `list` of fields where the contained `xs:complexType` should not be treated as a separate Django model, but is to be merged in the current model, with member field names prefixed with this field's name and a `_`:
    ```python
    'flatten_fields': ['section1', 'section2'],
    ```
  * `flatten_prefixes` - a `list` of field name prefixes which trigger the same flattening logic as `flatten_fields`. I do not recommend to use this option as things get too much automatic with it.
  * `foreign_key_overrides` - a `dict` mapping field names to XSD type names when this field is to be treated as a `ForeignKey` reference to that type`s corresponding model.
  * `gin_index_fields` - a `list` of Django model fields for which indexes are defined in `Meta` using Django 1.11+ `indexes` and `django.contrib.postgres.indexes.GinIndex`.
  * `if_type` - a `dict` mapping XSD type names to `dict`s of options which are used only when source XSD type name matches that type name; used by model merging logic to ensure merged fields do not differ in models being merged:
    ```python
    'if_type': {
        'tMyStrangeTasteType': {
            'abstract': True,
        },
    },
    ```
  * `ignore_merge_mismatch_fields`: a `list` of fields for which generated code differences are to be ignored when model merging takes place. I do not recommend to use this option as it gets harder to track changes over time.
  * `include_parent_fields`: when `True`, the parent (`xs:complexType/xs:complexContent/xs:extension[@base]` type's fields are simply included in the current model instead of making Django model inheritance structures.
  * `index_fields` - a `list` of Django model fields for which `db_index=True` option should be generated.
  * `json_fields` - a `list` of XSD fields which do not get their own Django model fields but are all mapped to a single `attrs = django.contrib.postgres.fields.JSONField()`.
  * `many_to_many_field_overrides` - a `dict` mapping field names to XSD type names when this field is to be treated as a `ManyToManyField` reference to that type`s corresponding model.
  * `many_to_many_fields` - a `list` of field names that get mapped to Django `ManyToManyField`s.
  * `meta` - a `list` of Django model `Meta` options' string templates added to the generated model's `Meta`:
    ```python
    'meta': [
        'db_table = "my_%(model_lower)s"',
    ]
    ```
    Template variables:
    * `model_lower` resolves to lowercased model name.
  * `methods` - a list of strings that are included as methods in the generated model class, e.g.:
    ```python
    'methods': [
        '    def __unicode__(self): return "%s: %s" % (self.code, self.amount)',
    ],
    ```
  * `null_fields` - a `list` of field names for which `null=True` Django model field option should be enforced.
  * `one_to_many_field_overrides` - a `dict` mapping one-to-many relationship field names to related Django model names when automated logic does not work well.
  * `one_to_many_fields` - a `list` of field names that get mapped to one-to-many relationships in Django, that is, a `ForeignKey` to this model is added to the related model's fields.
  * `override_field_class` - a `dict` mapping field names to corresponding Django field class names when automated logic does not work well, e.g.:
    ```python
    'override_field_class': {
        'formattedNumber': 'models.TextField',  # The actual data does not meet maximum length restriction
    }
    ```
  * `parent_type` - an XSD `xs:complexType` name which is to be used as this model's parent type when automated logic does not work, e.g., when `custom` is `True`.
  * `plain_index_fields` - a `list` of Django model fields for which indexes are defined in `Meta` using Django 1.11+ `indexes` and `models.Index`. This is useful when you don't need extra index for `LIKE` queries autogenerated with `db_index=True` option.
  * `primary_key` specifies a single field name for which `primary_key=True` Django model field option is to be generated.
  * `reference_extension_fields` - a `list` of field names which use `xs:complexType/xs:complexContent/xs:extension' inheritance schema but extend the base type with extra members; these fields will be mapped to a `ForeignKey` to their base type's model, and the extension members will be flattened (as in `flatten_fields`).
  * `skip_code` - if `True`, the generated model code is to be skipped when saving the models module.
  * `strict_index_fields` - a `list` of Django model fields to generate a composite index including the primary key.
  * `unique_fields` - a `list` of Django model fields for which `unique=True` option should be generated.

* `GLOBAL_MODEL_OPTIONS` is a `dict` of model options applied to each model (but may be overridden by the respective options in that very model):
  ```python
  GLOBAL_MODEL_OPTIONS = {
      'json_fields': ['attachments', 'abracatabra'],
  }
  ```
  Additional global options:
  * `charfield_max_length_factor` (defaults to `1`) - a factor to multiply every `CharField`s `max_length` by, e.g. when actual XML data is bigger than XSD dictates.

* `TYPE_OVERRIDES` is a `dict` mapping XSD type names to Django model fields when automatic heuristics don't work, e.g.:
  ```python
  TYPE_OVERRIDES = {
      'xs:IDREF': ('A reference to xs:ID', 'CharField', {'max_length': 64}),
  }
  ```

* `BASETYPE_OVERRIDES` is a `dict` mapping XSD base type names to Django field to override default ones, e.g.:
  ```python
  BASETYPE_OVERRIDES = {
      'xs:hexBinary': 'CharField',
  }
  ```

* `IMPORTS` is a string inserted after the automatically generated `import`s in the output models file, e.g.:
   ```python
   IMPORTS = "from mycooldjangofields import StarTopologyField"
   ```

* `DOC_PREPROCESSOR` is a function to preprocess documentation strings, e.g.:
   ```python
   def doc_preprocessor(s):
       return s.replace('\n', ' ')

   DOC_PREPROCESSOR = doc_preprocessor
   ```

* `JSON_DOC_HEADING` is documentation heading template for `JsonField`s.

* `JSON_GROUP_HEADING` is documentation attribute group heading template for `JsonField`s.

* `JSON_DOC_INDENT` is indent prefix in documentation sublists. The default is 4 spaces, which is compatible between Markdown and reStructuredText.
