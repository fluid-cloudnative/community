<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Motivation](#motivation)
- [Goals](#goals)
- [Non-Goals](#non-goals)
- [API](#api)
  - [Container Image](#container-image)
  - [Custom Resource Definition](#custom-resource-definition)
  - [Resulting Worker](#resulting-worker)
  - [Resulting Launcher](#resulting-launcher)
- [Design](#design)
- [Alternatives Considered](#alternatives-considered)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

_Status_

* 2020-09-04 - 初始版本


## Motivation


## Goals
* Provide a common Custom Resource Definition (CRD) for defining a single-gpu,
multi-gpu, or multi-node training job.


## Non-Goals


## API



### Custom Resource Definition
The custom resource can be defined in two ways, in terms of how GPU resources
are specified.



### Resulting Worker
```yaml
```

## Design


## Alternatives Considered

