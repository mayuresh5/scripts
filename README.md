# scripts

#!/bin/bash

# Get the current date for any logging or output filenames if needed
# CURRENT_DATE=$(date +"%Y-%m-%d_%H-%M-%S")

echo "Starting script to fetch Active Assist Cost recommendations..."
echo "----------------------------------------------------------"

# Get all project IDs
PROJECT_IDS=$(gcloud projects list --format="value(projectId)" --quiet)

if [ -z "$PROJECT_IDS" ]; then
  echo "No projects found or gcloud CLI is not configured correctly."
  exit 1
fi

# Loop through each project ID
for PROJECT_ID in $PROJECT_IDS; do
  echo ""
  echo "Processing project: $PROJECT_ID"

  # Check if the Recommender API is enabled for the project
  API_ENABLED=$(gcloud services list --project="$PROJECT_ID" --filter="config.name=recommender.googleapis.com" --format="value(config.name)" --quiet)

  if [ "$API_ENABLED" != "recommender.googleapis.com" ]; then
    echo "  Skipping project $PROJECT_ID: Recommender API (recommender.googleapis.com) is not enabled."
    continue
  fi

  echo "  Recommender API is enabled for $PROJECT_ID."

  # Fetch cost recommendations.
  # We use 'google.cost.recommender' which is a common ID for general cost recommendations.
  # Recommendations are typically global, but some specific cost recommenders might be regional.
  # We filter for ACTIVE recommendations.
  echo "  Fetching 'Category: Cost' recommendations for $PROJECT_ID..."
  RECOMMENDATIONS_JSON=$(gcloud recommender recommendations list \
    --project="$PROJECT_ID" \
    --recommender="google.cost.recommender" \
    --location="global" \
    --filter="stateInfo.state=ACTIVE" \
    --format="json" --quiet)

  # Check if the command was successful and if recommendations were found
  if [ $? -ne 0 ]; then
    echo "  Skipping project $PROJECT_ID: Error occurred while fetching recommendations. This might be due to permissions or the recommender not being applicable."
    continue
  fi

  # Check if the JSON array is empty (no recommendations)
  if [ -z "$RECOMMENDATIONS_JSON" ] || [ "$RECOMMENDATIONS_JSON" == "[]" ]; then
    echo "  Skipping project $PROJECT_ID: No active 'Category: Cost' recommendations found (using google.cost.recommender in global location)."
  else
    echo "  Active 'Category: Cost' recommendations for project $PROJECT_ID:"
    echo "$RECOMMENDATIONS_JSON"
    # If you want to save output to individual files per project:
    # echo "$RECOMMENDATIONS_JSON" > "${PROJECT_ID}_cost_recommendations_${CURRENT_DATE}.json"
    # echo "  Recommendations saved to ${PROJECT_ID}_cost_recommendations_${CURRENT_DATE}.json"
  fi

done

echo ""
echo "----------------------------------------------------------"
echo "Script finished."
