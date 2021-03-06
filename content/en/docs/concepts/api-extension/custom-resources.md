---
title: Custom Resources
reviewers:
- enisoc
- deads2k
content_template: templates/concept
---

{{% capture overview %}}
This page explains [*custom resources*](/docs/concepts/api-extension/custom-resources/), which are extensions of the Kubernetes API. This page explains when to add a custom resource to your Kubernetes cluster and when to use a standalone service. It describes the two methods for adding custom resources and how to choose between them.

{{% /capture %}}

{{% capture body %}}
## Custom resources

A *resource* is an endpoint in the [Kubernetes API](/docs/reference/api-overview/) that stores a collection of [API objects](/docs/concepts/overview/working-with-objects/kubernetes-objects/) of a certain kind. For example, the built-in *pods* resource contains a collection of Pod objects.

A *custom resource* is an extension of the Kubernetes API that is not necessarily available on every
Kubernetes cluster.
In other words, it represents a customization of a particular Kubernetes installation.

Custom resources can appear and disappear in a running cluster through dynamic registration,
and cluster admins can update custom resources independently of the cluster itself.
Once a custom resource is installed, users can create and access its objects with
[kubectl](/docs/user-guide/kubectl-overview/), just as they do for built-in resources like *pods*.

### Custom controllers

