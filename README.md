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
# Most cost recommenders are global or don't require explicit location at the project level for listing.
# If a recommender consistently fails without a location, you might add relevant locations here.
# Example: LOCATIONS_TO_CHECK=("global" "us-central1" "europe-west1" "asia-south1")
# For many cost recommenders, not specifying --location works best as it covers global and relevant regional ones.
LOCATIONS_TO_CHECK=("--location=global") # Default to global, gcloud handles many cases without explicit location.
                                          # You can expand this array if needed, e.g. ("--location=global" "--location=us-central1")

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

    # Ensure the Recommender API is enabled for the project (optional check, gcloud will error out if not)
    # Consider enabling it manually if the script fails here for multiple projects:
    # gcloud services enable recommender.googleapis.com --project="$PROJECT_ID"

    HAS_RECOMMENDATIONS_FOR_PROJECT=false

    # Loop through each cost recommender
    for RECOMMENDER_ID in "${COST_RECOMMENDERS[@]}"; do
        echo "  Checking Recommender: $RECOMMENDER_ID"

        # Attempt to list recommendations (often global, or gcloud handles location context)
        # Forcing a specific location might be needed if the default doesn't work.
        # However, many cost recommenders operate globally or across relevant regions by default.

        # First, try without an explicit location (or with --location=global)
        # The --location flag for `gcloud recommender recommendations list` can be tricky
        # as not all recommenders support all locations or the 'global' keyword in the same way.
        # Often, omitting it for project-level queries lets gcloud pick the correct scope.
        # If you need to check specific locations, uncomment and adapt the loop below.

        COMMAND="gcloud recommender recommendations list --project=\"${PROJECT_ID}\" --recommender=\"${RECOMMENDER_ID}\" --format=\"yaml(description,lastRefreshTime,primaryImpact.costProjection.cost.units,primaryImpact.costProjection.cost.nanos,primaryImpact.costProjection.duration,name,stateInfo.state,content.operationGroups[0].operations[0].resource)\""

        # Some recommenders might be location-specific.
        # If a recommender typically requires a location and the above doesn't work,
        # you can iterate through a list of common locations.
        # For most cost recommenders, a single call (global or no location specified) is sufficient.

        # Example for checking specific locations:
        # for LOCATION_FLAG in "${LOCATIONS_TO_CHECK[@]}"; do
        #   echo "    Attempting with ${LOCATION_FLAG}..."
        #   RECOMMENDATIONS=$(gcloud recommender recommendations list \
        #       --project="${PROJECT_ID}" \
        #       --recommender="${RECOMMENDER_ID}" \
        #       ${LOCATION_FLAG} \
        #       --filter="stateInfo.state=ACTIVE" \
        #       --format="yaml(description,lastRefreshTime,primaryImpact.costProjection.cost,name,stateInfo.state)" 2>/dev/null)
        #   # ... (rest of the logic)
        # done

        RECOMMENDATIONS=$(eval "$COMMAND 2>/dev/null") # Suppress errors if recommender not found or no recommendations

        if [ -n "$RECOMMENDATIONS" ]; then
            ACTIVE_RECOMMENDATIONS=$(echo "$RECOMMENDATIONS" | grep "state: ACTIVE" -B5 -A5) # Basic filter for active ones, adjust as needed
            if [ -n "$ACTIVE_RECOMMENDATIONS" ]; then
                echo "    Active Recommendations Found:"
                echo "$RECOMMENDATIONS" # Print the full YAML for active recommendations
                echo "    -----------------------------------"
                HAS_RECOMMENDATIONS_FOR_PROJECT=true
            else
                echo "    No active recommendations found for $RECOMMENDER_ID."
            fi
        else
            # This can also mean the recommender is not applicable or enabled for the project/location combination.
             # Check if the API is enabled, though gcloud usually gives a specific error for that.
            API_ENABLED_CHECK=$(gcloud services list --project="$PROJECT_ID" --enabled --filter="config.name:recommender.googleapis.com" --format="value(config.name)")
            if [[ -z "$API_ENABLED_CHECK" ]]; then
                 echo "    Note: recommender.googleapis.com API might be disabled for project $PROJECT_ID."
            else
                 echo "    No recommendations found or recommender not applicable for $RECOMMENDER_ID."
            fi
        fi
    done

    if ! $HAS_RECOMMENDATIONS_FOR_PROJECT; then
        echo "  No cost recommendations found for project $PROJECT_ID with the checked recommenders."
    fi

done <<< "$PROJECT_IDS"

echo ""
echo "============================================="
echo "Finished fetching recommendations."
