# OpenFeature Integration to Strimzi

Proposal to integrate OpenFeature into Strimzi to enhance feature management capabilities.
The integration of OpenFeature across all operators will be streamlined, thanks to the enhancements proposed in [#74](https://github.com/strimzi/proposals/pull/118).

## Current Situation
Strimzi, currently leverages environment variables for feature management. 
While this approach is effective, it requires rolling updates for any changes, which can lead to service downtime. 
Moreover, it lacks the flexibility offered by modern feature flag systems, which can provide dynamic control with zero downtime when toggling features.
Furthermore, it does not allow for per-cluster or per-namespace feature gate management.

## Motivation

To provide Strimzi users with dynamic control over features without needing service restarts, enhancing the efficiency and adaptability of their deployments.

## Proposal

This proposal suggests the integration of OpenFeature, providing two primary methods for feature management:

For the current users (i.e., having classic ENV variable STRIMZI_FEATURE_GATES), we will use [Environment Variable Provider](https://github.com/open-feature/java-sdk-contrib/tree/main/providers/env-var)
already implemented by OpenFeature folks. 
Some of the common characteristics of such provider are:
1. Default method currently used. 
2. Feature gates are set via environment variables.
3. Requires rolling updates for changes.

The third one is the most important for us because if we remove it, we will benefit it from in testing and also reducing 
overall time during changes features, which users might want to include in their infrastructure.
The third point is particularly crucial because the requirement for rolling updates introduces delays in the deployment process. 
Every time a feature gate is changed or updated, the entire system needs to be restarted, which can slow down testing and development cycles. 
By removing this constraint, we could dramatically enhance testing efficiency, allowing for more rapid iterations and immediate validation of feature changes. 
Additionally, it would reduce the overall deployment time for any new feature or change, providing users with a infrastructure that responds faster to updates and modifications. 
This would lead to smoother operations, potentially fewer disruptions, improving both developer and user experiences.

On the other side for the more complex users with need of flagging system, e.g, [FlagD](https://github.com/open-feature/flagd),
which is pretty known because [OpenFeature is CNCF Incubating project](https://www.cncf.io/projects/openfeature/).
Common characteristics of this provider are:
1. New method proposed for more complex users, which needs better management over multiple features (of Strimzi)
2. Integrates with feature flagging system for dynamic feature flagging
3. Allows changes without the need for rolling updates.
4. Allows us to handle feature gates per-cluster, namespace (or both).

### Implementation Steps

Currently, there are a few classes where we use `FeatureGates` related stuff (i.e., FeatureGates abstraction).
Important thing is that we have only one feature gates across all operators so we can't configure `feature-gate-a` for 
UserOperator or `feature-gate-b` for TopicOperator. 
Both has to be configured and shared for all operators. 
There are a few classes, where we use `FeatureGates` instance:
1. ClusterOperatorConfig 
2. EntityTopicOperator 
3. EntityUserOperator 
4. AbstractConnectOperator
5. KafkaReconciler
6. ZooKeeperReconciler

In each of these classes, we essentially do the following:

```java
result.featureGatesEnvVarValue = config.featureGates().toEnvironmentVariable()
```

Here, we parse the currently configured `FeatureGates` from the `STRIMZI_FEATURE_GATES` environment variable and assign it to specific classes (e.g., the `EntityTopicOperator` class).
Furthermore, if we dive deeper into the implementation of the `UserOperator` class, we retrieve the feature gates as follows:
```java
/**
 * @return Feature gates configuration
 */
public FeatureGates featureGates() {
    return get(FEATURE_GATES);
}
```
These are fetched directly from `ConfigParameter`, which encapsulates a map implementation with additional utility methods (e.g., a parser to convert the string representation into a specified type).
In this case, the `FeatureGates` type is defined as:
```java
 /**
 * Configuration string with feature gates settings
 */
public static final ConfigParameter<FeatureGates> FEATURE_GATES = new ConfigParameter<>("STRIMZI_FEATURE_GATES", parseFeatureGates(), "", CONFIG_VALUES);
```
Here, the `parseFeatureGates()` method is defined, which calls the constructor of the `FeatureGates` class.

This is the basic flow of how it currently works, and my proposal is to modify the `FeatureGates` class to use `EnvVarProvider`, which would cover all aspects of the current implementation. 
For this, we would need to add a few dependencies (e.g., `dev.openfeature.sdk` and `dev.openfeature.contrib.providers.env-var`).
Additionally, we can easily extend support by adding other providers like `FlagDProvider` for users who want to use a feature flagging system (i.e., FlagD with its `OpenFeature` Operator). 

Moreover, the config classes will be removed and every logic (i.e., parsing part) will be moved to `FeatureGates` class as it might be the best choice. 
By encapsulating the logic within the `FeatureGates` class and removing dependencies on configuration classes, we simplify the feature gate management and make it more maintainable.
Making `FeatureGates` a singleton is beneficial because it is called from multiple parts of the code and just need one instance at the time.

The overall setup would look like this:

```java
// ...
// imports omitted for brevity
// ...

/**
 * Class for handling the configuration of feature gates
 */
public class FeatureGates {
    private static final String CONTINUE_ON_MANUAL_RU_FAILURE = "ContinueReconciliationOnManualRollingUpdateFailure";
    private static final String OPEN_FEATURE_PROVIDER_NAME_ENV = "OPEN_FEATURE_PROVIDER_NAME"; // Environment variable to choose OpenFeature provider
    private static final String STRIMZI_FEATURE_GATES_ENV = "STRIMZI_FEATURE_GATES";
    
    // instance holder
    private static FeatureGates instance;
    
    private final Client featureClient;
    private final FeatureProvider provider;

    // When adding new feature gates, do not forget to add them to allFeatureGates(), toString(), equals(), and `hashCode() methods
    private FeatureGate continueOnManualRUFailure;

    // getInstance() method impl.
    
    /**
     * Constructs the feature gates configuration.
     *
     * @param featureGatesConfig String with a comma-separated list of enabled or disabled feature gates (mainly for testing purposes)
     * @param evaluationContext EvaluationContext
     */
    private FeatureGates(String featureGatesConfig, final EvaluationContext evaluationContext) {
        this.provider = this.getProviderFromEnv();
        OpenFeatureAPI.getInstance().setProvider(this.provider);
        this.featureClient = OpenFeatureAPI.getInstance().getClient();

        if (this.isEnvVarProvider()) {
            if (featureGatesConfig == null || featureGatesConfig.isEmpty()) {
                // feature gates config is null or empty so we are gonna retrieve it from ENV VAR
                featureGatesConfig = this.featureClient.getStringValue(STRIMZI_FEATURE_GATES_ENV, "");
            }

            // parse the featureGateConfig if it's provided
            this.validateFeatureGateConfig(featureGatesConfig);
        } else {
            // other providers (e.g., flagd)
            this.continueOnManualRUFailure = evaluationContext != null ?
                new FeatureGate(CONTINUE_ON_MANUAL_RU_FAILURE, fetchFeatureFlag(CONTINUE_ON_MANUAL_RU_FAILURE, false, Boolean.class, evaluationContext)) :
                new FeatureGate(CONTINUE_ON_MANUAL_RU_FAILURE, fetchFeatureFlag(CONTINUE_ON_MANUAL_RU_FAILURE, false, Boolean.class));
        }

        // Validate interdependencies (if any)
        this.validateInterDependencies();
    }
}
```
In the context of `EnvVarProvider`, it should be fairly simple to implement. 
By using the new provider and adapting the current implementation, it should function the same as our existing approach.
Alternatively, when using other `Provider` (e.g., FlagD), there are a couple of options: (i) an user can deploy just standalone application FlagD as a deployment, 
or (ii) deploy the [OpenFeature Operator](https://github.com/open-feature/open-feature-operator), which supports FlagD as one of its flagging systems. 
While researching, I also discovered several other feature flagging systems, such as:
1. [Go Feature Flag](https://gofeatureflag.org/)
2. [CloudBees Feature Management](https://www.cloudbees.com/capabilities/feature-management)
3. [Split](https://www.split.io/)
4. [Harness](https://harness.io/products/feature-flags)
5. [LaunchDarkly](https://launchdarkly.com/)
6. [Flagsmith](https://flagsmith.com/)
7. [Flipt](https://www.flipt.io/)

Users have the flexibility to choose any feature flagging provider that suits their needs, thanks to the integration with OpenFeature, which supports multiple providers.
Given its community support and inclusion in the CNCF ecosystem, OpenFeature offers a versatile solution for feature management.

Conceptually, the communication between Strimzi and a feature flagging system can be illustrated as:


    +------------------------------+      +------------------------------------+
    | Centralized Feature Flagging |      |               Strimzi              |
    |            Server            |      | (Cluster, User and Topic Operator) |
    +------------------------------+      +------------------------------------+
                   |                                        |
                   |                                        |
                   |   <----------- API Calls ----------->  |

Where in each component (i.e., ClusterOperator, UserOperator and TopicOperator), we will fetch feature flags dynamically from the OpenFeature API, which is managed by feature flagging server.
Each operator’s logic that is controlled and **centralized** by FeatureGates class (e.g., enabling new behaviors, managing rolling updates) will dynamically receive flag updates from feature flagging server.

Example of the feature flags with using `FlagD` within OpenFeature Operator.
```yaml
apiVersion: core.openfeature.dev/v1beta1
kind: FeatureFlag
metadata:
  name: strimzi-feature-gates
  labels:
    app: strimzi-feature-gates
spec:
  flagSpec:
    flags:
      feature-gate-a:
        variants:
          'on': true
          'off': false
        defaultVariant: 'off'
      feature-gate-b:
        variants:
          'on': true
          'off': false
        defaultVariant: 'on'
        state: ENABLED
#        ... and more
```

and then we would need to implement in reconcile loop of each component call for OpenFeature API using its client.
For `UserOperator` that's `UserControllerLoop` class.
```java
// UserControllerLoop.java content 
class UserControllerLoop {
    // ...
    /**
     * The main reconciliation logic which handles the reconciliations.
     *
     * @param reconciliation    Reconciliation identifier used for logging
     */
    @Override
    protected void reconcile(Reconciliation reconciliation) {
        LOGGER.infoCr(reconciliation, "{} will be reconciled", reconciliation.kind());

        //  update the state of feature gates dynamically from feature flagging system
        FeatureGates.getInstance().updateFeatureGateStates();
        LOGGER.infoCr(reconciliation, "Fetching from feature flagging system: continueOnManualRUFailureEnabled is enabled: {}", featureGates.continueOnManualRUFailureEnabled());

        KafkaUser user = userLister.namespace(reconciliation.namespace()).get(reconciliation.name());

    // ...
    }
}
```
And `maybeUpdateFeatureGateA()` would change the state of inner FeatureGate instance for each component.
Meaning that for TopicOperator we will have different `FEATURE_GATES` as for `UserOperator` if necessary.
```java
/**
 * Fetches and updates the feature gates state dynamically from the OpenFeature API.
 */
public void maybeUpdateFeatureGateA() {
    if (!this.isEnvVarProvider()) {
        // Fetch dynamically from flagging system and update internal states
        this.continueOnManualRUFailure.setValue(fetchFeatureFlag(FEATURE_GATE_A, true, Boolean.class));
    }
    // if using EnvVar provider there is no need to set such value twice 
}
```
After such update we can easily access those updated values by simply calling:
```java
if (FeatureGates.getInstance().continueOnManualRUFailureEnabled()) {
    // ... and do some logic...
}
```
and it would be accessible from `UserControllerLoop` class with form of getter. 
For now, we do not have any `FEATURE_GATES` for `UserOperator` so there will be no such logic needed
but maybe in the future we can simply add methods for each `FEATURE_GATE`; meaning for `UserOperator, TopicOperator or ClusterOperator` we will have
`featureGateA`, `featureGateB`, and in each of their reconciles loop we would simply call `maybeUpdateFeatureGate<A-B>`.

### Potential configuration of Feature Gates per Kafka cluster

With `OpenFeature` there is a possibility to define feature gates specific to each Kafka cluster by using the cluster name as part of the metadata or by associating flags with specific clusters. 
This allows you to customize feature gates per cluster within the `ClusterOperator`.
To design feature gates based on the Kafka cluster name for the `ClusterOperator`, we can extend the configuration to include cluster-specific feature gates. Here’s a potential design:

#### Example YAML Configuration for Cluster-specific Feature Gates

```yaml
# Feature gates for Kafka clusters
apiVersion: core.openfeature.dev/v1beta1
kind: FeatureFlag
metadata:
  name: kafka-cluster-feature-flags
  labels:
    app: kafka-cluster
spec:
  flagSpec:
    flags:
      kafka-cluster-a-feature-gate-a:
        variants:
          'on': true
          'off': false
        defaultVariant: 'on'
        state: ENABLED
      kafka-cluster-b-feature-gate-b:
        variants:
          'on': true
          'off': false
        defaultVariant: 'off'
        state: ENABLED
    # Additional cluster-specific feature gates can be added here
```

Moreover, other simplified implementation approach would be to use `single configuration for cluster-specific gates`. 
Meaning, instead of creation separate configurations for each cluster, leverage labels or annotations to differentiate clusters within a single YAML configuration:
```yaml
apiVersion: core.openfeature.dev/v1beta1
kind: FeatureFlag
metadata:
  name: kafka-cluster-feature-flags
  labels:
    app: kafka-cluster-feature-flags
spec:
  flagSpec:
    flags:
      feature-gate-x:
        variants:
          'on': true
          'off': false
        defaultVariant: 'off'
        state: ENABLED
        targeting:
          - if:
            - or:
                - and:
                    - "==":
                        - var: clusterName
                        - "kafka-cluster-a"
                - and:
                    - "==":
                      - var: clusterName
                      - "kafka-cluster-b"
                - "on"
                - "off"
          
#  ...
```
In this case, we have defined `feature-gate-x`, which is by default disabled (i.e., defaultVariant is set to `off`).
And, if we deploy Kafka cluster with `kafka-cluster-a` or `kafka-cluster-b` then such feature gate would be enabled (in case of deploying `kafka-cluster-c` is following the default value; meaning `off`).

To extend this approach to be namespace-wide, we can introduce targeting rules that consider the namespace along with the cluster name. 
This allows the feature flags to be applied to specific namespaces across different clusters.

```yaml
apiVersion: core.openfeature.dev/v1beta1
kind: FeatureFlag
metadata:
  name: kafka-cluster-feature-flags
  labels:
    app: kafka-cluster-feature-flags
spec:
  flagSpec:
    flags:
      feature-gate-x:
        variants:
          'on': true
          'off': false
        defaultVariant: 'off'
        state: ENABLED
        targeting:
          targeting:
          if:
            - or:
                - and:
                    - "==":
                        - var: clusterName
                        - "kafka-cluster-a"
                    - "==":
                        - var: namespace
                        - "namespace-a"
                - and:
                    - "==":
                        - var: clusterName
                        - "kafka-cluster-b"
                    - "==":
                        - var: namespace
                        - "namespace-b"
            - "on"
            - "off"
#  ...
```

In the client code, we would handle this by `EvaluationContext`, which is provided by OpenFeature SDK. 
That way for us, we would just call `fetchFeatureFlag()` method where we also specify `EvaluationContext` and then 
we would have result. 
For instance:
```java
// assuming 
//  cluster name = kafka-cluster-a
//  namespace    = namespace-a
this.evaluationContent.add("clusterName", kafkaCr.getMetadata().getName());
this.evaluationContent.add("namespace", kafkaCr.getMetadata().getNamespace());

this.featureGates.fetchFeatureFlag("feature-gate-x", false, Boolean.class, evaluationContent)
// this will return `true`, because our defined targeting above.
```

### Benefits

- **Flexibility:** Users can toggle features without redeploying or restarting services.
- **Faster Iterations/Testing:** Features can be tested and rolled out quickly, speeding up development cycles.
- **Centralized Management:** feature flagging system integration allows centralized control of feature flags, simplifying management across multiple components.
- **Scalability:** The approach scales efficiently for larger deployments without adding operational complexity.
- **Backwards Compatibility:** The proposal maintains support for the existing `STRIMZI_FEATURE_GATES` method (i.e., env-var provider), ensuring a smooth transition.

### Potential Challenges

- **Complexity:** Increased complexity in configuration management.
- **Dependency:** Additional dependency on the feature flagging systems.

## Affected/Not Affected Projects

`Cluster Operator`, `Topic Operator` and `User Operator`; meaning the modification will be done in scope of `strimzi-cluster-operator` project.
Moreover, `operator-common` is also affected, because we modify FeatureGates class.

## Questions

1. What if an user configure `STRIMZI_FEATURE_GATES` as environment variable and also configure flagging system?

Flagging system has priority and if flagging system is used then `STRIMZI_FEATURE_GATES` should be ignored.

2. What if an user configured flagging system and then move on classic `STRIMZI_FEATURE_GATES` env-var?

If we follow proposal design, then it would simply fetch value, which is set from `STRIMZI_FEATURE_GATES` environment variable (expected).

## Compatibility

The introduction of OpenFeature is backwards compatible, designed to enhance, not replace, current configurations.

## Rejected Alternatives

- **Single Provider Approach:** Initially considered using only FlagD, but rejected to maintain flexibility for users accustomed to the current environment variable method.

This proposal aims to modernize Strimzi's feature management, providing a bridge to more dynamic configuration methods while respecting traditional deployment practices.