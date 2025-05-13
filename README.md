# scripts

#!/bin/bash

# Define a list of common cost-related recommender IDs
# You can find more recommender IDs at:
# https://cloud.google.com/recommender/docs/recommenders
COST_RECOMMENDERS=(
    "google.compute.instance.IdleResourceRecommender"
    "google.compute.disk.IdleResourceRecommender"
    "google.compute.address.IdleResourceRecommender"
    "google.compute.image.IdleResourceRecommender"
    "google.compute.instance.MachineTypeRecommender" # Rightsizing recommendations
    "google.compute.instanceGroupManager.MachineTypeRecommender"
    "google.cloudsql.instance.IdleRecommender"
    "google.cloudsql.instance.OverprovisionedRecommender"
    "google.resourcemanager.projectUtilization.Recommender" # Unattended projects
    "google.billing.commitment.SpendBasedCommitmentRecommender" # CUDs for spend-based
    "google.compute.commitment.UsageCommitmentRecommender" # CUDs for resource-based
    "google.cloudrun.service.CostRecommender"
    # Add or remove recommenders as needed
)

# Optional: Define specific locations to check if global/default doesn't work for some recommenders.
LOCATIONS_TO_CHECK=("--location=global") # Default to global, gcloud handles many cases without explicit location.

echo "Fetching Active Assist Cost Recommendations..."
echo "============================================="

# Get a list of all project IDs
PROJECT_IDS=$(gcloud projects list --format="value(projectId)")

if [ -z "$PROJECT_IDS" ]; then
    echo "No projects found or you may not have permissions to list projects."
    exit 1
fi

# Loop through each project
while IFS= read -r PROJECT_ID; do
    echo ""
    echo "---------------------------------------------"
    echo "Project: $PROJECT_ID"
    echo "---------------------------------------------"

    HAS_RECOMMENDATIONS_FOR_PROJECT=false

    # Loop through each cost recommender
    for RECOMMENDER_ID in "${COST_RECOMMENDERS[@]}"; do
        echo "  Checking Recommender: $RECOMMENDER_ID"

        # Command to list recommendations
        COMMAND="gcloud recommender recommendations list --project=\"${PROJECT_ID}\" --recommender=\"${RECOMMENDER_ID}\" --format=\"yaml(description,lastRefreshTime,primaryImpact.costProjection.cost.units,primaryImpact.costProjection.cost.nanos,primaryImpact.costProjection.duration,name,stateInfo.state,content.operationGroups[0].operations[0].resource)\""

        RECOMMENDATIONS=$(eval "$COMMAND 2>/dev/null") # Suppress errors if recommender not found or no recommendations

        if [ -n "$RECOMMENDATIONS" ]; then
            ACTIVE_RECOMMENDATIONS=$(echo "$RECOMMENDATIONS" | grep "state: ACTIVE" -B5 -A5) # Basic filter for active ones
            if [ -n "$ACTIVE_RECOMMENDATIONS" ]; then
                echo "    Active Recommendations Found:"
                echo "$RECOMMENDATIONS" # Print the full YAML for active recommendations
                echo "    -----------------------------------"
                HAS_RECOMMENDATIONS_FOR_PROJECT=true
            else
                echo "    No active recommendations found for $RECOMMENDER_ID."
            fi
        else
            echo "    No recommendations found or recommender not applicable for $RECOMMENDER_ID."
        fi
    done

    if ! $HAS_RECOMMENDATIONS_FOR_PROJECT; then
        echo "  No cost recommendations found for project $PROJECT_ID with the checked recommenders."
    fi

done <<< "$PROJECT_IDS"

echo ""
echo "============================================="
echo "Finished fetching recommendations."
