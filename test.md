# code samples

## terraform-provider-akamai

- Akamai has good infrastructure, but bad APIs. Specially cloudlets is one of the worst APIs I have ever used, too many inconsistencies.
  - I had to cope with them, as this API was marked "to rewrite" and until then there wouldn't be any fix.
  - Several modifications have been done since my implentation - I am giving you here my last version which dates from Nov 2021
- I was in charge of developing this terraform resource - you can check [git blame|https://github.com/akamai/terraform-provider-akamai/blob/b342417bd0a41f4e960c91c6f8167f465625dc64/pkg/providers/cloudlets/resource_akamai_cloudlets_policy_activation.go]
- The terraform-plugin-sdk allows to create resources as CRUD components
  - Create context
  - Read context
  - Update context
  - Delete context
- more comments below

```go
package cloudlets

import (
	"context"
	"errors"
	"fmt"
	"reflect"
	"sort"
	"strings"
	"time"

	"github.com/apex/log"

	"github.com/akamai/AkamaiOPEN-edgegrid-golang/v2/pkg/cloudlets"
	"github.com/akamai/AkamaiOPEN-edgegrid-golang/v2/pkg/session"
	"github.com/akamai/terraform-provider-akamai/v2/pkg/akamai"
	"github.com/akamai/terraform-provider-akamai/v2/pkg/tools"
	"github.com/hashicorp/terraform-plugin-sdk/v2/diag"
	"github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
)

func resourceCloudletsPolicyActivation() *schema.Resource {
	return &schema.Resource{
                // these are the different methods defining the CRUD stages
		CreateContext: resourcePolicyActivationCreate,
		ReadContext:   resourcePolicyActivationRead,
		UpdateContext: resourcePolicyActivationUpdate,
		DeleteContext: resourcePolicyActivationDelete,
                // resource schema, to store information from the remote api
		Schema:        resourceCloudletsPolicyActivationSchema(),
		Timeouts: &schema.ResourceTimeout{
			Default: &PolicyActivationResourceTimeout,
		},
	}
}

func resourceCloudletsPolicyActivationSchema() map[string]*schema.Schema {
	return map[string]*schema.Schema{
		"status": {
			Type:        schema.TypeString,
			Computed:    true,  // this value cannot be set by `terraform plan` or `apply`
			Description: "Activation status for this Cloudlets policy",
		},
		"policy_id": {
			Type:        schema.TypeInt,
			Required:    true,
			Description: "ID of the Cloudlets policy you want to activate",
			ForceNew:    true,
		},
		"network": {
			Type:             schema.TypeString,
			Required:         true,
			ForceNew:         true,  // this switch causes terraform to destroy + create new resource, if this value changes
			ValidateDiagFunc: tools.ValidateNetwork,  // we can define validation methods
			StateFunc:        statePolicyActivationNetwork,  // this function maps the given value to the actual value of the field
			Description:      "The network you want to activate the policy version on (options are Staging and Production)",
		},
		"version": {
			Type:        schema.TypeInt,
			Required:    true,
			Description: "Cloudlets policy version you want to activate",
		},
		"associated_properties": {
			Type:        schema.TypeSet,
			Required:    true,
			Elem:        &schema.Schema{Type: schema.TypeString},
			Description: "Set of property IDs to link to this Cloudlets policy",
		},
	}
}

var (
	// ActivationPollMinimum is the minimum polling interval for activation creation
	ActivationPollMinimum = time.Minute

	// ActivationPollInterval is the interval for polling an activation status on creation
	ActivationPollInterval = ActivationPollMinimum

	// PolicyActivationResourceTimeout is the default timeout for the resource operations
	PolicyActivationResourceTimeout = time.Minute * 90

	// ErrNetworkName is used when the user inputs an invalid network name
	ErrNetworkName = errors.New("invalid network name")
)

func resourcePolicyActivationDelete(ctx context.Context, rd *schema.ResourceData, m interface{}) diag.Diagnostics {
	meta := akamai.Meta(m)
	logger := meta.Log("Cloudlets", "resourcePolicyActivationDelete")
	logger.Debug("Deleting cloudlets policy activation")
	ctx = session.ContextWithOptions(ctx, session.WithContextLog(logger))
	client := inst.Client(meta)

	pID, err := tools.GetIntValue("policy_id", rd)  // we start by getting data from the existing schema (this is owned by the remote API, terraform does not dictate its format in any way)
	if err != nil {
		return diag.FromErr(err)
	}
	policyID := int64(pID)

	network, err := getPolicyActivationNetwork(strings.Split(rd.Id(), ":")[1])  // parsing the network name from the given id
	if err != nil {
		return diag.FromErr(err)
	}

	policyProperties, err := client.GetPolicyProperties(ctx, cloudlets.GetPolicyPropertiesRequest{PolicyID: policyID})  // fetching policy properties using the akamaiopen client
	if err != nil {
		return diag.Errorf("%s: cannot find policy %d properties: %s", ErrPolicyActivation.Error(), policyID, err.Error())
	}
	activations, err := client.ListPolicyActivations(ctx, cloudlets.ListPolicyActivationsRequest{ // getting further details from the remote API
		PolicyID: policyID,
		Network:  network,
	})
	if err != nil {
		return diag.FromErr(err)
	}

        // to delete the activation, we need to delete all its policy properties
	logger.Debugf("Removing all policy (ID=%d) properties", policyID)
	for propertyName, policyProperty := range policyProperties {
		// filter out property by network
		validProperty := false
		for _, act := range activations {
			if act.PropertyInfo.Name == propertyName {
				validProperty = true
				break
			}
		}
		if !validProperty {
			continue
		}
		// wait for removal until there aren't any pending activations - this is how the API works
		if err = waitForNotPendingPolicyActivation(ctx, logger, client, policyID, network); err != nil {
			return diag.FromErr(err)
		}

		// proceed to delete property from policy
		err = client.DeletePolicyProperty(ctx, cloudlets.DeletePolicyPropertyRequest{
			PolicyID:   policyID,
			PropertyID: policyProperty.ID,
			Network:    network,
		})
		if err != nil {
			return diag.Errorf("%s: cannot delete property '%s' from policy ID %d and network '%s'. Please, try once again later.\n%s", ErrPolicyActivation.Error(), propertyName, policyID, network, err.Error())
		}
	}
	logger.Debugf("All properties have been removed from policy ID %d", policyID)
	rd.SetId("")
	return nil
}

func resourcePolicyActivationUpdate(ctx context.Context, rd *schema.ResourceData, m interface{}) diag.Diagnostics {
        // creates a new policy activation if there is any remote change
	meta := akamai.Meta(m)
	logger := meta.Log("Cloudlets", "resourcePolicyActivationUpdate")

	// 1. check if version has changed.
	if !rd.HasChanges("version", "associated_properties") {
		logger.Debugf("nothing to update")
		return nil
	}

	logger.Debugf("proceeding to create and activate a new policy activation version")

	ctx = session.ContextWithOptions(ctx, session.WithContextLog(logger))
	client := inst.Client(meta)

	// 2. In such case, create a new version to activate (for creation, look into resource policy)
	policyID, err := tools.GetIntValue("policy_id", rd)
	if err != nil {
		return diag.FromErr(err)
	}

	network, err := tools.GetStringValue("network", rd)
	if err != nil {
		return diag.FromErr(err)
	}
	activationNetwork, err := getPolicyActivationNetwork(network)
	if err != nil {
		return diag.FromErr(err)
	}

	v, err := tools.GetIntValue("version", rd)
	if err != nil {
		return diag.FromErr(err)
	}
	version := int64(v)
	// policy version validation
	_, err = client.GetPolicyVersion(ctx, cloudlets.GetPolicyVersionRequest{
		PolicyID:  int64(policyID),
		Version:   version,
		OmitRules: true,
	})
	if err != nil {
		return diag.Errorf("%s: cannot find the given policy version (%d): %s", ErrPolicyActivation.Error(), version, err.Error())
	}

	associatedProps, err := tools.GetSetValue("associated_properties", rd)
	if err != nil && !errors.Is(err, tools.ErrNotFound) {
		return diag.FromErr(err)
	}
	if errors.Is(err, tools.ErrNotFound) {
		return diag.Errorf("Field associated_properties should not be empty. If you want to remove all policy associated properties, please run `terraform destroy` instead.")
	}
	var newPolicyProperties []string
	for _, prop := range associatedProps.List() {
		newPolicyProperties = append(newPolicyProperties, prop.(string))
	}
	sort.Strings(newPolicyProperties)

	// 3. look for activations with this version which is active in the given network
	activations, err := client.ListPolicyActivations(ctx, cloudlets.ListPolicyActivationsRequest{
		PolicyID: int64(policyID),
		Network:  activationNetwork,
	})
	if err != nil {
		return diag.Errorf("%v update: %s", ErrPolicyActivation, err.Error())
	}
	// activations, at this point, contains old and new activations

	// sort by activation date, reverse. To find out the state of the latest activations
	activations = sortPolicyActivationsByDate(activations)

	// find out which properties are activated in those activations
	// version does not matter at this point
	activeProps := getActiveProperties(activations)

	// 4. all "additional_properties" are active for the given version, policyID and network, proceed to read stage
	if reflect.DeepEqual(activeProps, newPolicyProperties) && !rd.HasChanges("version") {
		// in such case, return
		logger.Debugf("This policy (ID=%d, version=%d) is already active.", policyID, version)
		return resourcePolicyActivationRead(ctx, rd, m)
	}

	// 5. remove from the server all unnecessary policy associated_properties
	removedProperties, err := syncToServerRemovedProperties(ctx, logger, client, int64(policyID), activationNetwork, activeProps, newPolicyProperties)
	if err != nil {
		return diag.FromErr(err)
	}

	logger.Debugf("This policy (ID=%d, version=%d, properties=[%s], network='%s') is not active. Proceeding to activation.",
		policyID, version, strings.Join(newPolicyProperties, ", "), activationNetwork)

	err = client.ActivatePolicyVersion(ctx, cloudlets.ActivatePolicyVersionRequest{
		PolicyID: int64(policyID),
		Async:    true,
		Version:  version,
		PolicyVersionActivation: cloudlets.PolicyVersionActivation{
			Network:                 activationNetwork,
			AdditionalPropertyNames: newPolicyProperties,
		},
	})
	if err != nil {
		return diag.Errorf("%v update: %s", ErrPolicyActivation, err.Error())
	}

	// 4. poll until active - we have to wait here
	_, err = waitForPolicyActivation(ctx, client, int64(policyID), version, activationNetwork, newPolicyProperties, removedProperties)
	if err != nil {
		return diag.Errorf("%v update: %s", ErrPolicyActivation, err.Error())
	}

	return resourcePolicyActivationRead(ctx, rd, m)
}

func resourcePolicyActivationCreate(ctx context.Context, rd *schema.ResourceData, m interface{}) diag.Diagnostics {
	meta := akamai.Meta(m)
	logger := meta.Log("Cloudlets", "resourcePolicyActivationCreate")
	ctx = session.ContextWithOptions(ctx, session.WithContextLog(logger))
	client := inst.Client(meta)

	logger.Debug("Creating policy activation")

	policyID, err := tools.GetIntValue("policy_id", rd)
	if err != nil {
		return diag.FromErr(err)
	}
	network, err := tools.GetStringValue("network", rd)
	if err != nil {
		return diag.FromErr(err)
	}
	versionActivationNetwork, err := getPolicyActivationNetwork(network)
	if err != nil {
		return diag.FromErr(err)
	}
	associatedProps, err := tools.GetSetValue("associated_properties", rd)
	if err != nil && !errors.Is(err, tools.ErrNotFound) {
		return diag.FromErr(err)
	}
	var associatedProperties []string
	for _, prop := range associatedProps.List() {
		associatedProperties = append(associatedProperties, prop.(string))
	}
	sort.Strings(associatedProperties)

	v, err := tools.GetIntValue("version", rd)
	if err != nil {
		return diag.FromErr(err)
	}
	version := int64(v)

        // 1. check if the policy version is active
	logger.Debugf("checking if policy version %d is active", version)
	policyVersion, err := client.GetPolicyVersion(ctx, cloudlets.GetPolicyVersionRequest{
		Version:   version,
		PolicyID:  int64(policyID),
		OmitRules: true,
	})
	if err != nil {
		return diag.Errorf("%s: cannot find the given policy version (%d): %s", ErrPolicyActivation.Error(), version, err.Error())
	}
	policyActivations := sortPolicyActivationsByDate(policyVersion.Activations)

	// just the first activations must correspond to the given properties: first we filter by network and status
	var activeProperties []string
	for _, act := range policyActivations {
		if act.Network == versionActivationNetwork &&
			act.PolicyInfo.Status == cloudlets.PolicyActivationStatusActive {
			activeProperties = append(activeProperties, act.PropertyInfo.Name)
		}
	}
	sort.Strings(activeProperties) // then we sort the filtered properties
	if reflect.DeepEqual(activeProperties, associatedProperties) {
		// 2. if the given version is active, just refresh status and quit
		logger.Debugf("policy %d, with version %d and properties [%s], is already active in %s. Fetching all details from server", policyID, version, strings.Join(associatedProperties, ", "), string(versionActivationNetwork))
		rd.SetId(formatPolicyActivationID(int64(policyID), cloudlets.PolicyActivationNetwork(network)))
		return resourcePolicyActivationRead(ctx, rd, m)
	}

	// 3. at this point, we are sure that the given version is not active -> use the clieNt to activate the policy version
	logger.Debugf("activating policy %d version %d, network %s and properties [%s]", policyID, version, string(versionActivationNetwork), strings.Join(associatedProperties, ", "))
	err = client.ActivatePolicyVersion(ctx, cloudlets.ActivatePolicyVersionRequest{
		PolicyID: int64(policyID),
		Version:  version,
		Async:    true,
		PolicyVersionActivation: cloudlets.PolicyVersionActivation{
			Network:                 versionActivationNetwork,
			AdditionalPropertyNames: associatedProperties,
		},
	})
	if err != nil {
		return diag.Errorf("%v create: %s", ErrPolicyActivation, err.Error())
	}

	// 4. wait until policy activation is done - we cannot continue until finished
	act, err := waitForPolicyActivation(ctx, client, int64(policyID), version, versionActivationNetwork, associatedProperties, nil)
	if err != nil {
		return diag.Errorf("%v create: %s", ErrPolicyActivation, err.Error())
	}
	rd.SetId(formatPolicyActivationID(act[0].PolicyInfo.PolicyID, act[0].Network)) // set id to the schema

	return resourcePolicyActivationRead(ctx, rd, m)
}

func resourcePolicyActivationRead(ctx context.Context, rd *schema.ResourceData, m interface{}) diag.Diagnostics {
	meta := akamai.Meta(m)
	logger := meta.Log("Cloudlets", "resourcePolicyActivationRead")
	ctx = session.ContextWithOptions(ctx, session.WithContextLog(logger))
	client := inst.Client(meta)

	logger.Debug("Reading policy activations")

        // reading input variables
	policyID, err := tools.GetIntValue("policy_id", rd)
	if err != nil {
		return diag.FromErr(err)
	}

	network, err := tools.GetStringValue("network", rd)
	if err != nil {
		return diag.FromErr(err)
	}
	net, err := getPolicyActivationNetwork(network)
	if err != nil {
		return diag.FromErr(err)
	}

	activations, err := client.ListPolicyActivations(ctx, cloudlets.ListPolicyActivationsRequest{
		PolicyID: int64(policyID),
		Network:  net,
	})
	if err != nil {
		return diag.Errorf("%v read: %s", ErrPolicyActivation, err.Error())
	}

	if len(activations) == 0 {
                // if there are no activations, error out
		return diag.Errorf("%v read: cannot find any activation for the given policy (%d) and network ('%s')", ErrPolicyActivation, policyID, net)
	}

        // sort by date: we just use the latest activation
	activations = sortPolicyActivationsByDate(activations)

        // setting the latest values to the local schema
	if err := rd.Set("status", activations[0].PolicyInfo.Status); err != nil {
		return diag.Errorf("%v: %s", tools.ErrValueSet, err.Error())
	}
	if err := rd.Set("version", activations[0].PolicyInfo.Version); err != nil {
		return diag.Errorf("%v: %s", tools.ErrValueSet, err.Error())
	}

	associatedProperties := getActiveProperties(activations)
	if err := rd.Set("associated_properties", associatedProperties); err != nil {
		return diag.Errorf("%v: %s", tools.ErrValueSet, err.Error())
	}

	return nil
}

func formatPolicyActivationID(policyID int64, network cloudlets.PolicyActivationNetwork) string {
	return fmt.Sprintf("%d:%s", policyID, network)
}

func getActiveProperties(policyActivations []cloudlets.PolicyActivation) []string {
	var activeProps []string
	for _, act := range policyActivations {
		if act.PolicyInfo.Status == cloudlets.PolicyActivationStatusActive {
			activeProps = append(activeProps, act.PropertyInfo.Name)
		}
	}
	sort.Strings(activeProps)
	return activeProps
}

// waitForPolicyActivation polls server until the activation has active status or until context is closed (because of timeout, cancellation or context termination)
//        - if I could rewrite this today, I would try to leverage context.Context instead. We were quite constrained about which patterns to use and not
func waitForPolicyActivation(ctx context.Context, client cloudlets.Cloudlets, policyID, version int64, network cloudlets.PolicyActivationNetwork, additionalProps, removedProperties []string) ([]cloudlets.PolicyActivation, error) {
	activations, err := client.ListPolicyActivations(ctx, cloudlets.ListPolicyActivationsRequest{
		PolicyID: policyID,
		Network:  network,
	})
	if err != nil {
		return nil, err
	}

        // we iterate in the following loop until all versions were active
	activations = filterActivations(activations, version, additionalProps)
	for len(activations) > 0 {
		allActive, allRemoved := true, true
	activations:
		for _, act := range activations {
			if act.PolicyInfo.Version == version {
				if act.PolicyInfo.Status == cloudlets.PolicyActivationStatusFailed {
					return nil, fmt.Errorf("%v: policyID %d: %s", ErrPolicyActivation, act.PolicyInfo.PolicyID, act.PolicyInfo.StatusDetail)
				}
				if act.PolicyInfo.Status != cloudlets.PolicyActivationStatusActive {
					allActive = false
					break
				}
			}
			for _, property := range removedProperties {
				if property == act.PropertyInfo.Name {
					allRemoved = false
					break activations
				}
			}
		}
		if allActive && allRemoved {
			return activations, nil
		}
		select {
		case <-time.After(tools.MaxDuration(ActivationPollInterval, ActivationPollMinimum)):
			activations, err = client.ListPolicyActivations(ctx, cloudlets.ListPolicyActivationsRequest{
				PolicyID: policyID,
				Network:  network,
			})
			if err != nil {
				return nil, err
			}
			activations = filterActivations(activations, version, additionalProps)

		case <-ctx.Done():
			if errors.Is(ctx.Err(), context.DeadlineExceeded) {
				return nil, ErrPolicyActivationTimeout
			}
			if errors.Is(ctx.Err(), context.Canceled) {
				return nil, ErrPolicyActivationCanceled
			}
			return nil, fmt.Errorf("%v: %w", ErrPolicyActivationContextTerminated, ctx.Err())
		}
	}

	if len(activations) == 0 {
		return nil, fmt.Errorf("%v: policyID %d: not all properties are active", ErrPolicyActivation, policyID)
	}

	return activations, nil
}

// filterActivations filters the latest activation for the given properties and version. In case of length mismatch (not all
// properties present in the last activation): it returns nil.
func filterActivations(activations []cloudlets.PolicyActivation, version int64, properties []string) []cloudlets.PolicyActivation {
	// inverse sorting by activation date -> first activations will be the most recent
	activations = sortPolicyActivationsByDate(activations)
	var lastActivationBlock []cloudlets.PolicyActivation
	var lastActivationDate int64
	// collect lastActivationBlock slice, with all activations sharing the latest activation date
	for _, act := range activations {
		// Each call to cloudlets.ActivatePolicyVersion() will result in a different activation date, and each activated
		// property will have the same activation date.
		if lastActivationDate != 0 && lastActivationDate != act.PolicyInfo.ActivationDate {
			break
		}
		lastActivationDate = act.PolicyInfo.ActivationDate
		lastActivationBlock = append(lastActivationBlock, act)
	}
	// find out if the all given properties were activated with the given policy version in last activation date
	allPropertiesActive := true
	for _, name := range properties {
		propertyPresent := false
		for _, act := range lastActivationBlock {
			if act.PropertyInfo.Name == name && act.PolicyInfo.Version == version {
				propertyPresent = true
				break
			}
		}
		if !propertyPresent {
			allPropertiesActive = false
			break
		}
	}
	if !allPropertiesActive {
		return nil
	}
	return lastActivationBlock
}

func sortPolicyActivationsByDate(activations []cloudlets.PolicyActivation) []cloudlets.PolicyActivation {
	sort.Slice(activations, func(i, j int) bool {
		return activations[i].PolicyInfo.ActivationDate > activations[j].PolicyInfo.ActivationDate
	})
	return activations
}

func getPolicyActivationNetwork(net string) (cloudlets.PolicyActivationNetwork, error) {

	net = tools.StateNetwork(net)

	switch net {
	case "production":
		return cloudlets.PolicyActivationNetworkProduction, nil
	case "staging":
		return cloudlets.PolicyActivationNetworkStaging, nil
	}

	return "", ErrNetworkName
}

func statePolicyActivationNetwork(i interface{}) string {

	net := tools.StateNetwork(i)

	switch net {
	case "production":
		return string(cloudlets.PolicyActivationNetworkProduction)
	case "staging":
		return string(cloudlets.PolicyActivationNetworkStaging)
	}

	// this should never happen :-)
	return net
}

func syncToServerRemovedProperties(ctx context.Context, logger log.Interface, client cloudlets.Cloudlets, policyID int64, network cloudlets.PolicyActivationNetwork, activeProps, newPolicyProperties []string) ([]string, error) {
	policyProperties, err := client.GetPolicyProperties(ctx, cloudlets.GetPolicyPropertiesRequest{PolicyID: policyID})
	if err != nil {
		return nil, fmt.Errorf("%w: cannot find policy %d properties: %s", ErrPolicyActivation, policyID, err.Error())
	}
	removedProperties := make([]string, 0)
activePropertiesLoop:
	for _, activeProp := range activeProps {
		for _, newProp := range newPolicyProperties {
			if activeProp == newProp {
				continue activePropertiesLoop
			}
		}
		// find out property id
		associateProperty, ok := policyProperties[activeProp]
		if !ok {
			logger.Warnf("Policy %d server side discrepancies: '%s' is not present in GetPolicyProperties response", policyID, activeProp)
			continue activePropertiesLoop
		}
		propertyID := associateProperty.ID

		// wait for removal until there aren't any pending activations
		if err = waitForNotPendingPolicyActivation(ctx, logger, client, policyID, network); err != nil {
			return nil, err
		}

		// remove property from policy
		logger.Debugf("proceeding to delete property '%s' from policy (ID=%d)", activeProp, policyID)
		if err := client.DeletePolicyProperty(ctx, cloudlets.DeletePolicyPropertyRequest{PolicyID: policyID, PropertyID: propertyID, Network: network}); err != nil {
			return nil, fmt.Errorf("%w: cannot remove policy %d property %d and network '%s'. Please, try once again later.\n%s", ErrPolicyActivation, policyID, propertyID, network, err.Error())
		}
		removedProperties = append(removedProperties, activeProp)
	}

	// wait for removal until there aren't any pending activations
	if err = waitForNotPendingPolicyActivation(ctx, logger, client, policyID, network); err != nil {
		return nil, err
	}

	// at this point, there are no activations in pending state
	return removedProperties, nil
}

func waitForNotPendingPolicyActivation(ctx context.Context, logger log.Interface, client cloudlets.Cloudlets, policyID int64, network cloudlets.PolicyActivationNetwork) error {
	logger.Debugf("waiting until there none of the policy (ID=%d) activations are in pending state", policyID)
	activations, err := client.ListPolicyActivations(ctx, cloudlets.ListPolicyActivationsRequest{PolicyID: policyID})
	if err != nil {
		return fmt.Errorf("%w: failed to list policy activations for policy %d: %s", ErrPolicyActivation, policyID, err.Error())
	}
	for len(activations) > 0 {
		pending := false
		for _, act := range activations {
			if act.PolicyInfo.Status == cloudlets.PolicyActivationStatusFailed {
				return fmt.Errorf("%v: policyID %d: %s", ErrPolicyActivation, act.PolicyInfo.PolicyID, act.PolicyInfo.StatusDetail)
			}
			if act.PolicyInfo.Status == cloudlets.PolicyActivationStatusPending {
				pending = true
				break
			}
		}
		if !pending {
			break
		}
		select {
		case <-time.After(tools.MaxDuration(ActivationPollInterval, ActivationPollMinimum)):
			activations, err = client.ListPolicyActivations(ctx, cloudlets.ListPolicyActivationsRequest{
				PolicyID: policyID,
				Network:  network,
			})
			if err != nil {
				return fmt.Errorf("%w: failed to list policy activations for policy %d: %s", ErrPolicyActivation, policyID, err.Error())
			}

		case <-ctx.Done():
			if errors.Is(ctx.Err(), context.DeadlineExceeded) {
				return ErrPolicyActivationTimeout
			}
			if errors.Is(ctx.Err(), context.Canceled) {
				return ErrPolicyActivationCanceled
			}
			return fmt.Errorf("%v: %w", ErrPolicyActivationContextTerminated, ctx.Err())
		}
	}

	return nil
}
```

## Akamai CLI

### background / introduction

- Akamai CLI is just a binary
  - We can install other modules available on github
  - Those modules can be: binary, node.js or python (also PHP and Ruby modules but almost nobody was using them)
- Here I reimplement the logic for installing and handling python modules
  - Previous approach was installing all dependencies globally
    - Dependencies of different installed modules were overlapping each other -> CONFLICTS
  - My refactoring leveraged python virtual environment concept
    - This was my idea, nobody in Akamai had thought about this approach
    - It was initially received with skepticism from architects and former engineers
- to my current taste, I would make better use of Public/private methods (Visible/not visible from outside)
  - it is efficient, it fixed all old and new issues with CLI python modules at once
    - after I left 3 years ago, it's barely been touched -> it works very well
    - my latest changes date from April 2022
  - but some parts are a little bit chunky: either
    - I would have done that slightly different today
      - not enough refactoring?
    - After several patches, the code may smell a little bit
      - in any case: `langManager` is used to support different programming languages + different operating systems and architectures

### detailed explanation

- `langManager` is a struct defined in `cli/packages/packages.go`
  - Implemented to support the different languages
  - Used internally to install, use, upgrade, remove other packages
  - Points of entry
    - `validatePythonDeps` -> from package.go, we check if the client environment matches all Python dependencies
    - `setup` -> setting up the module virtual environment with required python packages
    - `installPython` -> main point of entry: it calls the above two to install all the module requirements
    - All other methods are not used outside of this file

- [Source|https://github.com/akamai/cli/blob/master/pkg/packages/python.go]

```go
// Copyright 2018. Akamai Technologies, Inc
// [...]

package packages

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"log/slog"
	"os"
	"os/exec"
	"path/filepath"
	"regexp"
	"strings"

	"github.com/akamai/cli/v2/pkg/log"
	"github.com/akamai/cli/v2/pkg/terminal"
	"github.com/akamai/cli/v2/pkg/tools"
	"github.com/akamai/cli/v2/pkg/version"
)

var (
	pythonVersionPattern = `Python ([2,3]\.\d+\.\d+).*`
	pythonVersionRegex   = regexp.MustCompile(pythonVersionPattern)
	pipVersionPattern    = `^pip \d{1,2}\..+ \(python [2,3]\.\d+\)`
	venvHelpPattern      = `usage: venv `
	pipVersionRegex      = regexp.MustCompile(pipVersionPattern)
	venvHelpRegex        = regexp.MustCompile(venvHelpPattern)
)

// main point of entry for langManager - it orchestrates installation
func (l *langManager) installPython(ctx context.Context, venvPath, srcPath, requiredPy string) error {
	logger := log.FromContext(ctx)
	logger.Debug("Starting Python installation")

        // Validation of client environment
	pythonBin, pipBin, err := l.validatePythonDeps(ctx, logger, requiredPy, filepath.Base(srcPath))
	if err != nil {
		logger.Error(fmt.Sprintf("Dependency validation failed: %v", err))
		return fmt.Errorf("unable to validate python dependency: %v", err)
	}

        // proceed by setting up the virtual environment
	if err = l.setup(ctx, venvPath, srcPath, pythonBin, pipBin, requiredPy, false); err != nil {
		logger.Error(fmt.Sprintf("Setup failed: %v", err))
		return fmt.Errorf("unable to setup python virutalenv for the given module: %v", err)
	}

	return nil
}

// setup does the python virtualenv set up for the given module. It may return an error.
func (l *langManager) setup(ctx context.Context, pkgVenvPath, srcPath, python3Bin, pipBin, requiredPy string, passthru bool) error {
	logger := log.FromContext(ctx)
	logger.Debug("Setting up Python environment")

	switch version.Compare(requiredPy, "3.0.0") {
	case version.Greater, version.Equals:
		// Python 3.x required: build virtual environment
		defer func() {
			if !passthru { // under some circumstances, we needed to bypass the VE
				l.deactivateVirtualEnvironment(ctx, pkgVenvPath, requiredPy)
			}
			logger.Debug("All virtualenv dependencies successfully installed")
		}()

		veExists, err := l.commandExecutor.FileExists(pkgVenvPath)
		if err != nil {
			logger.Error(fmt.Sprintf("Failed to check package virtualenv existence: %v", err))
			return err
		}

		if !passthru || !veExists {
			logger.Debug(fmt.Sprintf("The virtual environment %s does not exist yet - installing dependencies", pkgVenvPath))

			// upgrade pip and setuptools
			if err := l.upgradePipAndSetuptools(ctx, python3Bin); err != nil {
				return err
			}

			// create virtual environment
			if err := l.createVirtualEnvironment(ctx, python3Bin, pkgVenvPath); err != nil {
				return err
			}
		}

		// activate virtual environment
		if err := l.activateVirtualEnvironment(ctx, pkgVenvPath); err != nil {
			return err
		}

		// install packages from requirements.txt
		vePy, err := l.getVePython(pkgVenvPath)
		if err != nil {
			logger.Error(fmt.Sprintf("Failed to get virtualenv python executable: %v", err))
			return err
		}
		if err := l.installVeRequirements(ctx, srcPath, pkgVenvPath, vePy); err != nil {
			return err
		}
	case version.Smaller:
		// no virtualenv for python 2.x -> unsupported: we fall back to the old installation method
                // I left this because any client may have their own python 2 module (quite likely, a big one). And we do not like to annoy those customers.
		return installPythonDepsPip(ctx, l.commandExecutor, pipBin, srcPath)
	}

	return nil
}

func (l *langManager) getVePython(vePath string) (string, error) {
	switch l.GetOS() {
	case "windows":
		return filepath.Join(vePath, "System", "python.exe"), nil
	case "linux", "darwin":
		return filepath.Join(vePath, "bin", "python"), nil
	default:
		return "", ErrOSNotSupported
	}
}

func (l *langManager) installVeRequirements(ctx context.Context, srcPath, vePath, py3Bin string) error {
	logger := log.FromContext(ctx)

	requirementsPath := filepath.Join(srcPath, "requirements.txt")
	if ok, _ := l.commandExecutor.FileExists(requirementsPath); !ok {
		logger.Error("requirements.txt not found")
		return ErrRequirementsTxtNotFound
	}
	logger.Info("requirements.txt found, running pip package manager")

	shell, err := l.GetShell(l.GetOS())
	if err != nil {
		logger.Error("cannot determine OS shell")
		return fmt.Errorf("unable to determine OS shell: %v", err)
	}
	if shell == "" {
		// windows
		pipPath := filepath.Join(vePath, "Scripts", "pip.exe")
		if output, err := l.commandExecutor.ExecCommand(&exec.Cmd{
			Path: pipPath,
			Args: []string{pipPath, "install", "--upgrade", "--ignore-installed", "-r", requirementsPath},
		}, true); err != nil {
			term := terminal.Get(ctx)
			_, _ = term.Writeln(string(output))
			logger.Error("failed to run pip install --upgrade --ignore-installed -r requirements.txt")
			return fmt.Errorf("%w: %s", ErrRequirementsInstall, string(output))
		}
		logger.Debug("Python virtualenv requirements successfully installed")

		return nil
	}

	if output, err := l.commandExecutor.ExecCommand(&exec.Cmd{
		Path: py3Bin,
		Args: []string{py3Bin, "-m", "pip", "install", "--upgrade", "--ignore-installed", "-r", requirementsPath},
	}, true); err != nil {
		logger.Error("failed to run pip install --upgrade --ignore-installed -r requirements.txt")
		logger.Error(string(output))
		return fmt.Errorf("%w: %v", ErrRequirementsInstall, string(output))
	}

	logger.Debug("Python virtualenv requirements successfully installed")

	return nil
}

/*
validatePythonDeps does system dependencies validation based on the required python version

It returns:

* route to required python executable

* route to pip executable, for modules which require python < v3

* error, if any
*/
func (l *langManager) validatePythonDeps(ctx context.Context, logger *slog.Logger, requiredPy, name string) (string, string, error) {
        // here we compare the python version required by the module being installed with the user environment
	switch version.Compare(requiredPy, "3.0.0") {
	case version.Smaller:
		// v2 required -> no virtualenv
		logger.Debug("Validating dependencies for python 2.x module")
		pythonBin, err := findPythonBin(ctx, l.commandExecutor, requiredPy, "")
		if err != nil {
			logger.Error("Python >= 2 (and < 3.0) not found in the system. Please verify your setup")
			return "", "", err
		}

		if err := l.resolveBinVersion(pythonBin, requiredPy, "--version", logger); err != nil {
			return "", "", err
		}

		pipBin, err := findPipBin(ctx, l.commandExecutor, requiredPy)
		if err != nil {
			logger.Error("Pip not found for python 2.x module. Please verify your setup")
			return pythonBin, "", err
		}
		return pythonBin, pipBin, nil
	case version.Greater, version.Equals:
		// v3 required -> virtualenv
		// requirements for setting up VE: python3, pip3, venv
		logger.Debug(fmt.Sprintf("Validating dependencies for python %s module", requiredPy))
		pythonBin, err := findPythonBin(ctx, l.commandExecutor, requiredPy, name)
		if err != nil {
			logger.Error(fmt.Sprintf("Python >= %s not found in the system. Please verify your setup", requiredPy))
			return "", "", err
		}

		if err := l.resolveBinVersion(pythonBin, requiredPy, "--version", logger); err != nil {
			return "", "", err
		}

		// validate that the use has python pip package installed
		if err = l.findPipPackage(ctx, requiredPy, pythonBin); err != nil {
			logger.Error("Pip not found in the system. Please verify your setup")
			return "", "", err
		}

		// validate that venv module is present
		if err = l.findVenvPackage(ctx, pythonBin); err != nil {
			logger.Error("Python venv module not found in the system. Please verify your setup")
			return "", "", err
		}

		return pythonBin, "", nil
	default:
		// not supported
		logger.Error(fmt.Sprintf("%s: %s", ErrPythonVersionNotSupported.Error(), requiredPy))
		return "", "", fmt.Errorf("%w: %s", ErrPythonVersionNotSupported, requiredPy)
	}
}

// this method resolves if `venv` is within the system python packages
func (l *langManager) findVenvPackage(ctx context.Context, pythonBin string) error {
	logger := log.FromContext(ctx)
	cmd := exec.Command(pythonBin, "-m", "venv", "--version")
	output, _ := l.commandExecutor.ExecCommand(cmd, true)
	logger.Debug(fmt.Sprintf("%s %s: %s", pythonBin, "-m venv --version", bytes.ReplaceAll(output, []byte("\n"), []byte(""))))
	matches := venvHelpRegex.FindStringSubmatch(string(output))
	if len(matches) == 0 {
		return fmt.Errorf("%w: %s", ErrVenvNotFound, bytes.ReplaceAll(output, []byte("\n"), []byte("")))
	}
	return nil
}

// this resolves if `pip` is within the system python packages
func (l *langManager) findPipPackage(ctx context.Context, requiredPy string, pythonBin string) error {
	compare := version.Compare(requiredPy, "3.0.0")
	if compare == version.Greater || compare == version.Equals {
		logger := log.FromContext(ctx)

		// find pip python package, not pip executable
		cmd := exec.Command(pythonBin, "-m", "pip", "--version")
		output, _ := l.commandExecutor.ExecCommand(cmd, true)
		logger.Debug(fmt.Sprintf("%s %s: %s", pythonBin, "-m pip --version", bytes.ReplaceAll(output, []byte("\n"), []byte(""))))
		matches := pipVersionRegex.FindStringSubmatch(string(output))
		if len(matches) == 0 {
			return fmt.Errorf("%w: %s", ErrPipNotFound, bytes.ReplaceAll(output, []byte("\n"), []byte("")))
		}
	}

	return nil
}

// the name is self explanatory - it depends on the user OS (windows?)
func (l *langManager) activateVirtualEnvironment(ctx context.Context, pkgVenvPath string) error {
	logger := log.FromContext(ctx)
	logger.Debug(fmt.Sprintf("Activating Python virtualenv: %s", pkgVenvPath))
	oS := l.GetOS()
	interpreter, err := l.GetShell(oS)
	if err != nil {
		logger.Error("cannot determine OS shell")
		return fmt.Errorf("unable to determine OS shell: %v", err)
	}
	cmd := &exec.Cmd{}
	if oS == "windows" {
		activate := filepath.Join(pkgVenvPath, "Scripts", "activate.bat")
		cmd.Path = activate
		cmd.Args = []string{}
	} else {
		cmd.Path = interpreter
		cmd.Args = []string{"source", filepath.Join(pkgVenvPath, "bin", "activate")}
	}
	if output, err := l.commandExecutor.ExecCommand(cmd, true); err != nil {
		logger.Error(fmt.Sprintf("%v: %v", ErrVirtualEnvActivation, string(output)))
		return fmt.Errorf("%w: %s", ErrVirtualEnvActivation, string(output))
	}
	logger.Debug(fmt.Sprintf("Python virtualenv %s active", pkgVenvPath))

	return nil
}

// similar to the method above
func (l *langManager) deactivateVirtualEnvironment(ctx context.Context, dir, pyVersion string) {
	compare := version.Compare(pyVersion, "3.0.0")
	if compare == version.Equals || compare == version.Greater {
		logger := log.FromContext(ctx)
		logger.Debug(fmt.Sprintf("Deactivating virtual environment %s", dir))
		cmd := &exec.Cmd{}
		oS := l.GetOS()
		if oS == "windows" {
			logger.Debug("windows detected, executing deactivate.bat")
			deactivate := filepath.Join(dir, "Scripts", "deactivate.bat")
			cmd.Path = deactivate
			cmd.Args = []string{deactivate}
		} else {
			cmd.Path, _ = l.GetShell(oS)
			cmd.Args = []string{"deactivate"}
		}
		// errors are ignored at this step, as we may be trying to deactivate a non existent or not active virtual environment
		if _, err := l.commandExecutor.ExecCommand(cmd, true); err == nil {
			logger.Debug("Python virtualenv deactivated")
		} else {
			logger.Debug(fmt.Sprintf("Error deactivating VE %s: %v", dir, err))
		}
	}
}

// dealing with python versions
func (l *langManager) resolveBinVersion(bin, cmdReq, arg string, logger *slog.Logger) error {
	cmd := exec.Command(bin, arg)
	logger.Debug("Resolving python version")

	output, err := l.commandExecutor.ExecCommand(cmd, true)
	if err != nil {
		logger.Error(fmt.Sprintf("Failed to execute %s command: %v", cmd.Path, err))
		return fmt.Errorf("command %s execution failed: %v", cmd.Path, err)
	}
	logger.Debug(fmt.Sprintf("%s %s: %s", bin, arg, bytes.ReplaceAll(output, []byte("\n"), []byte(""))))

	matches := pythonVersionRegex.FindStringSubmatch(string(output))
	if len(matches) < 2 {
		return fmt.Errorf("%w: %s: %s", ErrRuntimeNoVersionFound, "python", cmd)
	}
	switch version.Compare(cmdReq, matches[1]) {
	case version.Greater:
		// python required > python installed
		logger.Error(fmt.Sprintf("%s version found: %s", bin, matches[1]))
		return fmt.Errorf("%w: required: %s:%s, have: %s. Please install the required Python branch", ErrRuntimeMinimumVersionRequired, bin, cmdReq, matches[1])
	case version.Smaller:
		// python required < python installed
		if version.Compare(cmdReq, "3.0.0") == 1 && version.Compare(matches[1], "3.0.0") <= 0 {
			// required: py2; found: py3. The user still needs to install py2
			logger.Error(fmt.Sprintf("Python version %s found, %s version required. Please, install the %s Python branch.", matches[1], cmdReq, cmdReq))
			return fmt.Errorf("%w: Please install the following Python branch: %s", ErrRuntimeNotFound, cmdReq)
		}
		return nil
	case version.Equals:
		return nil
	}

	return ErrPythonVersionNotSupported
}

// used for the CLI install and upgrade commands
func (l *langManager) upgradePipAndSetuptools(ctx context.Context, python3Bin string) error {
	logger := log.FromContext(ctx)

	// if python3 > v3.4, ensure pip
	if err := l.resolveBinVersion(python3Bin, "3.4.0", "--version", logger); err != nil && errors.Is(err, ErrRuntimeNotFound) {
		return err
	}

	logger.Debug("Installing/upgrading pip")

	// ensure pip is present
	cmdPip := exec.Command(python3Bin, "-m", "ensurepip", "--upgrade")
	if output, err := l.commandExecutor.ExecCommand(cmdPip, true); err != nil {
		logger.Warn(fmt.Sprintf("%v: %s", ErrPipUpgrade, string(output)))
	}

	// upgrade pip & setuptools
	logger.Debug("Installing/upgrading pip & setuptools")
	cmdSetuptools := exec.Command(python3Bin, "-m", "pip", "install" /*, "--user"*/, "--no-cache", "--upgrade", "pip", "setuptools")
	if output, err := l.commandExecutor.ExecCommand(cmdSetuptools, true); err != nil {
		logger.Error(fmt.Sprintf("%v: %s", ErrPipSetuptoolsUpgrade, string(output)))
		return fmt.Errorf("%w: %s", ErrPipSetuptoolsUpgrade, string(output))
	}
	logger.Debug("pip & setuptools successfully installed/upgraded")

	return nil
}

// self explanatory - used for CLI install (also for upgrading from old CLI versions without VE management)
func (l *langManager) createVirtualEnvironment(ctx context.Context, python3Bin string, pkgVenvPath string) error {
	logger := log.FromContext(ctx)

	// check if the .akamai-cli/venv directory exists - create it otherwise
	venvPath := filepath.Dir(pkgVenvPath)
	if exists, err := l.commandExecutor.FileExists(venvPath); err == nil && !exists {
		logger.Debug(fmt.Sprintf("%s does not exist; let's create it", venvPath))
		if err := os.Mkdir(venvPath, 0755); err != nil {
			logger.Error(fmt.Sprintf("%v %s: %v", ErrDirectoryCreation, venvPath, err))
			return fmt.Errorf("%w %s: %v", ErrDirectoryCreation, venvPath, err)
		}
		logger.Debug(fmt.Sprintf("%s directory created", venvPath))
	} else {
		if err != nil {
			logger.Error(fmt.Sprintf("Failed to check package virtualenv existence: %v", err))
			return err
		}
	}

	logger.Debug(fmt.Sprintf("Creating python virtualenv: %s", pkgVenvPath))
	cmdVenv := exec.Command(python3Bin, "-m", "venv", pkgVenvPath)
	if output, err := l.commandExecutor.ExecCommand(cmdVenv, true); err != nil {
		logger.Error(fmt.Sprintf("%v %s: %s", ErrVirtualEnvCreation, pkgVenvPath, string(output)))
		return fmt.Errorf("%w %s: %s", ErrVirtualEnvCreation, pkgVenvPath, string(output))
	}
	logger.Debug(fmt.Sprintf("Python virtualenv successfully created: %s", pkgVenvPath))

	return nil
}

func findPythonBin(ctx context.Context, cmdExecutor executor, ver, name string) (string, error) {
	logger := log.FromContext(ctx)
	logger.Debug("Looking for python binaries")

	var err error
	var bin string

	defer func() {
		if err == nil {
			logger.Debug(fmt.Sprintf("Python binary found: %s", bin))
		}
	}()
	if version.Compare("3.0.0", ver) != version.Greater {
		// looking for python3 or py (windows)
		bin, err = lookForBins(cmdExecutor, "python3", "python3.exe", "py.exe")
		if err != nil {
			return "", fmt.Errorf("%w: %s. Please verify if the executable is included in your PATH", ErrRuntimeNotFound, "python 3")
		}
		vePath, _ := tools.GetPkgVenvPath(name)
		if _, err := os.Stat(vePath); !os.IsNotExist(err) {
			if cmdExecutor.GetOS() == "windows" {
				bin = filepath.Join(vePath, "Scripts", "python.exe")
			} else {
				bin = filepath.Join(vePath, "bin", "python")
			}
		}
		return bin, nil
	}
	if version.Compare("2.0.0", ver) != version.Greater {
		// looking for python2 or py (windows) - no virtualenv
		bin, err = lookForBins(cmdExecutor, "python2", "python2.exe", "py.exe")
		if err != nil {
			return "", fmt.Errorf("%w: %s. Please verify if the executable is included in your PATH", ErrRuntimeNotFound, "python 2")
		}
		return bin, nil
	}
	// looking for any version
	bin, err = lookForBins(cmdExecutor, "python2", "python", "python3", "py.exe", "python.exe")
	if err != nil {
		return "", fmt.Errorf("%w: %s. Please verify if the executable is included in your PATH", ErrRuntimeNotFound, "python")
	}
	return bin, nil
}

func findPipBin(ctx context.Context, cmdExecutor executor, requiredPy string) (string, error) {
	logger := log.FromContext(ctx)

	var bin string
	var err error
	defer func() {
		if err == nil {
			logger.Debug(fmt.Sprintf("Pip binary found: %s", bin))
		}
	}()
	switch version.Compare(requiredPy, "3.0.0") {
	case version.Greater, version.Equals:
		bin, err = lookForBins(cmdExecutor, "pip3", "pip3.exe")
		if err != nil {
			return "", fmt.Errorf("%w: %s", ErrPackageManagerNotFound, "pip3")
		}
	case version.Smaller:
		bin, err = lookForBins(cmdExecutor, "pip2")
		if err != nil {
			return "", fmt.Errorf("%w, %s", ErrPackageManagerNotFound, "pip2")
		}
	}

	return bin, nil
}

func installPythonDepsPip(ctx context.Context, cmdExecutor executor, bin, dir string) error {
	logger := log.FromContext(ctx)

	if ok, _ := cmdExecutor.FileExists(filepath.Join(dir, "requirements.txt")); !ok {
		logger.Debug("requirements.txt not found")
		return nil
	}
	logger.Info("requirements.txt found, running pip package manager")

	if err := os.Setenv("PYTHONUSERBASE", dir); err != nil {
		logger.Error(fmt.Sprintf("Failed to set environment for the key %s in %s: %v", "PYTHONUSERBASE", dir, err))
		return err
	}
	args := []string{bin, "install", "--user", "--ignore-installed", "-r", filepath.Join(dir, "requirements.txt")}
	cmd := exec.Command(args[0], args[1:]...)
	cmd.Dir = dir
	if _, err := cmdExecutor.ExecCommand(cmd); err != nil {
		var exitErr *exec.ExitError
		if errors.As(err, &exitErr) {
			logger.Debug(fmt.Sprintf("Unable execute package manager (PYTHONUSERBASE=%s %s): \n %s", dir, strings.Join(args, " "), exitErr.Stderr))
		}
		return fmt.Errorf("%w: %s. Please verify pip system dependencies (setuptools, python3-dev, gcc, libffi-dev, openssl-dev)", ErrPackageManagerExec, "pip")
	}
	logger.Debug(fmt.Sprintf("Python dependencies successfully installed using pip: %s", strings.Join(args, " ")))

	return nil
}

func lookForBins(cmdExecutor executor, bins ...string) (string, error) {
	var err error
	var bin string
	for _, binName := range bins {
		bin, err = cmdExecutor.LookPath(binName)
		if err == nil {
			return bin, nil
		}
	}
	return bin, err
}
```
