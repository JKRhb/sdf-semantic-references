---
v: 3
coding: utf-8

title: >
  Semantic Definition Format (SDF): Semantic Enhancements for sdfRef
abbrev: Semantic SDF References
docname: draft-romann-asdf-semantic-sdfref-latest

category: std
submissiontype: IETF
consensus: true

area: "Applications and Real-Time"
workgroup: "A Semantic Definition Format for Data and Interactions of Things"
keyword: Internet-Draft

venue:
  group: "A Semantic Definition Format for Data and Interactions of Things"
  mail: "asdf@ietf.org"
  github: "JKRhb/sdf-semantic-references"

author:
  - name: Jan Romann
    org: Universität Bremen
    email: jan.romann@uni-bremen.de

normative:
  RFC9880: sdf
  RFC9535: json-path

informative:

...

--- abstract


[^intro-]

[^intro-]:
    The Semantic Definition Format (SDF) for Data and Interactions of Things allows for referencing and importing definitions from other documents via the `sdfRef` quality.
    In the base SDF specification {{-sdf}}, these references are only structural in nature and do not consider semantic aspects such as the version of the referenced model.

    In this memo, we are proposing an extension for SDF's namespace concept that makes it possible to take into account semantic aspects when using `sdfRef`, but also in other scenarios when referring to external definitions.


--- middle

# Introduction

<!-- Just copying the abstract, for now... -->

[^intro-]

# Conventions and Definitions

The definitions of {{-sdf}} and {{-json-path}} apply.

<!-- TODO: Maybe find a better name for this term -->
SDF Repository:
: A web server hosting SDF models.
In accordance to {{-sdf}}, one or more SDF models may belong to the same namespace.
The SDF Repository is responsible to return the correct models that satisfy a SDF Consumer's request.
For this purpose, the SDF Repository might combine multiple individual models that belong to the same namespace into one.

<!-- TODO: Not sure if this term should be introduced. -->
SDF Consumer:
: A client that retrieves and processes SDF models (e.g., for translation purposes or to interact with a device that implements an SDF model).

{::boilerplate bcp14-tagged}

# Namespace Extensions

SDF's namespace concept maps shorthands such as `foo` to URIs that identify global names.
These URIs use the `https` scheme and a path.
Using JSON pointers as fragement identifiers, it is possible to refer to concrete definitions from other SDF models.

In base SDF, the query part of the URIs is deliberately left out and reserved for extensions.
This memo utilizes query parameters to enhance the namespacing concept with additional semantic capabilities, especially when it comes to referring to specific versions or version ranges of SDF models.

For this purpose, this memo introduces a set of standardized query parameters (see {{parameters-table}}) that fall into two categories:

1. A set of version-related parameters that allow for semantic references (`version`, `minVersion`, `maxVersion`, `exclusiveMinVersion`, `exclusiveMaxVersion`) and
2. the `subtree` parameter that restricts the part of the namespace that should be returned when deferencing it.

An SDF Repository SHOULD adhere to the version constraints indicated via the version-related parameters and return a model with either the exact or a compatible version number.
If the combination of version-related parameters is unsatisfiable (e.g., by specifying a `minVersion` of 1.2.0 and a `maxVersion` of 1.1.0), the SDF Repository SHOULD return an appropriate error response with an HTTP status code 400 (Bad Request).

The introduction of the `subtree` parameter is motivated by the fact that multiple models may contribute to the same SDF namespace (and fragment identifiers are not included in HTTP requests).
<!-- TODO: Check whether you can actually use JSON Path with HTTP query parameters -->
Using a JSON Path query with this paramter allows for giving the SDF Repository hosting the namespace definitions a hint on which parts of the target namespace should be included in the output.
When an SDF Consumer specifies a set of `sdfObject`s and/or `sdfThing`s in its request, the SDF Repository MAY leave out definitions that are not needed to fulfill the request.
Similarly, the SDF Repository MAY combine multiple models from the same namespace into one to satisfy the Consumer's request.

