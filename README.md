# XML dataclasses

This is a very rough prototype of how a library might look like for (de)serialising XML into  Python dataclasses. XML dataclasses build on normal dataclasses from the standard library and [`lxml`](https://pypi.org/project/lxml/) elements. Loading and saving these elements is left to the consumer for flexibility of the desired output.

It isn't ready for production if you aren't willing to do your own evaluation/quality assurance. I don't recommend using this library with untrusted content. It inherits all of `lxml`'s flaws with regards to XML attacks, and recursively resolves data structures. Because deserialisation is driven from the dataclass definitions, it shouldn't be possible to execute arbitrary Python code. But denial of service attacks would very likely be feasible.

Requires Python 3.7 or higher.

## Example

(This is a simplified real world example - the container can also include optional `links` child elements.)

```xml
<?xml version="1.0"?>
<container version="1.0" xmlns="urn:oasis:names:tc:opendocument:xmlns:container">
  <rootfiles>
    <rootfile full-path="OEBPS/content.opf" media-type="application/oebps-package+xml" />
  </rootfiles>
</container>
```

```python
from lxml import etree
from typing import List
from xml_dataclasses import xml_dataclass, rename, load, dump

CONTAINER_NS = "urn:oasis:names:tc:opendocument:xmlns:container"

@xml_dataclass
class RootFile:
    __ns__ = CONTAINER_NS
    full_path: str = rename(name="full-path")
    media_type: str = rename(name="media-type")


@xml_dataclass
class RootFiles:
    __ns__ = CONTAINER_NS
    rootfile: List[RootFile]


@xml_dataclass
class Container:
    __ns__ = CONTAINER_NS
    version: str
    rootfiles: RootFiles
    # WARNING: this is an incomplete implementation of an OPF container


if __name__ == "__main__":
    nsmap = {None: CONTAINER_NS}
    lxml_el_in = etree.parse("container.xml").getroot()
    container = load(Container, lxml_el_in, "container")
    lxml_el_out = dump(container, "container", nsmap)
    print(etree.tostring(lxml_el_out, encoding="unicode", pretty_print=True))
```

## Features

* XML dataclasses are also dataclasses, and only require a single decorator
* Convert XML documents to well-defined dataclasses, which should work with IDE auto-completion
* Loading and dumping of attributes, child elements, and text content
* Required and optional attributes and child elements
* Lists of child elements are supported, as are unions and lists or unions
* Inheritance does work, but has the same limitations as dataclasses. Inheriting from base classes with required fields and declaring optional fields doesn't work due to field order. This isn't recommended
* Namespace support is decent as long as correctly declared. I've tried on several real-world examples, although they were known to be valid. `lxml` does a great job at expanding namespace information when loading and simplifying it when saving

## Patterns

### Defining attributes

Attributes can be either `str` or `Optional[str]`. Using any other type won't work. Attributes can be renamed or have their namespace modified via the `rename` function. It can be used either on its own, or with an existing field definition:

```python
@xml_dataclass
class Foo:
    __ns__ = None
    required: str
    optional: Optional[str] = None
    renamed_with_default: str = rename(default=None, name="renamed-with-default")
    namespaced: str = rename(ns="http://www.w3.org/XML/1998/namespace")
    existing_field: str = rename(field(...), name="existing-field")
```

I would like to add support for validation in future, which might also make it easier to support other types. For now, you can work around this limitation with properties that do the conversion.

### Defining text

Like attributes, text can be either `str` or `Optional[str]`. You must declare text content with the `text` function. Similar to `rename`, this function can use an existing field definition, or take the `default` argument. Text cannot be renamed or namespaced. Every class can only have one field defining text content. If a class has text content, it cannot have any children.

```python
@xml_dataclass
class Foo:
    __ns__ = None
    value: str = text()

@xml_dataclass
class Foo:
    __ns__ = None
    content: Optional[str] = text(default=None)

@xml_dataclass
class Foo:
    __ns__ = None
    uuid: str = text(field(default_factory=lambda: str(uuid4())))
```

### Defining children/child elements

Children must ultimately be other XML dataclasses. However, they can also be `Optional`, `List`, and `Union` types:

* `Optional` must be at the top level. Valid: `Optional[List[XmlDataclass]]`. Invalid: `List[Optional[XmlDataclass]]`
* Next, `List` should be defined (if multiple child elements are allowed). Valid: `List[Union[XmlDataclass1, XmlDataclass2]]`. Invalid: `Union[List[XmlDataclass1], XmlDataclass2]`
* Finally, if `Optional` or `List` were used, a union type should be the inner-most (again, if needed)

Children can be renamed via the `rename` function, however attempting to set a namespace is invalid, since the namespace is provided by the child type's XML dataclass. Also, unions of XML dataclasses must have the same namespace (you can use different fields if they have different namespaces).

If a class has children, it cannot have text content.

## Gotchas

### Whitespace

If you are able to, it is strongly recommended you strip whitespace from the input via `lxml`:

```python
parser = etree.XMLParser(remove_blank_text=True)
```

By default, `lxml` preserves whitespace. This can cause a problem when checking if elements have no text. The library does attempt to strip these; literally via Python's `strip()`. But `lxml` is likely faster and more robust.

### Optional vs required

On dataclasses, optional fields also usually have a default value to be useful. But this isn't required; `Optional` is just a type hint to say `None` is allowed.

For XML dataclasses, on loading/deserialisation, whether or not a field is required is determined by if it has a `default`/`default_factory` defined. If so, and it's missing, that default is used. Otherwise, an error is raised.

For dumping/serialisation, the default isn't considered. Instead, if a value is marked as `Optional` and the value is `None`, it isn't written.

This makes sense in many cases, but possibly not every case.

### Other limitations and Assumptions

Most of these limitations/assumptions are enforced. They may make this project unsuitable for your use-case.

* It isn't possible to pass any parameters to the wrapped `@dataclass` decorator
* Setting the `init` parameter of a dataclass' `field` will lead to bad things happening, this isn't supported
* Deserialisation is strict; missing required attributes and child elements will cause an error. I want this to be the default behaviour, but it should be straightforward to add a parameter to `load` for lenient operation
* Dataclasses must be written by hand, no tools are provided to generate these from, DTDs, XML schema definitions, or RELAX NG schemas

## Development

This project uses [pre-commit](https://pre-commit.com/) to run some linting hooks when committing. When you first clone the repo, please run:

```
pre-commit install
```

You may also run the hooks at any time:

```
pre-commit run --all-files
```

Dependencies are managed via [poetry](https://python-poetry.org/). To install all dependencies, use:

```
poetry install
```

This will also install development dependencies such as `black`, `isort`, `pylint`, `mypy`, and `pytest`. I've provided a simple script to run these during development called `lint`. You can either run it from a shell session with the poetry-installed virtual environment, or run as follows:

```
poetry run ./lint
```

Auto-formatters will be applied, and static analysis/tests are run in order. The script stops on failure to allow quick iteration.

## License

This library is licensed under the Mozilla Public License Version 2.0. For more information, see `LICENSE`.
