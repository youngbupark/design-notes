# Support multi compute platforms

* **Status**: Draft
* **Author**: Young Bu Park (@youngbupark)

## Overview

Supporting multiple compute platforms in Radius has the following challenges with the current design:

1. Applications.Core/environments resource defines the scope of environment with compute type, cloud provider, and recipes configuration. When creating this resource in the current design, Radius creates a environment-scoped namespace in Kubernetes as a landing zone. To enable environment resource in the other platform, environment resource will need to create several network resources, identity, configure policies, and etc to run applications.

2. Renderer and Deployment handler in deployment processor are overcomplicated and platform-dependent in the current Radius. Renderer includes the business logic to create Applications.Core resource types such as how to associate identity with container and mount volume to container, etc. To support multiple platforms in the current deployment processor, one of the solutions is to add a multiplexer renderer for each platform's renderer. However, with this approach, platform provider developer needs to understand the business logic in the renderer, which can diverge renderer implementations easily if they misunderstand it.

This proposal revisits the current implementation of resource type to reduce its complexity and support multi compute platforms beyond Kubernetes, such as Azure Container instance, Azuer Functions, AWS ECS, AWS Lambda, etc.

## Terms and definitions

* Deployment engine: It is known as [Bicep Deployment Resource Provider](https://docs.radapp.io/concepts/technical/architecture/#bicep-deployments-resource-provider). It is a workflow engine to process ARM templates, converted from bicep template, by resolving the dependencies of each resource in the template and create or update them in a dependency order.
* Deployment processor: Deployment processor is the part of application resource provider. It was designed to deploy Radius resources before leveraging Deployment Engine. It is kind of workflow engine to resolve the dependencies of resources and deploy and delete these resources in a correct order.
* Renderer: It converts Radius resource to platform-specific resource to be deployed to the target platform such as kubernetes, Azure, and AWS. 
* Platform provider: It is a vendor-specific provider to offer the functionality to manage the resources for the specific 

## Objectives

### Goals

1. Simplify application.core deployment processor.
2. Make resource provider platform-independent and bring platform-provider model.
3. Improve the testability of platform provider

### Non goals

TBW

### User scenarios (optional)

#### Use only Kubernetes resources

Developers use only Kubernetes resources to run their applications.

#### Use Kubernetes deployment and network resources, but uses IAM of public clouds.

Developers use Kubernetes deployments/services/gateways/secrets/configmaps resources and use blob storage and keyvault resources from Azure. To authenticate azure resources, they created user assigned managed identity and use workload identity operator in kubernetes to use the identity in their application.

#### Use Serverless computes + network resource of public clouds

Developers use Azure container instance, azure network resources to, and User managed identity to run their applications.

## Design

WIP 

### Challenges of deployment processor

**Deployment processor** is a component of Radius that resolves the dependency graph of rendered resources and deploys them in the correct order. It was designed and implemented before Radius adopted **Deployment Engine** and the new control plane provider design. However, the deployment processor has become challanging as Radius evolved and expanded its capabilities to support multiple platforms.

* Registering new resource type is challenging. It requires creating a renderer and a handler, defining a local ID, and associating them.
* Writing renderers and handlers separately is difficult. They must follow a specific interface and contract.
* Setting dependency of resources in renderers is unnecessary and complicated. The dependency graph is already resolved by the Deployment Engine, and the renderers should not have to duplicate this logic.
* Associating the outcomes (or computedValues) from the previous deployed resources to the current resources is a complex process. The deployment processor has to pass the computedValues through the data structure and the handlers have to access them.
* End-to-end validation is not easy. Renderer validation can be done independently, but to test handlers with renderers, it requires running Radius and sending requests to create environment/application/container resources.
* The current deletion operation is inflexible. It iterates the list of output resources and calls Delete on each output resource. However, some resource types may require additional logic before or after deleting each output resource. In the current design, this would require changing the code in the deployment processor, impacting all resource types rather than the specific ones.
* The update scenario is not handled properly. The dependent resources may change in the update process, but the existing resources are not updated or deleted accordingly.


### Design details

#### Platform Provider

```go
// Provider is the interface that must be implemented by a platform provider.
type Provider interface {
	Initialize() error

	// Name returns the name of the platform provider.
	Name() string

	// Container returns the container interface. Container represents container orchestration.
	Container() (ContainerProvider, error)

	// LoadBalancer returns the load balancer interface.
	LoadBalancer() (LoadBalancerProvider, error)

	// Gateway returns the gateway interface.
	Gateway() (GatewayProvider, error)

	// Identity returns the identity interface.
	Identity() (IdentityProvider, error)

	// Volume returns the volume interface.
	Volume() (VolumeProvider, error)

	// SecretStore returns the secret store interface.
	SecretStore() (SecretStoreProvider, error)
}

type ContainerProvider interface {
	Deploy(ctx context.Context, spec *ContainerSpec) error

	Delete(ctx context.Context, spec *ContainerSpec) error
	
  Scale(ctx context.Context, spec *ContainerSpec) error
}

// ...
```

#### Environment resource

##### Pass existing resource id to environment to be associated with environment - e.g. vnet/lb resource

Kubernetes environment

```json
{
  "id": "/planes/radius/local/resourceGroups/testGroup/providers/Applications.Core/environments/env0",
  "name": "env0",
  "type": "Applications.Core/environments",
  "properties": {
    "provisioningState": "Succeeded",
    "compute": {
      "kind": "Kubernetes",
      "resourceId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/testGroup/providers/Microsoft.ContainerService/managedClusters/radiusTestCluster",
      "namespace": "default",
      "identity": {
        "kind": "azure.com.workload",
        "oidcIssuer": "https://oidcissuer/oidc"
      }
    },
    "providers" : {
      "azure" : {
          "scope":"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/testGroup" 
      }
    },
    "recipes": {
      "Applications.Datastores/mongoDatabases":{
        "cosmos-recipe": {
          "templateKind": "bicep",
          "templatePath": "br:ghcr.io/sampleregistry/radius/recipes/cosmosdb"
        }
      }
    }
  }
}
```

ACI environment - User can set the existing resources

```json
{
  "id": "/planes/radius/local/resourceGroups/testGroup/providers/Applications.Core/environments/env0",
  "name": "env0",
  "type": "Applications.Core/environments",
  "properties": {
    "provisioningState": "Succeeded",
    "compute": {
      "kind": "aci",
      "network": {
        "virtualNetwork": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/cs2-envoy/providers/Microsoft.Network/virtualNetworks/vnet",
        "internalLoadBalancer": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/willsmith-aci-21324/providers/Microsoft.Network/loadBalancers/radius-demo-ilb",
        "privateDNSZone": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/cs2-envoy/providers/Microsoft.Network/privateDnsZones/cs2-envoy.cluster.local",
        "networkSecurityGroup": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/cs2-envoy/providers/Microsoft.Network/networkSecurityGroups/internal_nsg"
      },
      "identity": {
        "kind": "azure.com.workload",
        "oidcIssuer": "https://oidcissuer/oidc"
      }
    },
    "providers" : {
      "azure" : {
          "scope":"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/testGroup" 
      }
    }
  }
}
```

##### Leverage recipe for environment

```json
{
  "id": "/planes/radius/local/resourceGroups/testGroup/providers/Applications.Core/environments/env0",
  "name": "env0",
  "type": "Applications.Core/environments",
  "properties": {
    "provisioningState": "Succeeded",
    "compute": {
      "kind": "aci",
      "resourceGroup": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/cs2-envoy",
      "recipe": {
        "name": "prod-east",
        // "parameters": {}
      },
      "identity": {
        "kind": "azure.com.workload",
        "oidcIssuer": "https://oidcissuer/oidc"
      }
    },
    "providers" : {
      "azure" : {
          "scope":"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/testGroup" 
      }
    },
    "recipes": {
      "Applications.Core/environment":{
        "prod-east": {
          "templateKind": "bicep",
          "templatePath": "br:ghcr.io/radius-project/recipes/env-default-prod/"
        }
      }
    }
  }
}
```

##### Platform provider will create the default resources - this will use default environment recipe.

```json
{
  "id": "/planes/radius/local/resourceGroups/testGroup/providers/Applications.Core/environments/env0",
  "name": "env0",
  "type": "Applications.Core/environments",
  "properties": {
    "provisioningState": "Succeeded",
    "compute": {
      "kind": "aci",
      "resourceGroup": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/cs2-envoy"
    },
    "providers" : {
      "azure" : {
          "scope":"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/testGroup" 
      }
    }
  }
}
```


### API design (if applicable)

## Alternatives considered


## Test plan


## Security


## Compatibility (optional)


## Monitoring


## Development plan


## Open issues

