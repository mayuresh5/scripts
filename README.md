# scripts

#!/bin/bash

File to store the output
OUTPUT_FILE="idle_cluster_recommendations.json"
echo "[]" > "$OUTPUT_FILE"

Get all active project IDs
PROJECT_IDS=$(gcloud projects list --format="value(projectId)")

echo "Scanning GCP projects for 'Delete idle cluster' recommendations under 'COST' category..."

for PROJECT_ID in $PROJECT_IDS; do
echo "Checking project: $PROJECT_ID"

bash
Copy
Edit
# Check if the Recommender API is enabled
API_ENABLED=$(gcloud services list \
    --project="$PROJECT_ID" \
    --filter="config.name:recommender.googleapis.com" \
    --format="value(config.name)")

if [[ -z "$API_ENABLED" ]]; then
    echo " - Recommender API is not enabled. Skipping."
    continue
fi

# Fetch recommendations (Delete Idle Cluster - Category COST)
RECOMMENDATIONS=$(gcloud recommender recommendations list \
    --project="$PROJECT_ID" \
    --location=global \
    --recommender=google.cloud.gkeHub.clusterIdleRecommender \
    --format=json 2>/dev/null)

if [[ $? -ne 0 || "$RECOMMENDATIONS" == "[]" ]]; then
    echo " - No recommendations found or error occurred. Skipping."
    continue
fi

# Filter for recommendations with subtype DELETE_IDLE_CLUSTER and category COST
MATCHING=$(echo "$RECOMMENDATIONS" | jq '[.[] | select(.recommenderSubtype == "DELETE_IDLE_CLUSTER" and .category == "COST")]')

if [[ "$MATCHING" == "[]" ]]; then
    echo " - No matching recommendations in COST category. Skipping."
    continue
fi

echo " - Found matching recommendations. Appending to output."

# Append results to output JSON file
jq -s '.[0] + .[1]' "$OUTPUT_FILE" <(echo "$MATCHING") > tmp.json && mv tmp.json "$OUTPUT_FILE"
done

echo "✅ Done. Final results saved to $OUTPUT_FILE"