On their own, custom resources simply let you store and retrieve structured data.
It is only when combined with a *controller* that they become a true [declarative API](/docs/concepts/overview/working-with-objects/kubernetes-objects/#understanding-kubernetes-objects). The controller interprets the structured data as a record of the user's desired state, and continually takes action to achieve and maintain that state.

A *custom controller* is a controller that users can deploy and update on a running cluster, independently of the cluster's own lifecycle. Custom controllers can work with any kind of resource, but they are especially effective when combined with custom resources. The [Operator](https://coreos.com/blog/introducing-operators.html) pattern is one example of such a combination. It allows developers to encode domain knowledge for specific applications into an extension of the Kubernetes API.

### Should I add a custom resource to my Kubernetes Cluster?

When creating a new API, consider whether to [aggregate your API with the Kubernetes cluster APIs](/docs/concepts/api-extension/apiserver-aggregation/) or let your API stand alone.

| Consider API aggregation if: | Prefer a stand-alone API if: |
|-|-|
| Your API is [Declarative](#declarative-apis). | Your API does not fit the [Declarative](#declarative-apis) model. |
| You want your new types to be readable and writable using `kubectl`.| `kubectl` support is not required |
| You want to view your new types in a Kubernetes UI, such as dashboard, alongside built-in types. | Kubernetes UI support is not required. |
| You are developing a new API. | You already have a program that serves your API and works well. |
| You are willing to accept the format restriction that Kubernetes puts on REST resource paths, such as API Groups and Namespaces. (See the [API Overview](/docs/concepts/overview/kubernetes-api/).) | You need to have specific REST paths to be compatible with an already defined REST API. |
| Your resources are naturally scoped to a cluster or to namespaces of a cluster. | Cluster or namespace scoped resources are a poor fit; you need control over the specifics of resource paths. |
| You want to reuse [Kubernetes API support features](#common-features).  | You don't need those features. |

#### Declarative APIs

In a Declarative API, typically:
 - Your API consists of a relatively small number of relatively small objects (resources).
 - The objects define configuration of applications or infrastructure.
 - The objects are updated relatively infrequently.
 - Humans often need to read and write the objects.
 - The main operations on the objects are CRUD-y (creating, reading, updating and deleting).
 - Transactions across objects are not required: the API represents a desired state, not an exact state.

Imperative APIs are not declarative.
Signs that your API might not be declarative include:
 - The client says "do this", and then gets a synchronous response back when it is done.
 - The client says "do this", and then gets an operation ID back, and has to check a separate Operation objects to determine completion of the request.
 - You talk about Remote Procedure Calls (RPCs).
 - Directly storing large amounts of data (e.g. > a few kB per object, or >1000s of objects).
 - High bandwidth access (10s of requests per second sustained) needed.
 - Store end-user data (such as images, PII, etc) or other large-scale data processed by applications.
 - The natural operations on the objects are not CRUD-y.
 - The API is not easily modeled as objects.
 - You chose to represent pending operations with an operation ID or operation object.

### Should I use a configMap or a custom resource?

Use a ConfigMap if any of the following apply:

* There is an existing, well-documented config file format, such as a `mysql.cnf` or `pom.xml`.
* You want to put the entire config file into one key of a configMap.
* The main use of the config file is for a program running in a Pod on your cluster to consume the file to configure itself.
* Consumers of the file prefer to consume via file in a Pod or environment variable in a pod, rather than the Kubernetes API.
* You want to perform rolling updates via Deployment, etc, when the file is updated.

{{< note >}}
**Note:** Use a [secret](/docs/concepts/configuration/secret/) for sensitive data, which is similar to a configMap but more secure.
{{< /note >}}

Use a custom resource (CRD or Aggregated API) if most of the following apply:

* You want to use Kubernetes client libraries and CLIs to create and update the new resource.
* You want top-level support from kubectl (for example: `kubectl get my-object object-name`).
* You want to build new automation that watches for updates on the new object, and then CRUD other objects, or vice versa.
* You want to write automation that handles updates to the object.
* You want to use Kubernetes API conventions like `.spec`, `.status`, and `.metadata`.
* You want the object to be an abstraction over a collection of controlled resources, or a summarization of other resources.

## Adding custom resources

Kubernetes provides two ways to add custom resources to your cluster:

- [Custom Resource Definitions](/docs/concepts/api-extension/custom-resources/) (CRDs) are easier to use: they do not require any programming in some cases.
- [API Aggregation](/docs/concepts/api-extension/apiserver-aggregation/) requires programming, but allows more control over API behaviors like how data is stored and conversion between API versions.

Kubernetes provides these two options to meet the needs of different users, so that neither ease of use nor flexibility are compromised.

Aggregated APIs are subordinate APIServers that sit behind the primary API server, which acts as a proxy. This arrangement is called [API Aggregation](/docs/concepts/api-extension/apiserver-aggregation/) (AA). To users, it simply appears that the Kubernetes API is extended.

Custom Resource Definitions (CRDS) allow users to create new types of resources without adding another APIserver. You do not need to understand API Aggregation to use CRDs.

Regardless of whether they are installed via CRDs or AA, the new resources are called Custom Resources to distinguish them from built-in Kubernetes resources (like pods).

## CustomResourceDefinitions

The [CustomResourceDefinition](/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/) (CRD) API resource allows you to define custom resources. Defining a CRD object creates a new custom resource with a name and schema that you specify. The Kubernetes API serves and handles the storage of your custom resource.

This frees you from writing your own API server to handle the custom resource,
but the generic nature of the implementation means you have less flexibility than with
[API server aggregation](#api-server-aggregation).

Refer to the [Custom Controller example, which uses Custom Resources](https://github.com/kubernetes/sample-controller)
for a demonstration of how to register a new custom resource, work with instances of your new resource type,
and setup a controller to handle events.

{{< note >}}
**Note:** CRD is the successor to the deprecated *ThirdPartyResource* (TPR) API, and is available as of Kubernetes 1.7.
{{< /note >}}

## API server aggregation

Usually, each resource in the Kubernetes API requires code that handles REST requests and manages persistent storage of objects. The main Kubernetes API server handles built-in resources like *pods* and *services*, and can also handle custom resources in a generic way through [CustomResourceDefinitions](#customresourcedefinitions).

The [aggregation layer](/docs/concepts/api-extension/apiserver-aggregation/) allows you to provide specialized
implementations for your custom resources by writing and deploying your own standalone API server.
The main API server delegates requests to you for the custom resources that you handle,
making them available to all of its clients.

### Choosing a method for adding custom resources

CRDs are easier to use. Aggregated APIs are more flexible. Choose the method that best meets your needs.

Typically, CRDs are a good fit if:

* You have a handful of fields
* You are using the resource within your company, or as part of a small open-source project (as opposed to a commercial product)

#### Comparing ease of use

CRDs are easier to create than Aggregated APIs.

| Custom Resource Definitions | Aggregated API |
|-|-|
| Do not require programming. Users can choose any language for a CRD controller. | Requires programming in Go and building binary and image. Users can choose any language for a CRD controller. |
| No additional service to run; CRs are handled by API Server. | An additional service to create and that could fail. |
| No ongoing support once the CRD is created. Any bug fixes are picked up as part of normal Kubernetes Master upgrades. | May need to periodically pickup bug fixes from upstream and rebuild and update the Aggregated APIserver. |
| No need to handle multiple versions of your API. For example: when you control the client for this resource, you can upgrade it in sync with the API. | You need to handle multiple versions of your API, for example: when developing an extension to share with the world. |

### Advanced features and flexibility

Aggregated APIs offer more advanced API features and customization of other features, for example: the storage layer.

| Feature | Description | CRDs | Aggregated API |
|-|-|-|-|
| Validation | Help users prevent errors and allow you to evolve your API independently of your clients. These features are most useful when there are many clients who can't all update at the same time. | Beta feature of CRDs in v1.9. Checks limited to what is supported by OpenAPI v3.0. | Yes, arbitrary validation checks |
| Defaulting | See above | No, but can achieve the same effect with an Initializer (requires programming) | Yes |
| Multi-versioning | Allows serving the same object through two API versions. Can help ease API changes like renaming fields. Less important if you control your client versions. | No | Yes |
| Custom Storage | If you need storage with a different performance mode (for example, time-series database instead of key-value store) or isolation for security (for example, encryption secrets or different | No | Yes |
| Custom Business Logic | Perform arbitrary checks or actions when creating, reading, updating or deleting an object | No, but can get some of the same effects with Initializers or Finalizers (requires programming) | Yes |
| Subresources | {::nomarkdown}<ul><li>Add extra operations other than CRUD, such as "scale" or "exec"</li><li>Allows systems like HorizontalPodAutoscaler and PodDisruptionBudget interact with your new resource</li><li>Finer-grained access control: user writes spec section, controller writes status section.</li><li>Allows incrementing object Generation on custom resource data mutation (requires separate spec and status sections in the resource)</li></ul>{:/} | No but planned | Yes, any Subresource |
| strategic-merge-patch | The new endpoints support PATCH with `Content-Type: application/strategic-merge-patch+json`. Useful for updating objects that may be modified both locally, and by the server. For more information, see ["Update API Objects in Place Using kubectl patch"](/docs/tasks/run-application/update-api-object-kubectl-patch/) | No | Yes |
| Protocol Buffers | The new resource supports clients that want to use Protocol Buffers | No | Yes |
| OpenAPI Schema | Is there an OpenAPI (swagger) schema for the types that can be dynamically fetched from the server? Is the user protected from misspelling field names by ensuring only allowed fields are set? Are types enforced (in other words, don't put an `int` in a `string` field?) | No but planned | Yes |

#### Common Features

When you create a custom resource, either via a CRDs or an AA, you get many features for your API, compared to implementing it outside the Kubernetes platform:

| Feature | What it does |
|-|-|
| CRUD | The new endpoints support CRUD basic operations via HTTP and `kubectl` |
| Watch | The new endpoints support Kubernetes Watch operations via HTTP |
| Discovery | Clients like kubectl and dashboard automatically offer list, display, and field edit operations on your resources |
| json-patch | The new endpoints support PATCH with `Content-Type: application/json-patch+json` |
| merge-patch | The new endpoints support PATCH with `Content-Type: application/merge-patch+json` |
| HTTPS | The new endpoints uses HTTPS |
| Built-in Authentication | Access to the extension uses the core apiserver (aggregation layer) for authentication |
| Built-in Authorization | Access to the extension can reuse the authorization used by the core apiserver (e.g. RBAC) |
| Finalizers | Block deletion of extension resources until external cleanup happens. |
| Admission Webhooks | Set default values and validate extension resources during any create/update/delete operation. |
| UI/CLI Display | Kubectl, dashboard can display extension resources. |
| Unset vs Empty | Clients can distinguish unset fields from zero-valued fields. |
| Client Libraries Generation | Kubernetes provides generic client libraries, as well as tools to generate type-specific client libraries. |
| Labels and annotations | Common metadata across objects that tools know how to edit for core and custom resources. |

## Preparing to install a custom resource

There are several points to be aware of before adding a custom resource to your cluster.

### Third party code and new points of failure

While creating a CRD does not automatically add any new points of failure (for example, by causing third party code to run on your API server), packages (for example, Charts) or other installation bundles often include CRDs as well as a Deployment of third-party code that implements the business logic for a new custom resource.

Installing an Aggregated APIserver always involves running a new Deployment.

### Storage

Custom resources consume storage space in the same way that ConfigMaps do. Creating too many custom resources may overload your API server's storage space.

Aggregated API servers may use the same storage as the main API server, in which case the same warning applies.

### Authentication, authorization, and auditing

CRDs always use the same authentication, authorization, and audit logging as the built-in resources of your API Server.

If you use RBAC for authorization, most RBAC roles will not grant access to the new resources (except the cluster-admin role or any role created with wildcard rules). You'll need to explicitly grant access to the new resources. CRDs and Aggregated APIs often come bundled with new role definitions for the types they add.

Aggregated API servers may or may not use the same authentication, authorization, and auditing as the primary API server.

## Accessing a custom resource

Kubernetes [client libraries](/docs/reference/client-libraries/) can be used to access custom resources. Not all client libraries support custom resources. The go and python client libraries do.

When you add a custom resource, you can access it using:
  - kubectl
  - The kubernetes dynamic client.
  - A REST client that you write.
  - A client generated using [Kubernetes client generation tools](https://github.com/kubernetes/code-generator) (generating one is an advanced undertaking, but some projects may provide a client along with the CRD or AA).

{{% /capture %}}

{{% capture whatsnext %}}
* Learn how to [Extend the Kubernetes API with the aggregation layer](/docs/concepts/api-extension/apiserver-aggregation/).
* Learn how to [Extend the Kubernetes API with CustomResourceDefinition](/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/).
* Learn how to [Migrate a ThirdPartyResource to CustomResourceDefinition](/docs/tasks/access-kubernetes-api/migrate-third-party-resource/).
{{% /capture %}}