| Parameter           | Format                   | Description                                                                                                  |
| ------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------ |
| version             | any                      | Defines an exact model version that must match the retrieved model                                           |
| minVersion          | any                      | Defines a lower version bound that also includes this particular version.                                    |
| maxVersion          | any                      | Defines an upper version bound that also includes this particular version.                                   |
| exclusiveMinVersion | any                      | Defines a lower version bound that does not include this particular version.                                 |
| exclusiveMaxVersion | any                      | Defines an upper version bound that does not include this particular version.                                |
| subtree             | JSON Path {{-json-path}} | Indicates the subtreee of the referenced model that should be returned when dereferencing the namespace URI. |
{: #parameters-table title="Table with the query parameters for namespace URIs defined in this specification."}

# Examples

A simple example for a model that uses the `version` parameter is shown in {{code-simple-version-parameter-example}}.
In this example, the version of the referenced model for the `cap` namespace must match the value 1.1.0.
Updates that are applied to referenced model are not retrieved automatically.

~~~ sdf
namespace:
  cap: https://example.com/capability/cap?version=1.1.0
  lamps: https://example.com/lamps
defaultNamespace: lamps
sdfObject:
  LightSwitch:
    sdfAction:
      toggle:
        sdfRef: cap:/sdfObject/Switch/sdfAction/toggle
~~~
{:sdf #code-simple-version-parameter-example
title="Example SDF model of a light switch that refers to the version 1.1.0 of an external model."}

A second example that also takes model updates into account is shown in {{code-version-parameter-example}}.
Here, the parameters `minVersion` and `exlusiveMaxVersion` indicate that model version for the `cap` namespace should be greater than or equal to 1.1.0 but less than 1.2.0.
In many programming language ecosystems, this would correspond with the "caret notation" (`^`) that indicates that patch versions for dependencies should be retrieved automatically.
As not all SDF models will use semantic versioning in practice, this approach is more flexible and also covers version numbers that rely on dates, for example.

~~~ sdf
namespace:
  cap: https://example.com/capability/cap?minVersion=1.1.0&exclusiveMaxVersion=1.2.0
  lamps: https://example.com/lamps
defaultNamespace: lamps
sdfObject:
  LightSwitch:
    sdfAction:
      toggle:
        sdfRef: cap:/sdfObject/Switch/sdfAction/toggle
~~~
{:sdf #code-version-parameter-example
title="Example SDF model of a light switch that refers to a model version that is greater than or equal 1.1.0 and less than 1.2.0."}

The parametrization of URIs can also be used to use different version ranges for certain parts of a referenced namespace.
In the example shown in {{code-different-versions-parameter-example}}, version 2.0.0 of the models hosted under `https://example.com/capability/cap` has added a new status property that has not been part of previous versions.
At the same time, a non-backwards-compatible change has been applied to the referenced `toggle` action that should not be taken over yet.
To keep using the old `toggle` action while already incorporating the new `status` property, the namespacing concept allows referring to different versions via the new query parameters that have been introduced.

~~~ sdf
namespace:
  cap11: https://example.com/capability/cap?minVersion=1.1.0&exclusiveMaxVersion=1.2.0
  cap2: https://example.com/capability/cap?minVersion=2.0.0&exclusiveMaxVersion=2.1.0
  lamps: https://example.com/lamps
defaultNamespace: lamps
sdfObject:
  LightSwitch:
    sdfAction:
      toggle:
        sdfRef: cap11:/sdfObject/Switch/sdfAction/toggle
    sdfProperty:
      status:
        sdfRef: cap2:/sdfObject/Switch/sdfProperty/status
~~~
{:sdf #code-different-versions-parameter-example
title="Example SDF model of a light switch that refers to two different version ranges of models from the capability namespace."}

Lastly, since we are only using definitions from the `sdfObject` `Switch` in the referenced namespace, we can limit the subtree that we are referring to via the `subtree` parameter.
{{code-jsonpath-example}} shows a simple example where a JSONPatch expression is being used for this purpose.
As JSONPath is very expressive, this approach allows for fine-grained control for limiting how much of the target namespace will actually be retrived when dereferencing the namespace URI.

~~~ sdf
namespace:
  cap: https://example.com/capability/cap?minVersion=1.1.0&exclusiveMaxVersion=1.2.0&subtree=%24.sdfObject.Switch
  lamps: https://example.com/lamps
defaultNamespace: lamps
sdfObject:
  LightSwitch:
    sdfAction:
      toggle:
        sdfRef: cap:/sdfObject/Switch/sdfAction/toggle
~~~
{:sdf #code-jsonpath-example
title="Example SDF model of a light switch restricting the deferenced namespace via a JSONPath expression in dot notation."}

Note that the use of the `subtree` parameter requires percent-encoding parts of the value, which decreases the human-readability of the namespace URIs.

# Security Considerations

The security considerations of {{-sdf}} and {{-json-path}} apply to this document as well.

TODO More security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
