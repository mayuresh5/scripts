# scripts

#!/bin/bash

# Exit on error, treat unset variables as an error, and ensure pipeline failures are caught
set -euo pipefail

# --- Configuration ---
# Default location to check, in addition to 'global'.
# You can change this to your primary operational region.
DEFAULT_LOCATION="asia-south1"
TARGET_LOCATION="${1:-$DEFAULT_LOCATION}" # Use first argument as location, or default

# Array of cost-related recommender IDs
# Based on GCP documentation for cost optimization and Active Assist.
# Note: You mentioned 'google.container.Diagnosis.Recommender'.
# Documentation often lists it as 'google.container.DiagnosisRecommender'.
# The script uses the latter, but you can adjust if your specific variant works.
COST_RECOMMENDERS=(
    "google.compute.instance.IdleResourceRecommender"           # Idle VMs
    "google.compute.disk.IdleResourceRecommender"               # Idle persistent disks
    "google.compute.address.IdleResourceRecommender"            # Idle IP addresses
    "google.compute.image.IdleResourceRecommender"              # Idle custom images
    "google.compute.commitment.UsageCommitmentRecommender"      # Resource-based CUDs
    "google.cloudbilling.commitment.SpendBasedCommitmentRecommender" # Spend-based CUDs
    "google.cloudsql.instance.IdleRecommender"                  # Cloud SQL idle instances
    "google.cloudsql.instance.OverprovisionedRecommender"       # Cloud SQL overprovisioned instances
    "google.run.service.CostRecommender"                        # Cloud Run CPU allocation
    "google.container.DiagnosisRecommender"                     # Idle GKE clusters (adjust if your variant google.container.Diagnosis.Recommender is preferred)
    "google.bigquery.capacityCommitments.Recommender"           # BigQuery slot commitments
    "google.resourcemanager.projectUtilization.Recommender"     # Unused/unattended projects
    "google.compute.instanceGroupManager.MachineTypeRecommender" # MIG machine types
    "google.compute.instance.MachineTypeRecommender"            # VM machine types
)

# Locations to check for each recommender
# Some recommenders are global, others regional. We'll try both.
LOCATIONS_TO_CHECK=("global" "$TARGET_LOCATION")
# Remove duplicate if TARGET_LOCATION is 'global'
if [[ "$TARGET_LOCATION" == "global" ]]; then
    LOCATIONS_TO_CHECK=("global")
fi

echo "Fetching cost recommendations for location: $TARGET_LOCATION and global"
echo "---"

# Get all project IDs
PROJECT_IDS=$(gcloud projects list --format="value(projectId)" --filter="lifecycleState:ACTIVE")

if [ -z "$PROJECT_IDS" ]; then
    echo "No active projects found or insufficient permissions to list projects."
    exit 1
fi

for PROJECT_ID in $PROJECT_IDS; do
    echo ""
    echo "======================================================================"
    echo "Processing Project: $PROJECT_ID"
    echo "======================================================================"

    # --- Recommender API Check ---
    # As per your note, recommender.googleapis.com might appear disabled
    # but individual gcloud recommender commands might still work.
    # If this check causes issues for projects where you expect recommendations,
    # you can comment out this block (from here to "End of API Check").
    #
    # echo "Checking if Recommender API (recommender.googleapis.com) is enabled for project $PROJECT_ID..."
    # API_ENABLED=$(gcloud services list --project="$PROJECT_ID" --enabled --filter="config.name:recommender.googleapis.com" --format="value(config.name)")
    #
    # if [ -z "$API_ENABLED" ]; then
    #     echo "Recommender API (recommender.googleapis.com) is NOT ENABLED for project $PROJECT_ID. Skipping."
    #     # Add a 'continue' here if you strictly want to skip based on this check.
    #     # continue
    #     echo "Warning: Recommender API appears disabled, but proceeding based on user feedback that individual recommenders might still work."
    # else
    #     echo "Recommender API (recommender.googleapis.com) is ENABLED for project $PROJECT_ID."
    # fi
    # --- End of API Check ---
    # The script will proceed to try recommenders regardless of the above check by default,
    # to align with your observation. Errors from 'gcloud recommender recommendations list'
    # will indicate if a specific recommender is truly unavailable.

    HAS_ANY_RECOMMENDATION_FOR_PROJECT=false

    for RECOMMENDER_ID in "${COST_RECOMMENDERS[@]}"; do
        echo "--------------------------------------------------"
        echo "Checking Recommender: $RECOMMENDER_ID for project $PROJECT_ID"
        echo "--------------------------------------------------"
        HAS_RECOMMENDATION_FOR_TYPE=false

        for LOCATION in "${LOCATIONS_TO_CHECK[@]}"; do
            echo "  Trying Location: $LOCATION"

            # Construct the gcloud command
            # We use || true to prevent the script from exiting if a specific recommender/location combo fails
            # Errors will be printed to stderr by gcloud itself.
            COMMAND="gcloud recommender recommendations list \
                        --project=\"$PROJECT_ID\" \
                        --location=\"$LOCATION\" \
                        --recommender=\"$RECOMMENDER_ID\" \
                        --filter=\"stateInfo.state=ACTIVE\" \
                        --format=\"json\""

            # For unattended project recommender, a more specific subtype filter is common
            if [[ "$RECOMMENDER_ID" == "google.resourcemanager.projectUtilization.Recommender" ]]; then
                 COMMAND="gcloud recommender recommendations list \
                            --project=\"$PROJECT_ID\" \
                            --location=\"$LOCATION\" \
                            --recommender=\"$RECOMMENDER_ID\" \
                            --filter=\"stateInfo.state=ACTIVE AND recommenderSubtype=CLEANUP_PROJECT\" \
                            --format=\"json\""
            fi

            # Execute the command and capture output
            # We check for specific errors that mean "not found" or "not applicable" vs. other errors.
            if RECOMMENDATIONS_JSON=$(eval "$COMMAND" 2> >(tee /dev/stderr >&2)); then
                if [ -n "$RECOMMENDATIONS_JSON" ] && [ "$RECOMMENDATIONS_JSON" != "[]" ]; then
                    echo "  SUCCESS: Found recommendations for $RECOMMENDER_ID in $LOCATION for project $PROJECT_ID:"
                    echo "$RECOMMENDATIONS_JSON"
                    HAS_RECOMMENDATION_FOR_TYPE=true
                    HAS_ANY_RECOMMENDATION_FOR_PROJECT=true
                else
                    echo "  INFO: No 'ACTIVE' recommendations found for $RECOMMENDER_ID in $LOCATION for project $PROJECT_ID."
                fi
            else
                # gcloud usually exits with non-zero on errors like API disabled, location not supported, or no recommendations at all.
                # The error message from gcloud (redirected to stderr) would have been printed.
                echo "  INFO: Failed to list recommendations or none found for $RECOMMENDER_ID in $LOCATION (Project: $PROJECT_ID). This recommender/location might not be applicable or API might be disabled for it."
            fi
        done # End of location loop

        if ! $HAS_RECOMMENDATION_FOR_TYPE; then
            echo "No active recommendations found for $RECOMMENDER_ID across checked locations for project $PROJECT_ID."
        fi
    done # End of recommender loop

    if ! $HAS_ANY_RECOMMENDATION_FOR_PROJECT; then
        echo ""
        echo "No cost-related recommendations found for project $PROJECT_ID with the configured recommenders and locations."
    fi
done # End of project loop

echo ""
echo "======================================================================"
echo "Script finished."
echo "======================================================================"
