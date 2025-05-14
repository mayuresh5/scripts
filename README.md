#!/bin/bash

# Set the location
LOCATION="asia-south1"

# Define the list of recommenders
RECOMMENDERS=(
  "google.container.Diagnosis.Recommender"
  "google.compute.instance.IdleResourceRecommender"
  "google.compute.disk.IdleResourceRecommender"
  "google.compute.address.IdleResourceRecommender"
  "google.compute.image.IdleResourceRecommender"
  "google.compute.instance.MachineTypeRecommender"
  "google.compute.instanceGroupManager.MachineTypeRecommender"
  "google.cloudsql.instance.IdleRecommender"
  "google.cloudsql.instance.OverprovisionedRecommender"
  "google.resourcemanager.projectUtilization.Recommender"
  "google.billing.commitment.SpendBasedCommitmentRecommender"
  "google.compute.commitment.UsageCommitmentRecommender"
  "google.cloudrun.service.CostRecommender"
)

# Get a list of all your GCP projects
PROJECTS=$(gcloud projects list --format="value(projectId)")

# Check if any projects were found
if [ -z "$PROJECTS" ]; then
  echo "No GCP projects found or you do not have permission to list them."
  exit 1
fi

echo "Starting to fetch recommendations for location: $LOCATION"
echo "========================================================"

# Loop through each project
for PROJECT in $PROJECTS; do
  echo ""
  echo "Processing Project: $PROJECT"
  echo "-------------------------------------"

  # Check if the Recommender API is enabled for the project
  # This is a good practice, though the gcloud command itself will error out if not enabled.
  # You might want to uncomment and adapt this if you need more sophisticated error handling
  # or want to skip projects where the API is not enabled.
  #
  # api_enabled=$(gcloud services list --project="$PROJECT" --filter="recommender.googleapis.com" --format="value(name)")
  # if [ -z "$api_enabled" ]; then
  #   echo "Recommender API is not enabled for project $PROJECT. Skipping."
  #   continue
  # fi

  # Loop through each recommender
  for RECOMMENDER in "${RECOMMENDERS[@]}"; do
    echo "  Fetching recommendations for Recommender: $RECOMMENDER"

    # Run the gcloud command
    # The command will output an error if no recommendations are found or if the recommender is not applicable.
    # We capture both stdout and stderr to provide more context.
    output=$(gcloud recommender recommendations list \
      --project="$PROJECT" \
      --location="$LOCATION" \
      --recommender="$RECOMMENDER" 2>&1)

    # Check if the output indicates no recommendations were found or an error occurred
    if [[ "$output" == *"Listed 0 items."* ]] || [[ "$output" == *"NOT_FOUND"* ]] || [[ "$output" == *"FAILED_PRECONDITION"* ]] || [[ "$output" == *"INVALID_ARGUMENT"* ]]; then
      echo "    No recommendations found or recommender not applicable for $RECOMMENDER in project $PROJECT, location $LOCATION."
      # You can print the specific error for debugging if needed:
      # echo "    Details: $output"
    elif [[ "$output" == *"ERROR"* ]]; then
      echo "    An error occurred while fetching recommendations for $RECOMMENDER in project $PROJECT:"
      echo "    $output"
    else
      echo "    Recommendations:"
      echo "$output"
    fi
    echo "  -----------------------------------"
  done
  echo "Finished processing Project: $PROJECT"
  echo "========================================================"
done

echo ""
echo "All projects and recommenders processed."
