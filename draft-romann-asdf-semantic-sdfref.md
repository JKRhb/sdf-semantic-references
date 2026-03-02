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

For this purpose, this memo introduces two standardized query parameters (see {{parameters-table}}) that

1. allow for such version-dependent references (using the `version` parameter),
2. make it possible to restrict the part of the namespace that should be returned when deferencing a namespace URI (using the `subtree` parameter).

The version constraints used for the `version` parameter SHOULD follow Semantic Versioning or a similar approach for semantic version numbers.
An SDF Repository SHOULD adhere to the version constraints indicated via the `version` parameter and return a model with either the exact or a compatible version number.

The introduction of the `subtree` parameter is motivated by the fact that multiple models may contribute to the same SDF namespace (and fragment identifiers are not transferred)
<!-- TODO: Check whether you can actually use JSON Path with HTTP query parameters -->
Using a JSON Path query with this paramter allows for giving the SDF Repository hosting the namespace definitions a hint on which parts of the target namespace should be included in the output.
When an SDF Consumer specifies a set of `sdfObject`s and/or `sdfThing`s in its request, the SDF Repository MAY leave out definitions that are not needed to fulfill the request.
Similarly, the SDF Repository MAY combine multiple models from the same namespace into one to satisfy the Consumer's request.

| Parameter | Format                   | Description                                                                                                  |
| --------- | ------------------------ | ------------------------------------------------------------------------------------------------------------ |
| version   | any                      | Defines the version number or version range of the model that is being referenced via the namespace URI.     |
| subtree   | JSON Path {{-json-path}} | Indicates the subtreee of the referenced model that should be returned when dereferencing the namespace URI. |
{: #parameters-table title="Table with the query parameters for namespace URIs defined in this specification."}

# Examples

A simple example for a model that uses the `version` parameter is shown in {{code-version-parameter-example}}.
Here, the caret character (`^`, percent-encoded as `%5E`) in the parameter value `^1.1.0` indicates that a version compatible with version `1.0.0` should be provided by the SDF repository.
The `^` indicates that this should be a compatible _patch_ version (i.e., the third digit may be greater than 0), while a `~` would indicate the same for the _ minor_ version or second digit.
As the first digit stands for major versions, these updates are considered non-backwards-compatible and should not be retrieved automatically.

~~~ sdf
namespace:
  cap: https://example.com/capability/cap?version=%5E1.1.0
  lamps: https://example.com/lamps
defaultNamespace: lamps
sdfObject:
  LightSwitch:
    sdfAction:
      toggle:
        sdfRef: cap:/sdfObject/Switch/sdfAction/toggle
~~~
{:sdf #code-version-parameter-example
title="Example SDF model of a light switch that refers to models of capabilities compatible with version 1.1.0."}

The parametrization of URIs can also be used to use different version ranges for certain parts of a referenced namespace.
In the example shown in {{code-different-versions-parameter-example}}, a version 2.0.0 of the models hosted under `https://example.com/capability/cap` has added a new status property that has not been part of previous versions.
At the same time, a non-backwards-compatible change has been applied to the referenced `toggle` action.
To keep using the old `toggle` action while already incorporating the new `status` property, the namespacing concept allows referring to different versions via the new query parameters that have been introduced.

~~~ sdf
namespace:
  cap11: https://example.com/capability/cap?version=%5E1.1.0
  cap2: https://example.com/capability/cap?version=%5E2.0.0
  lamps: https://example.com/lamps
defaultNamespace: lamps
sdfObject:
  LightSwitch:
    sdfAction:
      toggle:
        sdfRef: cap11:/sdfObject/Switch/sdfAction/toggle
    sdfProperty:
      status:
        sdfRef: cap:/sdfObject/Switch/sdfProperty/status
~~~
{:sdf #code-different-versions-parameter-example
title="Example SDF model of a light switch that refers to two different version ranges of models from the capability namespace."}

Lastly, since we are only using definitions from the `sdfObject` `Switch` in the referenced namespace, we can limit the subtree that we are referring to via the `subtree` parameter.
{{code-jsonpath-example}} shows a simple example where a JSONPatch expression is being used for this purpose.
As JSONPath is very expressive, this approach allows for fine-grained control for limiting how much of the target namespace will actually be retrived when dereferencing the namespace URI.

~~~ sdf
namespace:
  cap: https://example.com/capability/cap?version=%5E1.1.0&subtree=%24.sdfObject.Switch
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

Note that the use of both the `version` and the `subtree` parameters may require the values to be percent-encoded, which decreases the human-readability of the namespace URIs.

# Security Considerations

The security considerations of {{-sdf}} and {{-json-path}} apply to this document as well.

TODO More security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
